# [001058] [H] DoS in fee recomputation (`calculateFeeAndNewAmountForOneTVS`) due to uninitialized `newAmounts` (immediate array out‑of‑bounds)
  
  ### Summary
`FeesManager.calculateFeeAndNewAmountForOneTVS` writes into a memory array `newAmounts` that is never allocated. On the very first loop iteration it attempts `newAmounts[0] = …`, which triggers `panic: array out‑of‑bounds (0x32)`. Both `splitTVS` and `mergeTVS` in `AlignerzVesting` call this helper, so both operations are effectively bricked (hard revert) as soon as the function runs.

### Affected surface and call flow
- `AlignerzVesting.mergeTVS(...)` → calls `calculateFeeAndNewAmountForOneTVS(mergeFeeRate, amounts, nbOfFlows)`.
- `AlignerzVesting.splitTVS(...)` → calls `calculateFeeAndNewAmountForOneTVS(splitFeeRate, amounts, nbOfFlows)`.

Faulty helper (no allocation of `newAmounts`):
```169:175:protocol/src/contracts/vesting/feesManager/FeesManager.sol
function calculateFeeAndNewAmountForOneTVS(
    uint256 feeRate,
    uint256[] memory amounts,
    uint256 length
) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    // BUG: missing allocation → newAmounts has length 0
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount; // panics: array out-of-bounds (0x32)
        // (Loop increment is a separate issue, see H‑05)
    }
}
```

### Root cause
- `newAmounts` is declared but never initialized (`new uint256[](length)` is missing). Any write to `newAmounts[i]` is an out‑of‑bounds access when `i==0`.

### Impact (availability)
- **High severity DoS**: Any attempt to split or merge a TVS reverts immediately with `panic: array out‑of‑bounds (0x32)`. There is no user‑land workaround; core portfolio management operations are unusable.

### PoC (from tests)

```solidity
    function test_SSplitTVS_DOS_CalculateFeeLoop() public {
        // Arrange: set non-zero fees to ensure fee calc is exercised
        vm.prank(projectCreator);
        vesting.setFees(0, 0, 100, 100); // splitFeeRate=100 bp, mergeFeeRate=100 bp

        // Launch a reward project and mint one TVS (single flow) to a KOL
        address kol = bidders[10];
        uint256 tvsAmount = 10 ether;
        uint256 vestingPeriod = 90 days;

        vm.prank(projectCreator);
        vesting.launchRewardProject(address(token), address(usdt), block.timestamp, 7 days);
        address[] memory kols = new address[](1);
        uint256[] memory amounts = new uint256[](1);
        kols[0] = kol;
        amounts[0] = tvsAmount;
        vm.prank(projectCreator);
        vesting.setTVSAllocation(0, tvsAmount, vestingPeriod, kols, amounts);

        vm.prank(kol);
        vesting.claimRewardTVS(0);
        uint256 splitNftId = 1; // first minted NFT in ERC721A

        // Act + Assert: calling splitTVS triggers calculateFeeAndNewAmountForOneTVS missing i++ → DOS (revert)
        uint256[] memory perc = new uint256[](2);
        perc[0] = 5000;
        perc[1] = 5000;
        vm.prank(kol);
        vm.expectRevert();
        vesting.splitTVS(0, perc, splitNftId);
    }

  function test_MergeTVS_DOS_CalculateFeeLoop() public {
        // Arrange: set non-zero fees to ensure fee calc is exercised
        vm.prank(projectCreator);
        vesting.setFees(0, 0, 100, 100); // splitFeeRate=100 bp, mergeFeeRate=100 bp

        // Launch a reward project and mint two TVSs to two KOLs, then transfer to a single owner
        address kolA = bidders[11];
        address kolB = bidders[12];
        uint256 tvsAmount = 10 ether;
        uint256 vestingPeriod = 90 days;

        vm.prank(projectCreator);
        vesting.launchRewardProject(address(token), address(usdt), block.timestamp, 7 days);
        address[] memory kols = new address[](2);
        uint256[] memory amounts = new uint256[](2);
        kols[0] = kolA;
        kols[1] = kolB;
        amounts[0] = tvsAmount;
        amounts[1] = tvsAmount;
        vm.prank(projectCreator);
        vesting.setTVSAllocation(0, tvsAmount * 2, vestingPeriod, kols, amounts);

        // claim both NFTs
        vm.prank(kolA);
        vesting.claimRewardTVS(0); // nftId 1
        vm.prank(kolB);
        vesting.claimRewardTVS(0); // nftId 2

        // transfer nftId 2 to kolA so kolA owns both
        vm.prank(kolB);
        nft.approve(kolA, 2);
        vm.prank(kolB);
        nft.transferFrom(kolB, kolA, 2);

        // Act + Assert: calling mergeTVS triggers calculateFeeAndNewAmountForOneTVS missing i++ → DOS (revert)
        uint256[] memory projIds = new uint256[](1);
        uint256[] memory toMerge = new uint256[](1);
        projIds[0] = 0;
        toMerge[0] = 2;
        vm.prank(kolA);
        vm.expectRevert();
        vesting.mergeTVS(0, 1, projIds, toMerge);
    }
```

Test Result:

```solidity
    ├─ [0] VM::prank(bidder10: [0x6F9bF90C0eC8BC3b77aAECE12eEEb19bc78C5D6f])
    │   └─ ← [Return]
    ├─ [0] VM::expectRevert(custom error 0xf4844814)
    │   └─ ← [Return]
    ├─ [14995] ERC1967Proxy::fallback(0, [5000, 5000], 1)
    │   ├─ [14233] AlignerzVesting::splitTVS(0, [5000, 5000], 1) [delegatecall]
    │   │   ├─ [4318] AlignerzNFT::extOwnerOf(1) [staticcall]
    │   │   │   └─ ← [Return] bidder10: [0x6F9bF90C0eC8BC3b77aAECE12eEEb19bc78C5D6f]
    │   │   └─ ← [Revert] panic: array out-of-bounds access (0x32)
    │   └─ ← [Revert] panic: array out-of-bounds access (0x32)
    └─ ← [Return]

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 2.73s (1.63ms CPU time)
```

### Remediation
Allocate and write within bounds; also compute per‑flow fee and use a proper loop increment (the increment aspect is covered in H‑05):
```diff
  function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
+      newAmounts = new uint256[](length);
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
        }
    }
```
  