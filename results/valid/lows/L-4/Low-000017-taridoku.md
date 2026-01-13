# [000017] Medium: KOL can DOS `distributeRemainingRewardTVS`
  
  ### Summary

A malicious KOL can intentionally provide a protocol with a contract address or smart wallet that does not implement `onERC721Received`. Considering `distributeRemainingRewardTVS` only iterates over TVS that hasnt been claimed without skipping failed claims, the function will always lead to a DOS when called.

### Root Cause

`distributeRemainingRewardTVS` is called when the claim deadline is passed and KOLs have not claimed their rewards. It is owner gated, so the owner calls this to distribute rewards to the KOLs post deadline. This function calls the mint in `AlignersNFT.sol` for all affected KOLs. 
The `AlignersNFT.sol` contract calls `ERC721A`'s `_safeMint` which in turn leads to the mint function. As a standard NFT contract, `mint` checks if the receiver (`to`) is a contract or not. If it is, the function expects it to have the `onERC721Received` function or revert:
```solidity
            if (safe && isContract(to)) {
                do {
                    emit Transfer(address(0), to, updatedIndex);
                    if (!_checkContractOnERC721Received(address(0), to, updatedIndex++, _data)) {
                        revert TransferToNonERC721ReceiverImplementer();
                    }
// ...
```
This can be used to brick the function. 
Because the `distributeRemainingRewardTVS` iterates over all unclaimed without skipping fails, a malicious KOL which could be a smart wallet or contract that does not implement `onERC721Received`, causes a revert, which by extension reverts the entire transaction. 


### Internal Pre-conditions

1. `distributeRemainingRewardTVS` iterates over all KOLs without skipping failed mints
2. After `claimDeadline`, some KOLs still haven’t claimed

### External Pre-conditions

1. A KOL provides a contract address or smart wallet which gets registered by the protocol in `setTVSAllocation`

### Attack Path

1. A malicious KOL provides a smart wallet or contract address (`BadKol`) which gets registered 
2. Users/KOLs can claim individually before `claimDeadline`
3. After claimDeadline, some KOLs still haven’t claimed
4. Protocol attempts to be force distribute remaining rewards using `distributeRemainingRewardTVS(rewardProjectId)`
5. Loop reaches `BadKol`
- `nftContract.mint(BadKol)` calls `_safeMint` -> `_checkContractOnERC721Received` ->revert.
- Entire tx reverts. No one gets their forced distribution.
6. Any future attempt to call distributeRemainingRewardTVS will hit BadKol again and revert again

### Impact

Impact: Medium
`distributeRemainingRewardTVS` is permanently unusable for that `rewardProjectId`. It also breaks core functionality as TVS rewards are expected to be distributable even if KOLs arent able to claim before deadline runs out.

Likelihood: Medium
The protocol chooses KOL addresses. However, Smart wallets are increasingly in use. It is possible for one to be registered. It is also possible that not all smart wallets implement the `ERC721Receiver` properly and so a benign user could also hit this.


### PoC

Add this to `test/AlignerzProtocolVestingTest.t.sol`
```solidity
 function test_DistributeRemainingRewardTVS_brickedByNonERC721ReceiverKol() public {
        // Malicious / non-compliant KOL contract
        BadKol badKol = new BadKol();

        // Reward projects are owned by projectCreator (vesting.transferOwnership(projectCreator) in setUp)
        vm.startPrank(projectCreator);

        // 1) Launch a reward project
        // If launchRewardProject returns an ID, keep this. If it doesn't, use `uint256 rewardProjectId = 0;`
        vesting.launchRewardProject(
            address(token),
            address(usdt),
            block.timestamp, // startTime
            1 days           // claimWindow
        );
        uint256 rewardProjectId = 0;
        // 2) Set TVS allocation for two KOLs: one EOA and one bad contract
        address[] memory kols = new address[](2);
        kols[0] = bidders[0];           // normal EOA from setUp
        kols[1] = address(badKol);      // contract that will break safeMint

        uint256[] memory amounts = new uint256[](2);
        amounts[0] = 100 ether;
        amounts[1] = 200 ether;

        uint256 totalTVSAllocation = amounts[0] + amounts[1];
        uint256 vestingPeriod = 90 days;

        vesting.setTVSAllocation(
            rewardProjectId,
            totalTVSAllocation,
            vestingPeriod,
            kols,
            amounts
        );

        vm.stopPrank();

        // 3) Move time past claimDeadline so distributeRemainingRewardTVS can be called
        vm.warp(block.timestamp + 2 days);

        // 4) Owner tries to sweep remaining TVS to all KOLs.
        // When the loop hits badKol, nftContract.mint(badKol) -> _safeMint -> _checkContractOnERC721Received -> revert.
        vm.expectRevert(); // will hit TransferToNonERC721ReceiverImplementer from ERC721A
        vm.prank(projectCreator);
        vesting.distributeRemainingRewardTVS(rewardProjectId);
    }

}

contract BadKol {
    // No onERC721Received implementation.
    // Any safeMint to this contract will revert with TransferToNonERC721ReceiverImplementer.
}
```

### Mitigation

Catch failures during mint so other KOLs still get their NFTs and vesting.
```solidity 
for (uint256 i = len - 1; rewardProject.kolTVSAddresses.length > 0;) {
    address kol = rewardProject.kolTVSAddresses[i];
    uint256 amount = rewardProject.kolTVSRewards[kol];

    if (amount > 0) {
        rewardProject.kolTVSRewards[kol] = 0;

        bool success;
        uint256 nftId;

        // Wrap mint in try/catch so we don't revert the entire sweep
        try nftContract.mint(kol) returns (uint256 mintedId) {
            success = true;
            nftId = mintedId;
        } catch {
            success = false;
        }

        if (success) {
            // write vesting data, emit event, etc
            rewardProject.allocations[nftId].amounts.push(amount);
            ...
            emit RewardTVSDistributed(rewardProjectId, kol, nftId, amount, vestingPeriod);
        } else {
            // optional: re-credit the amount or mark it as failed distribution
            rewardProject.kolTVSRewards[kol] = amount;
            emit RewardTVSDistributionFailed(rewardProjectId, kol, amount);
        }
    }
    rewardProject.kolTVSAddresses.pop();
    unchecked { --i; }
}
```
  