

### [H-1] Incorrect logic `Ethereal::mint` function which denies a user from minting an NFT even the user sending more ether for it.

**Description:**
In `Ethereal::mint` function, require logic checks that the msg.value needs to be exactly equal to the corresponding gemid denomination value which reverts the user from minting a gem, even if the user sending more ether for the gem which belongs to that particular id.

```javascript
   function mint(uint256 _id, address _recipient)
        external
        payable
        nonReentrant
        whenNotPaused
        returns (uint256 tokenId_)
    {
 @>       require(msg.value == gems[_id].denomination, "Wrong ether amount"); // @audit-issue
        if (collections[gems[_id].collection].validator) {
            require(collections[gems[_id].collection].validatorAddress == msg.sender, "Not Validator");
        }
        require(gems[_id].active, "No longer mintable");
        if (collections[gems[_id].collection].ethereum) {
            return _mintEth(_id, gems[_id].collection, _recipient);
        } else {
            return _mintWstEth(_id, gems[_id].collection, _recipient);
        } // gem[_id].active = false
    }


```

**Impact:**

1. Any user who tries to mint a gem with sufficient ether will unable to mint due to the incorrect logic.

**Proof of Concept:**

```javascript
      function setUpMint() public {
        ethereal = new Ethereal();

        _createCollection("1st Collection", false, address(0), true, BASE_URL);
        _createGem(0, 100 * 1e18, 10, true);
    }

    function test_MintForUser() public {
        setUpMint();
        vm.deal(user1, 300 * 1e18);

        vm.prank(user1);
        vm.expectRevert();
        uint256 tokenId = ethereal.mint{value: 200 * 1e18}(0, user1); // this line will cause revert.

    }
```

**Recommended Mitigation:**

       Logic for checking, if a user has sufficient ether would be, msg.value for minting a gem is greater than or equal to denomination value of gem.

```diff
   function mint(uint256 _id, address _recipient)
        external
        payable
        nonReentrant
        whenNotPaused
        returns (uint256 tokenId_)
    {
+      require(msg.value >= gems[_id].denomination, "Wrong ether amount"); // @audit-issue
        if (collections[gems[_id].collection].validator) {
            require(collections[gems[_id].collection].validatorAddress == msg.sender, "Not Validator");
        }
        require(gems[_id].active, "No longer mintable");
        if (collections[gems[_id].collection].ethereum) {
            return _mintEth(_id, gems[_id].collection, _recipient);
        } else {
            return _mintWstEth(_id, gems[_id].collection, _recipient);
        }
    }


```

### [H-2] Owner can't able to withdraw Fee from the protocol since the same fee public variable is used to account native ETH and WstETH

**Description:**
When a user redeem their token to get the asset which is backing the NFT, protocol owner collects some fee and send the remaining to the user accordingly, the fee public variable which is responsible for accounting the fee collection from the user is used for both redeeming native eth and WstEth which cause transaction to revert when owner call the `withdrawFee()` from the system.

```javascript
 function _redeemEth(uint256 _tokenId) internal {
        safeTransferFrom(msg.sender, address(this), _tokenId);
        _burn(_tokenId);
        circulatingGems--;
        uint256 redeemFee = (metadata[_tokenId].balance * gems[metadata[_tokenId].gem].redeemFee) / 1e4; //2e18 *
        uint256 amount = metadata[_tokenId].balance - redeemFee;
 @>       fees += redeemFee; // same fee variable is used for below for _redeemWstETH
        metadata[_tokenId].balance = 0;
        (bool success,) = msg.sender.call{value: amount}(" ");
        require(success, " ");
        emit GemRedeemed(_tokenId, msg.sender, amount);
    }

   function _redeemWstEth(uint256 _tokenId) internal {
        safeTransferFrom(msg.sender, address(this), _tokenId);
        uint256 redeemFee = metadata[_tokenId].balance * gems[metadata[_tokenId].gem].redeemFee / 1e4;
        uint256 amount = metadata[_tokenId].balance - redeemFee;
 @>       fees += redeemFee; // using same fee variable
        metadata[_tokenId].balance = 0;
        _burn(_tokenId);
        circulatingGems--;
        IwstETH(wstETH).transfer(msg.sender, amount); // transfer will not return boolean tra
        emit GemRedeemed(_tokenId, msg.sender, amount);
    }
```

**Impact:**

1. Owner can't able to withdraw fee from the system.

**Proof of Concept:**

The following code demonstrates two users minting and redeeming token of one backed by ether and other by WstETH and when owner transaction will revert on calling withdrawFee().

```javascript
    function setUpRedeem() public {
        ethereal = new Ethereal();
        user1 = makeAddr("user1");
        user2 = makeAddr("user2");

        _createCollection("1st Collection", false, address(0), true , BASE_URL);
        _createGem(0, 100 * 1e18, 15, true); // setting redeemFee as 10

         _createCollection("2nd Collection", false, address(0), false , BASE_URL);
        _createGem(1, 100 * 1e18, 15, true);

        vm.deal(user1, 100 * 1e18); // Ensuring user1 has enough ETH
        vm.prank(user1);
        mintedTokenId = ethereal.mint{value: 100 * 1e18}(0, user1); // Mint a token and store its ID

         vm.deal(user2, 100 * 1e18);
         vm.prank(user2);
         minted2TokenId = ethereal.mint{value: 100 * 1e18}(1, user2);
    }

    function test_withdrawFee() public {
        setUpRedeem(); // fee will be collected from the user
        vm.prank(user1);
        ethereal.redeem(mintedTokenId);

        vm.prank(user2);
        ethereal.redeem(minted2TokenId);

        vm.prank(ethereal.payout());
        vm.expectRevert();
        ethereal.withdrawFees(); // transaction will revert.

    }

```

**Recommended Mitigation:**
It is recommended to use two difference fee variables, one for collecting native Ether and other for collecting WStETH to manage proper accounting.

### [M-1] Inconsistency in usage of parallel data structure will cause mint to revert

**Description:**
Collection object properties can be accessed via collections array with "id" value and also with collections array with "gem" object "collection" property as value. When a owner updating a Gem object with incorrect uint collection value which never existed or created will cause revert in mint function.

```javascript
    function mint(uint256 _id, address _recipient)
      external
      payable
      nonReentrant
      whenNotPaused
      returns (uint256 tokenId_)
  {
      require(msg.value == gems[_id].denomination, "Wrong ether amount"); // @audit-issue
@>       if (collections[gems[_id].collection].validator) { // accessing Collection object properties with  property in gem object
          require(collections[gems[_id].collection].validatorAddress == msg.sender, "Not Validator");
      }
  }

```

**Impact:**

1. Any user who tries to mint a gem using mint function will revert due to creation of gem with non-existent uint collection value.

**Proof of Concept:**
When owner creates gem with non-existent collection value will revert,

```javascript
  function setUpMint() public {
       ethereal = new Ethereal();

       _createCollection("1st Collection", false, address(0), true, BASE_URL);
@>       _createGem(2, 100 * 1e18, 10, true); // gem created with non-existent collection value
   }

   function test_MintForUser() public {
       setUpMint();
       vm.deal(user1, 300 * 1e18);

       vm.prank(user1);
       vm.expectRevert();
       uint256 tokenId = ethereal.mint{value: 100 * 1e18}(0, user1); // sending exact msg.value

   }

```

**Recommended Mitigation:**

    In state variables, add the following,

```diff
      /* ======== STATE VARIABLES ======== */

    struct Collection {
+       uint id;
        string name;
        bool validator;
        address validatorAddress;
        bool ethereum;
        string baseURI; // id is also tracking th
    }

    struct Gem {
-       uint collection;
        uint256 denomination;
        uint256 redeemFee; // % reward (3 decimals: 100 = 1%)
        bool active;
    }

```

In `createCollection()` function, add the following

```diff
    function createCollection(
+        uint memory id,
        string memory _name,
        bool _validator,
        address _validatorAddress,
        bool _ethereum,
        string memory _baseURI
    ) external onlyOwner returns (uint256 id_) {
        id_ = collections.length; // upon creating a first collection, id value takes 0

        collections.push(
            Collection({
+                id: _id,
                name: _name,
                validator: _validator,
                validatorAddress: _validatorAddress,
                ethereum: _ethereum,
                baseURI: _baseURI
            })
        );

        emit CreateCollection(id_);
    }

```

In mint() tweak some lines as follows,

```diff
    function mint(uint256 _id, address _recipient)
        external
        payable
        nonReentrant
        whenNotPaused
        returns (uint256 tokenId_)
    {
        require(msg.value == gems[_id].denomination, "Wrong ether amount");
+        if (collections[_id].validator) {
+            require(collections[_id].validatorAddress == msg.sender, "Not Validator");
        }
        require(gems[_id].active, "No longer mintable");
+        if (collections[_id].ethereum) {
+            return _mintEth(_id, collections[_id].id, _recipient);
        } else {
            return _mintWstEth(_id, collections[_id].id, _recipient);
        } // gem[_id].active = false
    }


```

### [M-2] Duplicate minting is allowed for same gem(NFT)

**Description:**
When minting a token, user will pass \_id of the gem and recepient address as parameter to mint the token with particular gemId, after the token got minted, another user can use the same gemId by passing a ether value to mint the same token without any transaction revert.

**Impact:**
User might lose funds when try to redeem the minted token after the first user who redeemed already.

**Proof of Concept:**

```javascript

  function setUpMint() public {
        ethereal = new Ethereal();

        _createCollection("1st Collection", false, address(0), false, BASE_URL);
        _createGem(0, 100 * 1e18, 10, true); // gem created with non-existent collection value
    }

     function test_UserMintSameTokenId() public {
        setUpMint();
        vm.deal(user1, 300* 1e18);
        vm.prank(user1);
        ethereal.mint{value: 100 * 1e18}(0, user1);

        vm.deal(user2, 200 * 1e18);
        vm.prank(user2);
        // vm.expectRevert();
@>        ethereal.mint{value: 100 * 1e18}(0, user2); // minting the token by passing same gemId

    }

```

**Recommended Mitigation:**

```diff
 function mint(uint256 _id, address _recipient)
     external
     payable
     nonReentrant
     whenNotPaused
     returns (uint256 tokenId_)

 {

     require(msg.value == gems[_id].denomination, "Wrong ether amount");
     if (collections[gems[_id].collection].validator) {
         require(collections[gems[_id].collection].validatorAddress == msg.sender, "Not Validator");
     }
     require(gems[_id].active, "No longer mintable");
     if (collections[gems[_id].collection].ethereum) {
         return _mintEth(_id, gems[_id].collection, _recipient);
     } else {
         return _mintWstEth(_id, gems[_id].collection, _recipient);
     }
+        gem[_id].active = false // setting the _id to false after minting
 }

```
