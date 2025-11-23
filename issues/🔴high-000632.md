# [000632] Any caller will trigger a Panic(0x32) due to writes to an uninitialized dynamic memory array
  
  ### Summary

The missing allocation of `newAmounts` in `calculateFeeAndNewAmountForOneTVS` will cause a `Panic(0x32) `(array out-of-bounds) for vesting `NFT` holders as any user invoking `splitTVS` or `mergeTVS `will reach the fee helper that writes into a zero-length memory array.



### Root Cause

In `FeesManager.calculateFeeAndNewAmountForOneTVS`:

```solidity
function calculateFeeAndNewAmountForOneTVS(
    uint256 feeRate,
    uint256[] memory amounts,
    uint256 length
) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount; // <-- newAmounts never allocated
    }
}
```

`newAmounts` is declared but never allocated `(e.g. newAmounts = new uint256[](length);).` Its implicit length is `0`; any index access reverts with `Panic(0x32)`.

### Internal Pre-conditions

- A vesting NFT has at least one flow `(amounts.length >= 1)`.
- User calls `splitTVS` or `mergeTVS `which unconditionally invokes `calculateFeeAndNewAmountForOneTVS`.
- `feeRate` may be zero or non-zero (revert occurs regardless).

### External Pre-conditions

None

### Attack Path

- User  calls `splitTVS(projectId, percentages, nftId)` with any valid percentages.
- Contract enters `calculateFeeAndNewAmountForOneTVS` with `length > 0`.
- First loop iteration attempts `newAmounts[0]` = ....
- EVM detects out-of-bounds write on zero-length array → `Panic(0x32)`.
- Operation reverts; `split/merge` functionality unusable (DoS).

### Impact

Users cannot split or restructure vesting flows.

### PoC

Add this function to `AlignerzVestingProtocolTest.sol`

```solidity
// ---
    // --- PoC for SplitTVS triggering Panic(0x32) via faulty calculateFeeAndNewAmountForOneTVS for UninitializedArrayPanic ---
    // ---
    function test_POC_UninitializedArrayPanic() public {
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);
        vesting.launchRewardProject(address(token), address(usdt), block.timestamp, 7 days);
        uint256 rewardProjectId = 0;

        address kol = makeAddr("kolSplit");
        address[] memory kols = new address[](1);
        kols[0] = kol;
        uint256[] memory amounts = new uint256[](1);
        amounts[0] = 1_000 ether;
        vesting.setTVSAllocation(rewardProjectId, 1_000 ether, 30 days, kols, amounts);
        vesting.setSplitFeeRate(100); // non-zero fee
        vm.stopPrank();

        vm.prank(kol);
        vesting.claimRewardTVS(rewardProjectId);
        uint256 nftId = nft.getTotalMinted();

        uint256[] memory percentages = new uint256[](2);
        percentages[0] = 5000;
        percentages[1] = 5000;
        vm.prank(kol);
        vm.expectRevert(abi.encodeWithSignature("Panic(uint256)", 0x32));
        vesting.splitTVS(rewardProjectId, percentages, nftId);
    }
```

### Mitigation

- Implement proper memory allocation before writes:

```solidity
function calculateFeeAndNewAmountForOneTVS(
    uint256 feeRate,
    uint256[] memory amounts,
    uint256 length
) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    require(length == amounts.length, "Length mismatch");
    newAmounts = new uint256[](length);
    for (uint256 i; i < length; ) {
        uint256 fee = calculateFeeAmount(feeRate, amounts[i]);
        feeAmount += fee;
        newAmounts[i] = amounts[i] - fee;
        unchecked { ++i; } 
    }
}
```
  