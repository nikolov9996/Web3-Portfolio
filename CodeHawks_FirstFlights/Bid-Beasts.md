# Bid Beasts - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)

- ## Medium Risk Findings
    - ### [M-01. Precision loss ](#M-01)
- ## Low Risk Findings
    - ### [L-01. Auctions end 15 min after the last bid](#L-01)
    - ### [L-02. Missing endAuction function](#L-02)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #49

### Dates: Sep 25th, 2025 - Oct 2nd, 2025

[See more contest details here](https://codehawks.cyfrin.io/c/2025-09-bid-beasts)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 0
- Medium: 1
- Low: 2



    
# Medium Risk Findings

## <a id='M-01'></a>M-01. Precision loss             



# Root + Impact

## Description

Calaculating the minimum Bid increese must me exact 5% from the previous.

Currently there is `loss of precision` issue due to not correct calculations of  `requiredAmount` in `placeBid` function

```Solidity
 function placeBid(uint256 tokenId) external payable isListed(tokenId) {
      
More code...
   
        if (previousBidAmount == 0) {
            requiredAmount = listing.minPrice;
            require(msg.value > requiredAmount, "First bid must be > min price");
            listing.auctionEnd = block.timestamp + S_AUCTION_EXTENSION_DURATION;
            emit AuctionExtended(tokenId, listing.auctionEnd);
        } else {
@>          requiredAmount = (previousBidAmount / 100) * (100 + S_MIN_BID_INCREMENT_PERCENTAGE);
            require(msg.value >= requiredAmount, "Bid not high enough");

            uint256 timeLeft = 0;
            if (listing.auctionEnd > block.timestamp) {
                timeLeft = listing.auctionEnd - block.timestamp;
            }
            if (timeLeft < S_AUCTION_EXTENSION_DURATION) {
                listing.auctionEnd = listing.auctionEnd + S_AUCTION_EXTENSION_DURATION;
                emit AuctionExtended(tokenId, listing.auctionEnd);
            }
        }
   
More code...
      
    }
```

## Risk

**Likelihood**:

It will happen every time there is when there is decimal that solidity cannot handle\
In the lifetime of a Auction it can stack multiple times

\
**Impact**:

consistent loss of percision in some cases making the bidding possible with less that exactly 5% increment.

## Proof of Concept

The test shows how the wrong calculation affect the price of the bid

The problematic way of calculating compared to the correct way.\
Added comments so it can be easier to read and understand.

```Solidity
function test_takeHighestBidPrecisionPoC() public {
        uint256 minBidIncrementPercentage = market.S_MIN_BID_INCREMENT_PERCENTAGE();

        _mintNFT();
        _listNFT();

        // required to prove the calculations are not rounding properly
        uint256 newBid = MIN_PRICE + 199999999999999999; // 1.199999999999999 ether

        vm.prank(BIDDER_1);
        market.placeBid{value: newBid}(TOKEN_ID);

        // 105 Exact MIN_BID_INCREMENT_PERCENTAGE
        uint256 secondBidAmount = (newBid * (100 + minBidIncrementPercentage)) / 100;
        vm.prank(BIDDER_2);
        market.placeBid{value: secondBidAmount}(TOKEN_ID);

        /*//////////////////////////////////////////////////////////////
                              CALCULATIONS
        //////////////////////////////////////////////////////////////*/
        BidBeastsNFTMarket.Bid memory highestBid = market.getHighestBid(TOKEN_ID);
        // reusing the logic from the contract to show the difference
        // requiredAmount = (previousBidAmount / 100) * (100 + S_MIN_BID_INCREMENT_PERCENTAGE);
        uint256 wrongCalculation = (highestBid.amount / 100) * (100 + minBidIncrementPercentage);
        uint256 correctCalculation = (highestBid.amount * (100 + minBidIncrementPercentage)) / 100;

        console.log("Wrong Calculation:", wrongCalculation); // 1322999999999999895
        console.log("Correct Calculation:", correctCalculation); // 1322999999999999997

        // 1322999999999999997 - 1322999999999999895 = 102
        console.log("Difference:", correctCalculation - wrongCalculation); // 102
        // I can simply subtract 102 wei to make it pass the require in `placeBid()`

        /*//////////////////////////////////////////////////////////////
                              CALCULATIONS
        //////////////////////////////////////////////////////////////*/

        // Subtracting 102 wei to make it revert, but its does not
        uint256 thirdBidAmount = (secondBidAmount * (100 + minBidIncrementPercentage) / 100) - 102 wei;
        vm.prank(BIDDER_3);
        market.placeBid{value: thirdBidAmount}(TOKEN_ID);
    }
```

## Recommended Mitigation

Multiplication must happent before division.\
After replacing the line the PoC test should fail! And it needs to remove the  - 102 wei part to run correctly just like the new calculations.

```diff
 function placeBid(uint256 tokenId) external payable isListed(tokenId) {
      
More code...
   
        if (previousBidAmount == 0) {
            requiredAmount = listing.minPrice;
            require(msg.value > requiredAmount, "First bid must be > min price");
            listing.auctionEnd = block.timestamp + S_AUCTION_EXTENSION_DURATION;
            emit AuctionExtended(tokenId, listing.auctionEnd);
        } else {
-           requiredAmount = (previousBidAmount / 100) * (100 + S_MIN_BID_INCREMENT_PERCENTAGE);
+           requiredAmount = (previousBidAmount * (100 + S_MIN_BID_INCREMENT_PERCENTAGE)) / 100;
            require(msg.value >= requiredAmount, "Bid not high enough");

            uint256 timeLeft = 0;
            if (listing.auctionEnd > block.timestamp) {
                timeLeft = listing.auctionEnd - block.timestamp;
            }
            if (timeLeft < S_AUCTION_EXTENSION_DURATION) {
                listing.auctionEnd = listing.auctionEnd + S_AUCTION_EXTENSION_DURATION;
                emit AuctionExtended(tokenId, listing.auctionEnd);
            }
        }
   
More code...
      
    }
```


# Low Risk Findings

## <a id='L-01'></a>L-01. Auctions end 15 min after the last bid            



# Root + Impact

## Description

* The Auctions are supposed to be active for 3 days after the first bid - Explained in the Docs. With the current logic this is not possible because the Auctions end when no bids are placed 15 min after the first bid.

* This creates a wrong behaviour:\
  \- Auctions can end just 15 min after the first bid if nobody bid in this time.\
  \- Missleading information for the behaviour of the contract.

* The `S_AUCTION_EXTENSION_DURATION` constant is set to 15 min and it's used to extend the duration, but this is not the desired behaviour

```Solidity
function placeBid(uint256 tokenId) external payable isListed(tokenId) {
      
More code...
  
        // --- Regular Bidding Logic ---
        uint256 requiredAmount;

        if (previousBidAmount == 0) {
            requiredAmount = listing.minPrice;
            require(msg.value > requiredAmount, "First bid must be > min price");
 @>         listing.auctionEnd = block.timestamp + S_AUCTION_EXTENSION_DURATION;
            emit AuctionExtended(tokenId, listing.auctionEnd);
        } else {
            requiredAmount = (previousBidAmount / 100) * (100 + S_MIN_BID_INCREMENT_PERCENTAGE);
            require(msg.value >= requiredAmount, "Bid not high enough");

            uint256 timeLeft = 0;
            if (listing.auctionEnd > block.timestamp) {
                timeLeft = listing.auctionEnd - block.timestamp;
            }
 @>         if (timeLeft < S_AUCTION_EXTENSION_DURATION) {
 @>             listing.auctionEnd = listing.auctionEnd + S_AUCTION_EXTENSION_DURATION;
                emit AuctionExtended(tokenId, listing.auctionEnd);
            }
        }

 More code...
    }
```

## Risk

**Likelihood**:

* This will happen in every single Listing.

**Impact**:

* Unwanted/Wrong behaviour. 

## Proof of Concept

The test `test_bidAfterOneDay` will Fail due to the logic that makes the Auction end 15 after the first bid.

```Solidity
function test_bidAfterOneDay() public {
        _mintNFT();
        _listNFT();

        uint256 newBid = MIN_PRICE + 10;

        vm.prank(BIDDER_1);
        market.placeBid{value: newBid}(TOKEN_ID);
        //we wait 1 day before BIDDER_2 places a bid
        // by the Documentation this MUST be possible
        // The auctions must "live" for 3 days after the first bid
        vm.warp(block.timestamp + 1 days);

        uint256 secondBidAmount = newBid * 120 / 100; // 20% increase
        vm.prank(BIDDER_2);
        market.placeBid{value: secondBidAmount}(TOKEN_ID);
    }
```

## Recommended Mitigation

We need to replace the `S_AUCTION_EXTENSION_DURATION` with `S_AUCTION_DEADLINE` and change the time from `15 minutes` to `3 days`
in the `placeBid` function to replace the constant in the if statement which handles 1st bid and remove the check for updating the `auctionEnd` .
After this changes the auction will end just after the 3rd day like it's expected by the Docs.

```diff
// on line 34
- uint256 constant public S_AUCTION_EXTENSION_DURATION = 15 minutes;
+ uint256 public constant S_AUCTION_DEADLINE = 3 days;

function placeBid(uint256 tokenId) external payable isListed(tokenId) {
      
More code...
  
        // --- Regular Bidding Logic ---
        uint256 requiredAmount;

        if (previousBidAmount == 0) {
            requiredAmount = listing.minPrice;
            require(msg.value > requiredAmount, "First bid must be > min price");
-           listing.auctionEnd = block.timestamp + S_AUCTION_EXTENSION_DURATION;
+           listing.auctionEnd = block.timestamp + S_AUCTION_DEADLINE;
            emit AuctionExtended(tokenId, listing.auctionEnd);
        } else {
            requiredAmount = (previousBidAmount / 100) * (100 + S_MIN_BID_INCREMENT_PERCENTAGE);
            require(msg.value >= requiredAmount, "Bid not high enough");

-           uint256 timeLeft = 0; //<@ This lines are not needed as far as we will not update the auctionEnd anymore
-           if (listing.auctionEnd > block.timestamp) {
-               timeLeft = listing.auctionEnd - block.timestamp;
-           } 
-           if (timeLeft < S_AUCTION_EXTENSION_DURATION) {
-              listing.auctionEnd = listing.auctionEnd + S_AUCTION_EXTENSION_DURATION;
-              emit AuctionExtended(tokenId, listing.auctionEnd);
-            }
        }

 More code...
    }
```

## <a id='L-02'></a>L-02. Missing endAuction function            



## Description

Leading by the docs we must have `endAuction` function and 3days lifespan of the Auctions.

Docs: <https://github.com/CodeHawks-Contests/2025-09-bid-beasts/blob/main/README.md>

## Risk

**Likelihood**:

* The auction logic is totaly different from documentation

* Incomplete Implementation / Documentation missmatch - the missing endAuction function

**Impact**:

* Unexpected  behaviour from described in the docs

  <br />

## Proof of Concept

Missing `endAuction` function that is described in the docs

Missing/Wrong logic for tracking 3 days required for proper functioning of the contract

### Incorrect 

The lifecycle of the auction is fundamentally different from what is documented\
It starts with 15 minutes window and then refreshes this window if anyone place a bid.

### Correct

The 3 days start when the auction is listed 
after this 3 days the `endAuction` function can be called to transfer the NFT
to the seller if no bids were placed or to the highest bidder.

Wrong logic for tracking Auction lifecycle:

```Solidity
function placeBid(uint256 tokenId) external payable isListed(tokenId) {
      
More code...
  
        // --- Regular Bidding Logic ---
        uint256 requiredAmount;

        if (previousBidAmount == 0) {
            requiredAmount = listing.minPrice;
            require(msg.value > requiredAmount, "First bid must be > min price");
 @>         listing.auctionEnd = block.timestamp + S_AUCTION_EXTENSION_DURATION;
            emit AuctionExtended(tokenId, listing.auctionEnd);
        } else {
            requiredAmount = (previousBidAmount / 100) * (100 + S_MIN_BID_INCREMENT_PERCENTAGE);
            require(msg.value >= requiredAmount, "Bid not high enough");

 @>         uint256 timeLeft = 0;
 @>         if (listing.auctionEnd > block.timestamp) {
 @>             timeLeft = listing.auctionEnd - block.timestamp;
 @>          }
 @>         if (timeLeft < S_AUCTION_EXTENSION_DURATION) {
 @>             listing.auctionEnd = listing.auctionEnd + S_AUCTION_EXTENSION_DURATION;
 @>             emit AuctionExtended(tokenId, listing.auctionEnd);
 @>          }
 @>       }

 More code...
    }
```

There is Incorect Auction lifecycle that leads to ending of listings after 15 minutes of innactivity.

To fix the issues we need to:

* add the missing function

* adjust the lifecycle of the Auctions

<br />

## Recommended Mitigation

To fix all the issues related to the Incomplete/Wrong Implementation \
We need to:

### Refactor `placeBid` function

1. on line 34 we need to change the constant <https://github.com/CodeHawks-Contests/2025-09-bid-beasts/blob/449341c55a57d3f078d1250051a7b34625d3aa04/src/BidBeastsNFTMarketPlace.sol#L34>

<br />

1. we refactor the `placeBid`  function code to handle correctly the 3 days as expected

```diff
- uint256 constant public S_AUCTION_EXTENSION_DURATION = 15 minutes;
+ uint256 public constant S_AUCTION_DEADLINE = 3 days;
 

 function placeBid(uint256 tokenId) external payable isListed(tokenId) {
      
More code...
  
        // --- Regular Bidding Logic ---
        uint256 requiredAmount;

        if (previousBidAmount == 0) {
            requiredAmount = listing.minPrice;
            require(msg.value > requiredAmount, "First bid must be > min price");
-           listing.auctionEnd = block.timestamp + S_AUCTION_EXTENSION_DURATION;
+           listing.auctionEnd = block.timestamp + S_AUCTION_DEADLINE;
            emit AuctionExtended(tokenId, listing.auctionEnd);
        } else {
            requiredAmount = (previousBidAmount / 100) * (100 + S_MIN_BID_INCREMENT_PERCENTAGE);
            require(msg.value >= requiredAmount, "Bid not high enough");

-          uint256 timeLeft = 0;
-          if (listing.auctionEnd > block.timestamp) {
-              timeLeft = listing.auctionEnd - block.timestamp;
-           }
-          if (timeLeft < S_AUCTION_EXTENSION_DURATION) {
-              listing.auctionEnd = listing.auctionEnd + S_AUCTION_EXTENSION_DURATION;
-              emit AuctionExtended(tokenId, listing.auctionEnd);
-           }
-        }

 More code...
    }
```

### Implement `endAuction`

1. Add the function + Event

```diff
  // --- Events ---
    event NftListed(uint256 tokenId, address seller, uint256 minPrice, uint256 buyNowPrice);
    event NftUnlisted(uint256 tokenId);
    event BidPlaced(uint256 tokenId, address bidder, uint256 amount);
    event AuctionExtended(uint256 tokenId, uint256 newDeadline);
    event AuctionSettled(uint256 tokenId, address winner, address seller, uint256 price);
    event FeeWithdrawn(uint256 amount);
+   event AuctionEnded(uint256 tokenId);
```

```Solidity
function endAuction(uint256 tokenId) external isListed(tokenId) {
        Listing storage listing = listings[tokenId];
        require(block.timestamp >= listing.auctionEnd, "Auction has not ended");

        if (listing.auctionEnd != 0) {
            _executeSale(tokenId);
        } else {
            listing.listed = false;
            delete bids[tokenId];
            BBERC721.transferFrom(address(this), listing.seller, tokenId);
        }

        emit AuctionEnded(tokenId);
    }
```

### Test the `endAuction`

```Solidity
 function test_endAuctionWithBidder() public {
        _mintNFT();
        _listNFT();

        uint256 newBid = MIN_PRICE + 10;

        vm.prank(BIDDER_1);
        market.placeBid{value: newBid}(TOKEN_ID);

        vm.warp(block.timestamp + 3 days);

        market.endAuction(TOKEN_ID);
        assertEq(nft.ownerOf(TOKEN_ID), BIDDER_1, "NFT should be transferred to highest bidder");
    }

    function test_endAuctionWithoutBidder() public {
        _mintNFT();
        _listNFT();

        vm.warp(block.timestamp + 3 days);

        market.endAuction(TOKEN_ID);
        assertEq(nft.ownerOf(TOKEN_ID), SELLER, "NFT should be transferred back to SELLER");
    }
```



