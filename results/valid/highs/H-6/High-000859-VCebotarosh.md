# [000859] DoS On Dividend Configuration
  
  ### Summary

Calling `setAmounts()` or `setUpTheDividends()` eventually leads to the execution of getTotalUnclaimedAmounts(), which iterates over all minted token IDs up to [_totalMinted](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L128). Since `_totalMinted` never decrements, the loop processes both existing and non-existing tokens.
An attacker can exploit this by repeatedly calling splitTVS() (minting) followed by mergeTVS() (burning), artificially inflating _totalMinted without increasing the number of actual NFTs. Eventually, calls to `setAmounts()` or `setUpTheDividends()` will run out of gas, preventing the owner from configuring dividends.

### Root Cause

`getTotalUnclaimedAmounts()` loops from 0 to `_totalMinted`. However, [_totalMinted](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/nft/ERC721A.sol#L123-L129) represents the total number ever minted, including burned tokens. This design enables attackers to inflate `_totalMinted` through repeated split/merge operations, forcing the dividend configuration functions into an unbounded loop that exceeds block gas limits. .

### Internal Pre-conditions

None

### External Pre-conditions

None

### Attack Path

1. User calls multiple times splitTVS() and mergeTVS() to increase the _totalMinted amount
2. Owner tries to call setAmount() in the A26ZDividendDistributor.sol and fails due to OOG

### Impact

The owner is unable to configure or update dividend distributions.
Dividend initialization becomes impossible, breaking core functionality of A26ZDividendDistributor.

### PoC

Add the following import to the top of the AlignerzVestingProtocolTest.t.sol
`import {A26ZDividendDistributor} from "../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";`

Create a public variable:
`A26ZDividendDistributor distributor;`

Modify the setUp() function:
```solidity
...
        vesting = AlignerzVesting(proxy);
        vm.prank(projectCreator);
        distributor = new A26ZDividendDistributor(address(vesting), address(nft),address(usdt), block.timestamp, 7 days, address(token));
...
```

Add the following function to the AlignerzVestingProtocolTest.t.sol and call it via:
`forge test --mt testForOutOfGas -vvv`

```solidity
function testForOutOfGas() public {
        uint256 _totalUnclaimedAmounts = 0;
        for (uint256 i = 0; i < 165_000;)
        {
            (,bool owned) = distributor.safeOwnerOf(0);
            if (owned) _totalUnclaimedAmounts += 1;
            unchecked{
                ++i;
            }
        }
    }
```

The results:
```bash
A26ZDividendDistributor::safeOwnerOf(0) [staticcall]
    │   ├─ [634] AlignerzNFT::extOwnerOf(0) [staticcall]
    │   │   └─ ← [OutOfGas] EvmError: OutOfGas
    │   └─ ← [OutOfGas] EvmError: OutOfGas
    └─ ← [OutOfGas] EvmError: OutOfGas

Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 6.61s (3.25s CPU time)

Ran 1 test suite in 47.73s (6.61s CPU time): 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in test/AlignerzVestingProtocolTest.t.sol:AlignerzVestingProtocolTest
[FAIL: EvmError: OutOfGas] testForOutOfGas()
```

This test demonstrates that for the _totalMinted = 165000, the call will fail due to the OOG. As stated in the root cause, a malicious user can increase the _totalMinted by calling splitTVS() and then mergeTVS().

### Mitigation

Consider iterating only through the existent tokens, and keep them separately.
  