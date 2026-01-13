# [001057] [H] DoS in fee recomputation (`calculateFeeAndNewAmountForOneTVS`) due to missing loop increment (infinite loop / gas exhaustion)
  
  **NOTE**
We discussed with @0xjarix that this issue has the same cause as [this](https://github.com/dualguard/2025-11-alignerz-konstantinvelev/issues/1), but the impact is different. In my opinion, this should be a different issue because it affects different parts of the code, which means the path to solve the problem is different, the effects are different, and the results are also different. Also, fixing one of them will not fix the other, which again confirms that it should be considered a separate issue.
### Summary

There is another issue related with `calculateFeeAndNewAmountForOneTVS` function that makes DoSes the protocol, but the root couse is different. After fixing H‑04 (allocating `newAmounts`), the helper still does not terminate because the `for (uint256 i; i < length;)` loop never increments `i`. As soon as `splitTVS` or `mergeTVS` invoke this helper, the transaction will spin until gas is exhausted. Without incrementing the index `i` we are basically create `while(true)` loop and this is the issue for the bug. Obviously we won't eat the whole gas because we will countinue substract `feeAmount` from the `amounts[i]` which is exactly the same amount every singe time because the `i` stays unchanged. So bassicaly the protocol would looping and substracting the fee from the amount until eventually we are reach a point where `amounts[i] < feeAmount` and this is where the contract revert with the `[Revert] panic: arithmetic underflow or overflow (0x11)`, leading to revert for the whole function and inablility for the contract to **merge** or **split** TVSs.

### Root cause
- The loop header omits `++i`, and the loop body never updates `i`:
```169:175:protocol/src/contracts/vesting/feesManager/FeesManager.sol
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        // @audit: we are missing i++ here leading to infinite loop and DoS
        newAmounts = new uint256[](length);
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
        }
    }
```

### Impact (availability)
- **High severity DoS**: `splitTVS` and `mergeTVS` remain unusable because underflow panic error or OOG.

### PoC (how to reproduce)
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
     │   └─ ← [Return]
    ├─ [18122] AlignerzNFT::transferFrom(bidder12: [0x410bb6864c0838A132619459cD22ADf905440835], bidder11: [0x0b5520Db6F7AB2229DF631bd088578C44Fb69c58], 2)
    │   ├─ emit Approval(owner: bidder12: [0x410bb6864c0838A132619459cD22ADf905440835], approved: 0x0000000000000000000000000000000000000000, tokenId: 2)
    │   ├─ emit Transfer(from: bidder12: [0x410bb6864c0838A132619459cD22ADf905440835], to: bidder11: [0x0b5520Db6F7AB2229DF631bd088578C44Fb69c58], tokenId: 2)
    │   └─ ← [Return]
    ├─ [0] VM::prank(bidder11: [0x0b5520Db6F7AB2229DF631bd088578C44Fb69c58])
    │   └─ ← [Return]
    ├─ [0] VM::expectRevert(custom error 0xf4844814)
    │   └─ ← [Return]
    ├─ [153998] ERC1967Proxy::fallback(0, 1, [0], [2])
    │   ├─ [153224] AlignerzVesting::mergeTVS(0, 1, [0], [2]) [delegatecall]
    │   │   ├─ [4318] AlignerzNFT::extOwnerOf(1) [staticcall]
    │   │   │   └─ ← [Return] bidder11: [0x0b5520Db6F7AB2229DF631bd088578C44Fb69c58]
    │   │   └─ ← [Revert] panic: arithmetic underflow or overflow (0x11)
    │   └─ ← [Revert] panic: arithmetic underflow or overflow (0x11)
    └─ ← [Return]

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 3.07s (1.47ms CPU time)

Ran 1 test suite in 3.09s (3.07s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

### Remediation
- Use a standard counting loop and compute per‑flow fee, not a cumulative subtraction per element:
```diff
function calculateFeeAndNewAmountForOneTVS(
    uint256 feeRate,
    uint256[] memory amounts,
    uint256 /* length */
) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    uint256 n = amounts.length;
    newAmounts = new uint256[]>(n);
-    for (uint256 i = 0; i < n;) {
+    for (uint256 i = 0; i < n; i++) {
        uint256 perFlowFee = (amounts[i] * feeRate) / BASIS_POINT;
        feeAmount += perFlowFee;
        newAmounts[i] = amounts[i] - perFlowFee;     // do not subtract cumulative fee per element
    }
}
```
  