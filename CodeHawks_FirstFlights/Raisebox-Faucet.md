# Raisebox Faucet - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Incorrect reset of dailyDrips](#H-01)
- ## Medium Risk Findings
    - ### [M-01. Owner receives all token when burning](#M-01)



# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #50

### Dates: Oct 9th, 2025 - Oct 16th, 2025

[See more contest details here](https://codehawks.cyfrin.io/c/2025-10-raisebox-faucet)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 1
- Medium: 1
- Low: 0


# High Risk Findings

## <a id='H-01'></a>H-01. Incorrect reset of dailyDrips            



# High: Daily drips reset unproperly

## Description

* Daily drips reset when a not first time user calls `claimFaucestTokens`

```Solidity

      if (!hasClaimedEth[faucetClaimer] && !sepEthDripsPaused) {
            uint256 currentDay = block.timestamp / 24 hours;

            if (currentDay > lastDripDay) {
                lastDripDay = currentDay;
                dailyDrips = 0;
                // dailyClaimCount = 0;
            }

            if (dailyDrips + sepEthAmountToDrip <= dailySepEthCap && address(this).balance >= sepEthAmountToDrip) {
                hasClaimedEth[faucetClaimer] = true;
                dailyDrips += sepEthAmountToDrip;

                (bool success,) = faucetClaimer.call{value: sepEthAmountToDrip}("");

                if (success) {
                    emit SepEthDripped(faucetClaimer, sepEthAmountToDrip);
                } else {
                    revert RaiseBoxFaucet_EthTransferFailed();
                }
            } else {
                emit SepEthDripSkipped(
                    faucetClaimer,
                    address(this).balance < sepEthAmountToDrip ? "Faucet out of ETH" : "Daily ETH cap reached"
                );
            }
        } else {
            dailyDrips = 0; // @> This part resets the drips when it's not supposed to.
        }
```

## Risk

**Likelihood**: High

* The dailyDrips can be reset many times per day

* Reset happens every time when a user claims tokens for second+ time

**Impact**: High

* Users can claim more sepolia ETH per day than the limit set in `dailySepEthCap`

## Proof of Concept

1. `alice` claims tokens for first time

   * receives tokens and sepolia ETH
2. we wait 3 days

   * bob claims for first time as long with 3 more users - first timers (to make it realistic)

   * dailyDrips are 20000000000000000 = 0.02 ETH for the 4 users that called for 0.005
3. alice call `claimFaucetTokens` for second time triggering the Unwanted clear of dailyDrips

   * daily drips are now 0

```Solidity
 function testResetDailyDrips() public {
        vm.prank(alice); // 1st claim for alice
        raiseBoxFaucet.claimFaucetTokens();
        // alice have claimed 1000 tokens and 0.005 sepolia ETH
        assertEq(raiseBoxFaucet.getBalance(alice), 1000e18);
        assertEq(raiseBoxFaucet.getHasClaimedEth(alice), true);

        vm.warp(block.timestamp + 3 days);

        vm.prank(bob);// bob calls and 3 other users also do (his friends maybe)
        raiseBoxFaucet.claimFaucetTokens();
        vm.prank(user1);
        raiseBoxFaucet.claimFaucetTokens();
        vm.prank(user2);
        raiseBoxFaucet.claimFaucetTokens();
        vm.prank(user3);
        raiseBoxFaucet.claimFaucetTokens();

        // dailyDrips are 20000000000000000 = 0.02 ETH
        assertEq(raiseBoxFaucet.dailyDrips(), 0.02 ether);

        vm.prank(alice);
        raiseBoxFaucet.claimFaucetTokens();

        // dailyDrips are now 0
        assertEq(raiseBoxFaucet.dailyDrips(), 0);
    }
```

## Recommended Mitigation

Remove the else statement after the check for already claimed and if the claiming is paused.
It just breaks the logic of distributing sepolia ETH properly.

```diff
if (!hasClaimedEth[faucetClaimer] && !sepEthDripsPaused) {
            uint256 currentDay = block.timestamp / 24 hours;

            if (currentDay > lastDripDay) {
                lastDripDay = currentDay;
                dailyDrips = 0;
                // dailyClaimCount = 0;
            }

            if (dailyDrips + sepEthAmountToDrip <= dailySepEthCap && address(this).balance >= sepEthAmountToDrip) {
                hasClaimedEth[faucetClaimer] = true;
                dailyDrips += sepEthAmountToDrip;

                (bool success,) = faucetClaimer.call{value: sepEthAmountToDrip}("");

                if (success) {
                    emit SepEthDripped(faucetClaimer, sepEthAmountToDrip);
                } else {
                    revert RaiseBoxFaucet_EthTransferFailed();
                }
            } else {
                emit SepEthDripSkipped(
                    faucetClaimer,
                    address(this).balance < sepEthAmountToDrip ? "Faucet out of ETH" : "Daily ETH cap reached"
                );
            }
-       } else {
-           dailyDrips = 0;
-       }
```

    
# Medium Risk Findings

## <a id='M-01'></a>M-01. Owner receives all token when burning            



# High: All contract tokens transfered to Owner when burning any amount

## Description

When `burnFaucetTokens` is called with any amount the `_transfer` function sends all tokens from the contract to the owner

```Solidity
 function burnFaucetTokens(uint256 amountToBurn) public onlyOwner {
        require(amountToBurn <= balanceOf(address(this)), "Faucet Token Balance: Insufficient");

        // transfer faucet balance to owner first before burning
        // ensures owner has a balance before _burn (owner only function) can be called successfully
        _transfer(address(this), msg.sender, balanceOf(address(this))); <@ All tokens are transfered to Owner 

        _burn(msg.sender, amountToBurn); <@ Burning only the amount supposed to be burned 
    }
```

## Risk

**Likelihood**: High

* When Owner calls \`burnFaucetTokens\`

**Impact**: High

* All the tokens from the faucet contract are transfered to the Owner.

## Proof of Concept

Scenario:

Owner burns whatevet amount, for the purpose of the test - 1 token.\
Result: The token is burned, but all the tokens from the contract are now in the Owner.

```Solidity
 function testBurnFaucetTokens() public {
        uint256 amountToBurn = 1e18; // 1 token
        // verify that the contract have the initial supply
        assertEq(raiseBoxFaucet.getFaucetTotalSupply(), INITIAL_SUPPLY_MINTED);

        // burn single token
        vm.prank(owner);
        raiseBoxFaucet.burnFaucetTokens(amountToBurn);

        // The tokens in the contract are now 0
        assertEq(raiseBoxFaucet.getFaucetTotalSupply(), 0);
        // The owner have all the tokens - amountToBurn.
        assertEq(raiseBoxFaucet.getBalance(owner),INITIAL_SUPPLY_MINTED - amountToBurn);
    }
```

## Recommended Mitigation

Replace the `balanceOf(address(this)` in the `_transfer` call with `amountToBurn`

```diff
 function burnFaucetTokens(uint256 amountToBurn) public onlyOwner {
        require(amountToBurn <= balanceOf(address(this)), "Faucet Token Balance: Insufficient");

        // transfer faucet balance to owner first before burning
        // ensures owner has a balance before _burn (owner only function) can be called successfully
-       _transfer(address(this), msg.sender, balanceOf(address(this)));
+       _transfer(address(this), msg.sender, amountToBurn);
        _burn(msg.sender, amountToBurn);
    }
```





