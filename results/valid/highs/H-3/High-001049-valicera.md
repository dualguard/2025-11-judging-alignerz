# [001049] splitTVS will hit runtime revert when user tries to split
  
  ### Summary

`_computeSplitArrays` is writing into dynamic array slots without first allocating the dynamic arrays in the returned `Allocation memory alloc`. In Solidity you must allocate memory for dynamic arrays before you write to their elements. Failing to do so writes to invalid memory and causes the runtime panic revert - array out-of-bounds.

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1113-L1141

### Root Cause

in `AlignerzVesting::splitTVS`, internal function `_computeSplitArrays` is called for an allocation that has at least one flow (nbOfFlows > 0). In practice that means almost always when you split a real TVS. So this is a reproducible, runtime panic 0x32 revert that will revert the transaction every time the function runs on a non-empty allocation.
`_computeSplitArrays` returns an Allocation memory alloc whose dynamic array fields (alloc.amounts, alloc.vestingPeriods, ...) are uninitialized in memory.
The function then writes directly to alloc.amounts[j], alloc.vestingPeriods[j], etc., without first allocating those memory arrays with new ... (length).
```solidity
    function _computeSplitArrays(Allocation storage allocation, uint256 percentage, uint256 nbOfFlows)
        internal
        view
        returns (Allocation memory alloc)
    {
        // ...
        for (uint256 j; j < nbOfFlows;) {
            //@audit writing to memory array must be allocated first with new []
            alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;
            alloc.vestingPeriods[j] = baseVestings[j];
            alloc.vestingStartTimes[j] = baseVestingStartTimes[j];
            alloc.claimedSeconds[j] = baseClaimed[j];
            alloc.claimedFlows[j] = baseClaimedFlows[j];
            unchecked {
                ++j;
            }
        // ...
    }
```
Writing into an uninitialised memory dynamic array is an out-of-bounds memory write under the EVM/solc runtime. It raises the Panic 0x32 (array out-of-bounds) and reverts.

### Internal Pre-conditions

1. User must have TVS with nbOfFlows > 0

### External Pre-conditions

N/A

### Attack Path

N/A

### Impact

Breaks core functionality  likelihood is high - normal flow of splitting TVS rely on `_computeSplitArrays` which is denied, impact is medium - user cannot split his TVS but he can essentially claim.

### PoC

`splitTVS` has another problematic function which will be in different report but in order to hit _computeSplitArrays we have to use my helper function:
In FeesManager add the following function:
```solidity
    function calculateFeeAndNewAmountForOneTVSSafe(uint256 feeRate, uint256[] memory amounts, uint256 length)
        public
        pure
        returns (uint256 feeAmount, uint256[] memory newAmounts)
    {
        require(length <= amounts.length, "Invalid length");
        newAmounts = new uint256[](length);
        for (uint256 i = 0; i < length;) {
            uint256 perFee = calculateFeeAmount(feeRate, amounts[i]);
            // Subtract per-flow fee, avoid underflow by clamping to zero
            if (amounts[i] >= perFee) {
                newAmounts[i] = amounts[i] - perFee;
            } else {
                newAmounts[i] = 0;
            }
            feeAmount += perFee;
            unchecked {
                ++i;
            }
        }
        return (feeAmount, newAmounts);
    }
```
And in AlignerzVesting use it in `splitTVS` function:
```solidity
    function splitTVS(uint256 projectId, uint256[] calldata percentages, uint256 splitNftId)
        external
        returns (uint256, uint256[] memory)
    {
           // ...
        uint256[] memory amounts = allocation.amounts;
        uint256 nbOfFlows = allocation.amounts.length;
        (uint256 feeAmount, uint256[] memory newAmounts) =
>>          calculateFeeAndNewAmountForOneTVSSafe(splitFeeRate, amounts, nbOfFlows);
        allocation.amounts = newAmounts;
        token.safeTransfer(treasury, feeAmount);
            // ...
    }
```
Then add in `AlignerzVestingProtocolTest` the following test and run `forge test --mt test_SplitTVS_Panic_Reverts -vvvv`

```solidity
    function test_SplitTVS_Panic_Reverts() public {
        address kol = bidders[0];

        // projectCreator launches a reward project and sets a single allocation
        vm.startPrank(projectCreator);
        vesting.launchRewardProject(address(token), address(usdt), block.timestamp, 1 days);
        address[] memory kols = new address[](1);
        kols[0] = kol;
        uint256[] memory amts = new uint256[](1);
        amts[0] = 1_000 ether;
        vesting.setTVSAllocation(PROJECT_ID, amts[0], 90 days, kols, amts);
        vm.stopPrank();

        // kol claims his TVS
        vm.prank(kol);
        vesting.claimRewardTVS(PROJECT_ID);
        uint256 nftId = nft.getTotalMinted();

        // Attempt to split the TVS into two parts - we expect the call to revert due to
        // the unsafe internal `_computeSplitArrays` writing into unallocated memory.
        uint256[] memory percentages = new uint256[](2);
        percentages[0] = 5000;
        percentages[1] = 5000;

        vm.prank(kol);
        vm.expectRevert();
        vesting.splitTVS(PROJECT_ID, percentages, nftId);
    }
```

### Mitigation

Allocate memory for the returned dynamic arrays before writing to memory allocation.
alloc.amounts = new uint256 [] (nbOfFlows); etc...
  