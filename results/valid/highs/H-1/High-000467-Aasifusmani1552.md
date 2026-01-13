# [000467] The `splitTVS` function uses storage pointer of allocation struct of original nft instead of using the memory snapshot, this corrupts storage in between the function call .
  
  ### Summary

The `splitTVS` function in `AlignerzVesting` contract splits the given `splitNftId` into various Nfts. The function splits the original nft (`splitNftId`) into the number of percentages given by the user who is splitting their Nft. But the function have a critical flaw. The `splitTVS` function is mutating the **storage allocation** (it overwrites `allocation` to `newAllocation` in `_assignAllocation` function) inside the same function, it calls `_computeSplitArrays(allocation, ...)` repeatedly — but because `allocation` is the **storage reference** and is changed/mutated inside the `_assignAllocation` function , the later iterations see the **already-modified** `allocation` (and other fields). That makes the split use the **post-fee / post-mutation** state instead of the original snapshot of original nft for each new NFT, producing incorrect splits.

Here is what's happening in the `splitTVS` function thoroughly:-
- `allocation` is an **Allocation storage reference** to contract storage of the original nft
- The function compute `(feeAmount, newAmounts)` then assign `allocation.amounts = newAmounts`.
- In the for loop, function calls `_computeSplitArrays(allocation, percentage, nbOfFlows)` — the function returns the splitted `allocation` struct. 
- Now, inside the for loop, **the function gets the storage pointer of the original nft** instead of getting its snapshot, here
```solidity
@>           Allocation storage newAlloc = isBiddingProject
                ? biddingProjects[projectId].allocations[nftId]
                : rewardProjects[projectId].allocations[nftId];
            _assignAllocation(newAlloc, alloc);
```
- Because `_assignAllocation` modified the storage of original nft (`splitNftId`), the `allocation` pointer used in the `_computeSplitArrays(allocation, percentage, nbOfFlows)` function is also modified, because it is storage pointer, not a memory snapshot.
- Result: first split uses **original values**, subsequent splits use already-halved amounts or otherwise mutated data → incorrect final allocations.

### Root Cause

The root cause of this issue is using storage pointer of `allocation`  struct of original nft in `_computeSplitArrays` internal function inside `splitTVS` function, instead of using its original snapshot, here:-

```solidity
    function splitTVS(...)
        external
        returns (uint256, uint256[] memory)
    {
        address nftOwner = nftContract.extOwnerOf(splitNftId);
        require(msg.sender == nftOwner, Caller_Should_Own_The_NFT());

        bool isBiddingProject = NFTBelongsToBiddingProject[splitNftId];
@>        (Allocation storage allocation, IERC20 token) = isBiddingProject                                  //here getting the storage pointer
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

        uint256 sumOfPercentages;
        for (uint256 i; i < nbOfTVS;) {
            uint256 percentage = percentages[i];
            sumOfPercentages += percentage;

            uint256 nftId = i == 0 ? splitNftId : nftContract.mint(msg.sender);
            if (i != 0) newNftIds[i - 1] = nftId;
 @>           Allocation memory alloc = _computeSplitArrays(allocation, percentage, nbOfFlows); //here using the storage pointer instead of memory snapshot
            NFTBelongsToBiddingProject[nftId] = isBiddingProject ? true : false;
@>            Allocation storage newAlloc = isBiddingProject
                ? biddingProjects[projectId].allocations[nftId]
                : rewardProjects[projectId].allocations[nftId]; //here storage pointer is extracted
            _assignAllocation(newAlloc, alloc); //and here it is modified
            allocationOf[nftId] = newAlloc;
            //......rest of the logic
    }
```

### Internal Pre-conditions

N/A

### External Pre-conditions

N/A

### Attack Path

1. Actors involved:-
- Protocol admin
- Alice
- Thomas
- Bob

2. Attack path:-
- `Admin` launches and `biddingProject`.
- `Admin` creates a pool with 3000 tokens to be vested with 1 usdt per token price.
- `Alice`, `Thomas`, and `Bob` place their bids with equal amount of usdt. They vest for different periods.
- `Alice` and `Thomas` vests for 180 days and `Bob` vests for 90 days.
- `Admin` finalizes the bidding, all of the bidders get their bids accepted and they recieved 100 tokens each.
- All of the bidders claim their nfts.
- Now, `Bob` sells  his nft to `Thomas`.
- `Thomas` merges his and `Bob`'s nft (he bought from `Bob`) together into his own nft. This creates flow 1 with `Thomas`'s nft and flow 2 with `Bob`'s nft.
- After 90 days, `Thomas` claims tokens, because 90 days passed, `Thomas` claims half tokens from flow 1 and full amount of tokens from flow 2. (Because `Bob` only vested for 90 days).
- Now, `Thomas` decides to split his nft equally, 50% of amount in each nft.
- Because of the incorrect use of storage pointer of `allocation` struct instead of its snaptshot, the `splitTVS` function splits nfts incorrectly effectively loosing aroung 25% of `Thomas`'s tokens.
- Now after 90 days, both `Alice` and `Thomas` claim their tokens, but because of incorrect splitting, around 25% of `Thomas`'s tokens remain in the project.

### Impact

1. **Incorrect split handling** — split NFTs do not represent proportional parts of the original NFT.
2. **Protocol accounting mismatch** — fees and payouts no longer line up with liabilities because of incorrect splitting.
3. **User financial harm** — users gets underpaid or recipients of split NFTs have wrong entitlements.
4. **Determinism lost** — split outcomes depend on internal ordering and side-effects. The split of original nft decides the loss.
5. **Can be exploited** — A malicious user can use the attack path to grief users.

**Severity - HIGH**

### PoC
- To run the PoC, i modified the `calculateFeeAndNewAmountForOneTVS` function because of broken loop, uninitialized array and fee assimilation issues. Please modify the function as follows to run the PoC (required for the mergeTVS function to run):-
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
- To get the verbose output and pinpoint the vulnerability, i have added some console log statements in the `AlignerzVesting` contract. And because the `_computeSplitArrays` function was broken because of out of bound array access (already submitted the issue), i have also modified the function to work properly. So,  Please use the modified `_computeSplitArrays` function , here:-
```solidity
//import at the top with other imports
import {console} from "forge-std/console.sol";

function _computeSplitArrays(Allocation storage allocation, uint256 percentage, uint256 nbOfFlows)
        internal
        view
        returns (Allocation memory alloc)
    {
        uint256[] memory baseAmounts = allocation.amounts;
        console.log("base amount[0]:", baseAmounts[0]);
        console.log("base amount[1]:", baseAmounts[1]);
        uint256[] memory baseVestings = allocation.vestingPeriods;
        uint256[] memory baseVestingStartTimes = allocation.vestingStartTimes;
        uint256[] memory baseClaimed = allocation.claimedSeconds;
        bool[] memory baseClaimedFlows = allocation.claimedFlows;
        console.log("base Claimed flow[0]:", baseClaimedFlows[0]);
        console.log("base Claimed flow[1]:", baseClaimedFlows[1]);

        //added this sanity check
        require(
            nbOfFlows == baseAmounts.length && nbOfFlows == baseVestings.length
                && nbOfFlows == baseVestingStartTimes.length && nbOfFlows == baseClaimed.length
                && nbOfFlows == baseClaimedFlows.length,
            "Invalid nbOfFlows"
        );
       
        //added these to allocate the storage and get rid of out of bound array access error
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
- Please copy/paste the given test into `AlignerzVestingProtocolTest.t.sol` test suite and run with command `forge test --mt test_splitTVSincorrectlyUseStorageInsteadOfMemoryDuringTheCallPoC -vvvv` to get verbose output:-

**PoC:-**
```solidity
function test_splitTVSincorrectlyUseStorageInsteadOfMemoryDuringTheCallPoC() public {
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

        console.log("protocol balance of token after nft claim:", token.balanceOf(address(vesting)));

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

        //Because 90 days have passed, thomas can claim half of tokens from flow 1 because vesting time for flow 1 is 180 days
        //And he can claim full amount of tokens from flow 2 because vesting time for flow 2 is 90 days (set by Bob)
        assertEq(token.balanceOf(address(vesting)), 1500e18);

        console.log("protocol balance of token after thomas made his first claim:", token.balanceOf(address(vesting)));

        /*Now here happens the issue. When thomas calls splitTVs,
        the function in the first loop sets the original nft amount
        to the given percentage. But because the splitTVS function
        uses storage pointer of original nft to calculate the percentage for subsequent nfts,
        the new nfts to be minted from this point use the overwritten amount
        instead of the original amount that was supposed to be split into
        given percentages. This results in loss of user tokens.*/

        //creating paramters required to split
        uint256[] memory percentages = new uint256[](2);
        percentages[0] = 5000; //50%
        percentages[1] = 5000; //50%

        vm.prank(thomas);
        (, uint256[] memory nftIdThomasNew) = vesting.splitTVS(0, percentages, nftIdThomas);
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
        vesting.claimTokens(0, nftIdAlice);

        //Now after all claims, protocol balance should be 0, but because split nft incorrectly
        //splits amounts between nfts, thomas's claims very less amount of token than intended
        assert(token.balanceOf(address(vesting)) > 0);
    }
```

**Logs:-**
```solidity
protocol balance of token now: 3000000000000000000000
  protocol balance of token after nft claim: 3000000000000000000000
  protocol balance of token after thomas made his first claim: 1500000000000000000000
  base amount[0]: 980000000000000000000
  base amount[1]: 980000000000000000000
  base Claimed flow[0]: false
  base Claimed flow[1]: true
  base amount[0]: 490000000000000000000           //how balance halved, because storage was modified
  base amount[1]: 490000000000000000000
  base Claimed flow[0]: false
  base Claimed flow[1]: true
  protocol balance of token after thomas called split tvs: 1460000000000000000000
  protocol balance of token after thomas made complete claim: 1092500000000000000000
```

**Logs:-**
```solidity
    ├─ [1306] Aligners26::balanceOf(ERC1967Proxy: [0xc7183455a4C133Ae270771860664b6B7ec320bB1]) [staticcall]
    │   └─ ← [Return] 92500000000000000000 [9.25e19]
    └─ ← [Return]

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 2.89s (8.95ms CPU time)
```

### Mitigation

To mitigate the issue, use the snapshot of `allocation` struct of the original nft instead of using the storage pointer. And also modify the `_computeSplitArrays` function to expect `Allocation memory allocation` instead of storage because it doesn't modify and storage. Here are the revised function:-

```diff
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

+        Allocation memory malloc = allocation;

        uint256 sumOfPercentages;
        for (uint256 i; i < nbOfTVS;) {
            uint256 percentage = percentages[i];
            sumOfPercentages += percentage;

            uint256 nftId = i == 0 ? splitNftId : nftContract.mint(msg.sender);
            if (i != 0) newNftIds[i - 1] = nftId;
+          Allocation memory alloc = _computeSplitArrays(malloc, percentage, nbOfFlows);
-           Allocation memory alloc = _computeSplitArrays(allocation, percentage, nbOfFlows);

          //---rest of the logic
    }

-     function _computeSplitArrays(Allocation storage allocation, uint256 percentage, uint256 nbOfFlows)
+    function _computeSplitArrays(Allocation memory allocation, uint256 percentage, uint256 nbOfFlows)

```
  