# [000639] [ High ]splitTVS Function Always Reverts Due to Uninitialized Memory Array, Causing a Denial of Service of a Core Feature
  
  ### Summary

An uninitialized memory array in the `_computeSplitArrays ` helper function will cause a fatal EVM panic (0x32) for any TVS holder as the user will trigger this panic simply by attempting to call the `splitTVS function`, rendering a core protocol feature unusable.

### Root Cause


In [src/contracts/vesting/AlignerzVesting.sol](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1113-#L1143)  inside the `_computeSplitArrays` function, the `Allocation memory alloc` struct is declared, but its internal dynamic arrays `(amounts, vestingPeriods, vestingStartTimes, claimedSeconds, claimedFlows) `are never initialized. The subsequent loop immediately tries to write to indices of these zero-length arrays, causing an array out-of-bounds access.

### Internal Pre-conditions

1.) A user (bidder or KOL) needs to own a valid TVS NFT.

2.) The TVS NFT must have at least one vesting flow (which is guaranteed upon minting).

### External Pre-conditions

None

### Attack Path

- A user (bidder) owns a valid `nftId` for `projectId 0`.

- The user decides to split their TVS into two 50% shares.

- The user prepares the percentages array: `[5000, 5000]`.

- The user calls `vesting.splitTVS(0, [5000, 5000], nftId)`.

- The splitTVS function internally calls _computeSplitArrays to calculate the new allocations.

- `_computeSplitArrays` attempts to write to alloc.amounts[0], but `alloc.amounts` is a memory array of length 0.

- The transaction immediately reverts with a Panic(0x32) (array out-of-bounds) error.

### Impact

All users `(TVS holders)` cannot execute the `splitTVS` function under any circumstances. This is a complete Denial of Service for a core protocol feature, preventing users from managing their vested positions, which could impact their ability to sell or transfer partial stakes.

### PoC

Create A new file and in the test directory and add this:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import "@openzeppelin/contracts/utils/cryptography/MerkleProof.sol";
import {Aligners26} from "../src/contracts/token/Aligners26.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";
import {IAlignerzVesting} from "../src/interfaces/IAlignerzVesting.sol";
import {Upgrades} from "openzeppelin-foundry-upgrades/Upgrades.sol";
import {CompleteMerkle} from "murky/src/CompleteMerkle.sol";

contract AlignerzVestingProtocolTest is Test {
    AlignerzVesting public vesting;
    Aligners26 public token;
    AlignerzNFT public nft;
    MockUSD public usdt;

    address public owner;
    address public projectCreator;
    address[] public bidders;

    // Constants
    uint256 constant NUM_BIDDERS = 20;
    uint256 constant TOKEN_AMOUNT = 26_000_000 ether;
    uint256 constant BIDDER_USD = 1_000 ether;
    uint256 constant PROJECT_ID = 0;

    // Project structure for organization
    struct BidInfo {
        address bidder;
        uint256 amount;
        uint256 vestingPeriod;
        uint256 poolId;
        bool accepted;
    }

    // Track allocated bids and their proofs
    mapping(address => bytes32[]) public bidderProofs;
    mapping(address => uint256) public bidderPoolIds;
    mapping(address => uint256) public bidderNFTIds;
    bytes32 public refundRoot;

    function setUp() public {
        owner = address(this);
        projectCreator = makeAddr("projectCreator");
        vm.deal(projectCreator, 100 ether);

        // Deploy contracts
        usdt = new MockUSD();
        token = new Aligners26("26Aligners", "A26Z");
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");

        address payable proxy = payable(Upgrades.deployUUPSProxy(
            "AlignerzVesting.sol",
            abi.encodeCall(AlignerzVesting.initialize, (address(nft))))
        );
        vesting = AlignerzVesting(proxy);

        // Set NFT minter to vesting contract
        vm.prank(owner);
        nft.addMinter(proxy);
        vesting.setTreasury(address(1));

        // Create bidders with ETH and USDT
        for (uint256 i = 0; i < NUM_BIDDERS; i++) {
            address bidder = makeAddr(string.concat("bidder", vm.toString(i)));
            vm.deal(bidder, 50 ether);
            bidders.push(bidder);
            usdt.mint(bidder, BIDDER_USD * 2); // Give them extra USDT for multiple bids
        }

        // Mint tokens for project creator
        token.transfer(projectCreator, TOKEN_AMOUNT);
        vesting.transferOwnership(projectCreator);

        // Approve tokens for vesting contract
        vm.prank(projectCreator);
        token.approve(address(vesting), TOKEN_AMOUNT);
        vm.stopPrank(); // <-- ** THE FIX IS HERE **
    }

    // Helper function to create a leaf node for the merkle tree
    function getLeaf(address bidder, uint256 amount, uint256 projectId, uint256 poolId)
        internal
        pure
        returns (bytes32)
    {
        return keccak256(abi.encodePacked(bidder, amount, projectId, poolId));
    }

    // Helper for generating merkle proofs
    function generateMerkleProofs(BidInfo[] memory bids, uint256 poolId) internal returns (bytes32) {
        bytes32[] memory leaves = new bytes32[](bids.length);
        uint256 leafCount = 0;

        // Create leaves for each bid in this pool
        for (uint256 i = 0; i < bids.length; i++) {
            if (bids[i].poolId == poolId && bids[i].accepted) {
                leaves[leafCount] = getLeaf(bids[i].bidder, bids[i].amount, PROJECT_ID, poolId);

                bidderPoolIds[bids[i].bidder] = poolId;
                leafCount++;
            }
        }

        CompleteMerkle m = new CompleteMerkle();
        bytes32 root = m.getRoot(leaves);
        uint256 indexTracker = 0;
        for (uint256 i = 0; i < bids.length; i++) {
            if (bids[i].poolId == poolId && bids[i].accepted) {
                bytes32[] memory proof = m.getProof(leaves, indexTracker);
                bidderProofs[bids[i].bidder] = proof;
                indexTracker++;
            }
        }

        return root;
    }

    // Helper for generating refund proofs
    function generateRefundProofs(BidInfo[] memory bids) internal returns (bytes32) {
        bytes32[] memory leaves = new bytes32[](bids.length);
        uint256 leafCount = 0;
        uint256 poolId = 0;

        // Create leaves for each bid in this pool
        for (uint256 i = 0; i < bids.length; i++) {
            if (!bids[i].accepted) {
                leaves[leafCount] = getLeaf(bids[i].bidder, bids[i].amount, PROJECT_ID, poolId);

                bidderPoolIds[bids[i].bidder] = poolId;
                leafCount++;
            }
        }

        CompleteMerkle m = new CompleteMerkle();
        bytes32 root = m.getRoot(leaves);
        uint256 indexTracker = 0;
        for (uint256 i = 0; i < bids.length; i++) {
            if (!bids[i].accepted) {
                bytes32[] memory proof = m.getProof(leaves, indexTracker);
                bidderProofs[bids[i].bidder] = proof;
                indexTracker++;
            }
        }

        return root;
    }

    function test__olaoyesalem_SplitTVS_Revert() public {
       
     // It demonstrates that splitTVS will *always* revert
        // due to the uninitialized memory array bug.

        address bidder = bidders[0];
        address dummy = bidders[1]; // To make Merkle tree valid
        uint256 bidAmount = 1000 ether;
        uint256 projectId = 0;

        // --- 1. SETUP PROJECT & GET ONE NFT ---
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);
        vesting.launchBiddingProject(address(token), address(usdt), block.timestamp, block.timestamp + 1_000_000, "0x0", true);
        vesting.createPool(projectId, 3_000_000 ether, 0.01 ether, true);
        
        address[] memory bidderList = new address[](2);
        bidderList[0] = bidder;
        bidderList[1] = dummy;
        vesting.addUsersToWhitelist(bidderList, projectId);
        vm.stopPrank();

        vm.startPrank(bidder);
        usdt.approve(address(vesting), bidAmount);
        vesting.placeBid(projectId, bidAmount, 90 days);
        vm.stopPrank();

        // Create tree with 2 leaves
        bytes32[] memory leaves = new bytes32[](2);
        leaves[0] = getLeaf(bidder, bidAmount, projectId, 0);
        leaves[1] = getLeaf(dummy, 0, projectId, 0);
        CompleteMerkle m = new CompleteMerkle();
        bytes32 root = m.getRoot(leaves);
        bytes32[] memory proof = m.getProof(leaves, 0);
        
        vm.prank(projectCreator);
        bytes32[] memory poolRoots = new bytes32[](1);
        poolRoots[0] = root;
        vesting.finalizeBids(projectId, bytes32(0), poolRoots, 60);

        vm.prank(bidder);
        uint256 nftId = vesting.claimNFT(projectId, 0, bidAmount, proof);

        // --- 2. PREPARE SPLIT & EXPECT REVERT ---
        uint256[] memory percentages = new uint256[](2);
        percentages[0] = 5000;
        percentages[1] = 5000;

        // We expect the EVM to panic with 0x32 (array out-of-bounds)
        // The panic selector is 0x4e487b71 followed by the error code 0x32
        vm.expectRevert(abi.encodeWithSignature("Panic(uint256)", 0x32));

        // --- 3. TRIGGER THE BUG ---
        vm.prank(bidder);
        vesting.splitTVS(projectId, percentages, nftId);
    
    }

}
```


Run this `forge clean && forge test --mt test__olaoyesalem_SplitTVS_Revert -vvvv`

Result:

```js
forge clean && forge test --mt test__olaoyesalem_SplitTVS_Revert -vvvv
[⠊] Compiling...
[⠒] Compiling 96 files with Solc 0.8.29
[⠔] Solc 0.8.29 finished in 16.10s
Compiler run successful with warnings:
Warning (2018): Function state mutability can be restricted to pure
  --> src/MockUSD.sol:13:5:
   |
13 |     function decimals() public view override returns (uint8) {
   |     ^ (Relevant source part starts here and spans across multiple lines).


Ran 1 test for test/new.t.sol:AlignerzVestingProtocolTest
[PASS] test__olaoyesalem_SplitTVS_Revert() (gas: 2061171)
Logs:
  npm warn exec The following package was not found and will be installed: @openzeppelin/upgrades-core@1.44.2


Traces:
  [2081071] AlignerzVestingProtocolTest::test__olaoyesalem_SplitTVS_Revert()
    ├─ [0] VM::startPrank(projectCreator: [0x3e221db247A1Dd4A0724cf57b6b05304A2DcA513])
    │   └─ ← [Return]
    ├─ [17429] ERC1967Proxy::fallback(1)
    │   ├─ [12198] AlignerzVesting::setVestingPeriodDivisor(1) [delegatecall]
    │   │   ├─ emit vestingPeriodDivisorUpdated(oldVestingPeriodDivisor: 2592000 [2.592e6], newVestingPeriodDivisor: 1)
    │   │   └─ ← [Return] true
    │   └─ ← [Return] true
    ├─ [168518] ERC1967Proxy::fallback(Aligners26: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], MockUSD: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], 1, 1000001 [1e6], 0x3078300000000000000000000000000000000000000000000000000000000000, true)
    │   ├─ [167763] AlignerzVesting::launchBiddingProject(Aligners26: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], MockUSD: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], 1, 1000001 [1e6], 0x3078300000000000000000000000000000000000000000000000000000000000, true) [delegatecall]
    │   │   ├─ emit BiddingProjectLaunched(projectId: 0, projectName: Aligners26: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], stablecoinAddress: MockUSD: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], startTime: 1, endTimeHash: 0x3078300000000000000000000000000000000000000000000000000000000000)
    │   │   └─ ← [Return]
    │   └─ ← [Return]
    ├─ [104441] ERC1967Proxy::fallback(0, 3000000000000000000000000 [3e24], 10000000000000000 [1e16], true)
    │   ├─ [103698] AlignerzVesting::createPool(0, 3000000000000000000000000 [3e24], 10000000000000000 [1e16], true) [delegatecall]
    │   │   ├─ [44322] Aligners26::transferFrom(projectCreator: [0x3e221db247A1Dd4A0724cf57b6b05304A2DcA513], ERC1967Proxy: [0xc7183455a4C133Ae270771860664b6B7ec320bB1], 3000000000000000000000000 [3e24])
    │   │   │   ├─ emit Transfer(from: projectCreator: [0x3e221db247A1Dd4A0724cf57b6b05304A2DcA513], to: ERC1967Proxy: [0xc7183455a4C133Ae270771860664b6B7ec320bB1], value: 3000000000000000000000000 [3e24])
    │   │   │   └─ ← [Return] true
    │   │   ├─ emit PoolCreated(projectId: 0, poolId: 0, totalAllocation: 3000000000000000000000000 [3e24], tokenPrice: 10000000000000000 [1e16], hasExtraRefund: true)
    │   │   └─ ← [Return]
    │   └─ ← [Return]
    ├─ [55619] ERC1967Proxy::fallback([0xf0Ad0970EE36C6FaF41575f3736D081ceEb36083, 0xb549357D1EA5b44c8F1d82046c7dfD6ed2265A8B], 0)
    │   ├─ [54870] AlignerzVesting::addUsersToWhitelist([0xf0Ad0970EE36C6FaF41575f3736D081ceEb36083, 0xb549357D1EA5b44c8F1d82046c7dfD6ed2265A8B], 0) [delegatecall]
    │   │   ├─ emit userWhitelisted(projectId: 0, user: bidder0: [0xf0Ad0970EE36C6FaF41575f3736D081ceEb36083])
    │   │   ├─ emit userWhitelisted(projectId: 0, user: bidder1: [0xb549357D1EA5b44c8F1d82046c7dfD6ed2265A8B])
    │   │   └─ ← [Return]
    │   └─ ← [Return]
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return]
    ├─ [0] VM::startPrank(bidder0: [0xf0Ad0970EE36C6FaF41575f3736D081ceEb36083])
    │   └─ ← [Return]
    ├─ [26927] MockUSD::approve(ERC1967Proxy: [0xc7183455a4C133Ae270771860664b6B7ec320bB1], 1000000000000000000000 [1e21])
    │   ├─ emit Approval(owner: bidder0: [0xf0Ad0970EE36C6FaF41575f3736D081ceEb36083], spender: ERC1967Proxy: [0xc7183455a4C133Ae270771860664b6B7ec320bB1], value: 1000000000000000000000 [1e21])
    │   └─ ← [Return] true
    ├─ [121250] ERC1967Proxy::fallback(0, 1000000000000000000000 [1e21], 7776000 [7.776e6])
    │   ├─ [120513] AlignerzVesting::placeBid(0, 1000000000000000000000 [1e21], 7776000 [7.776e6]) [delegatecall]
    │   │   ├─ [39522] MockUSD::transferFrom(bidder0: [0xf0Ad0970EE36C6FaF41575f3736D081ceEb36083], ERC1967Proxy: [0xc7183455a4C133Ae270771860664b6B7ec320bB1], 1000000000000000000000 [1e21])
    │   │   │   ├─ emit Transfer(from: bidder0: [0xf0Ad0970EE36C6FaF41575f3736D081ceEb36083], to: ERC1967Proxy: [0xc7183455a4C133Ae270771860664b6B7ec320bB1], value: 1000000000000000000000 [1e21])
    │   │   │   └─ ← [Return] true
    │   │   ├─ emit BidPlaced(projectId: 0, user: bidder0: [0xf0Ad0970EE36C6FaF41575f3736D081ceEb36083], amount: 1000000000000000000000 [1e21], vestingPeriod: 7776000 [7.776e6])
    │   │   └─ ← [Return]
    │   └─ ← [Return]
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return]
    ├─ [829502] → new CompleteMerkle@0xa0Cb889707d426A7A386870A03bc70d1b0697598
    │   └─ ← [Return] 4143 bytes of code
    ├─ [3682] CompleteMerkle::getRoot([0xeae26358ff3f56e45642b5e1fd493f43eabdec2daf745a4260c965f656d74003, 0xa90556a9d1924352700b3c99e87e396108749c95b4d8b4a47082e076ee4c000c]) [staticcall]
    │   └─ ← [Return] 0x7777c7f8d2d481b4aae1897a7ffe79e6788a4587c3f5892f68cedeaffd92b4da
    ├─ [3608] CompleteMerkle::getProof([0xeae26358ff3f56e45642b5e1fd493f43eabdec2daf745a4260c965f656d74003, 0xa90556a9d1924352700b3c99e87e396108749c95b4d8b4a47082e076ee4c000c], 0) [staticcall]
    │   └─ ← [Return] [0xa90556a9d1924352700b3c99e87e396108749c95b4d8b4a47082e076ee4c000c]
    ├─ [0] VM::prank(projectCreator: [0x3e221db247A1Dd4A0724cf57b6b05304A2DcA513])
    │   └─ ← [Return]
    ├─ [75266] ERC1967Proxy::fallback(0, 0x0000000000000000000000000000000000000000000000000000000000000000, [0x7777c7f8d2d481b4aae1897a7ffe79e6788a4587c3f5892f68cedeaffd92b4da], 60)
    │   ├─ [74511] AlignerzVesting::finalizeBids(0, 0x0000000000000000000000000000000000000000000000000000000000000000, [0x7777c7f8d2d481b4aae1897a7ffe79e6788a4587c3f5892f68cedeaffd92b4da], 60) [delegatecall]
    │   │   ├─ emit PoolAllocationSet(projectId: 0, poolId: 0, merkleRoot: 0x7777c7f8d2d481b4aae1897a7ffe79e6788a4587c3f5892f68cedeaffd92b4da)
    │   │   ├─ emit BiddingClosed(projectId: 0)
    │   │   └─ ← [Return]
    │   └─ ← [Return]
    ├─ [0] VM::prank(bidder0: [0xf0Ad0970EE36C6FaF41575f3736D081ceEb36083])
    │   └─ ← [Return]
    ├─ [559706] ERC1967Proxy::fallback(0, 0, 1000000000000000000000 [1e21], [0xa90556a9d1924352700b3c99e87e396108749c95b4d8b4a47082e076ee4c000c])
    │   ├─ [558948] AlignerzVesting::claimNFT(0, 0, 1000000000000000000000 [1e21], [0xa90556a9d1924352700b3c99e87e396108749c95b4d8b4a47082e076ee4c000c]) [delegatecall]
    │   │   ├─ [66188] AlignerzNFT::mint(bidder0: [0xf0Ad0970EE36C6FaF41575f3736D081ceEb36083])
    │   │   │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: bidder0: [0xf0Ad0970EE36C6FaF41575f3736D081ceEb36083], tokenId: 1)
    │   │   │   ├─ emit Minted(to: bidder0: [0xf0Ad0970EE36C6FaF41575f3736D081ceEb36083], tokenId: 1)
    │   │   │   └─ ← [Return] 1
    │   │   ├─ emit NFTClaimed(projectId: 0, user: bidder0: [0xf0Ad0970EE36C6FaF41575f3736D081ceEb36083], tokenId: 1, poolId: 0, amount: 1000000000000000000000 [1e21])
    │   │   └─ ← [Return] 1
    │   └─ ← [Return] 1
    ├─ [0] VM::expectRevert(custom error 0xf28dceb3:  $NH{q2)
    │   └─ ← [Return]
    ├─ [0] VM::prank(bidder0: [0xf0Ad0970EE36C6FaF41575f3736D081ceEb36083])
    │   └─ ← [Return]
    ├─ [14961] ERC1967Proxy::fallback(0, [5000, 5000], 1)
    │   ├─ [14199] AlignerzVesting::splitTVS(0, [5000, 5000], 1) [delegatecall]
    │   │   ├─ [4318] AlignerzNFT::extOwnerOf(1) [staticcall]
    │   │   │   └─ ← [Return] bidder0: [0xf0Ad0970EE36C6FaF41575f3736D081ceEb36083]
    │   │   └─ ← [Revert] panic: array out-of-bounds access (0x32)
    │   └─ ← [Revert] panic: array out-of-bounds access (0x32)
    └─ ← [Return]

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 5.57s (6.24ms CPU time)

Ran 1 test suite in 5.58s (5.57s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

### Mitigation

The dynamic arrays within the Allocation memory alloc struct in the `_computeSplitArrays` function must be initialized with the correct length `(nbOfFlows)` before the loop.

Corrected Code `(AlignerzVesting.sol)`:

```solidity

function _computeSplitArrays(
        Allocation storage allocation,
        uint256 percentage,
        uint256 nbOfFlows
    )
        internal
        view
        returns (
            Allocation memory alloc
        )
    {
        uint256[] memory baseAmounts = allocation.amounts;
        uint256[] memory baseVestings = allocation.vestingPeriods;
        uint256[] memory baseVestingStartTimes = allocation.vestingStartTimes;
        uint256[] memory baseClaimed = allocation.claimedSeconds;
        bool[] memory baseClaimedFlows = allocation.claimedFlows;
        alloc.assignedPoolId = allocation.assignedPoolId;
        alloc.token = allocation.token;

        // --- FIX: INITIALIZE THE MEMORY ARRAYS ---
        alloc.amounts = new uint256[](nbOfFlows);
        alloc.vestingPeriods = new uint256[](nbOfFlows);
        alloc.vestingStartTimes = new uint256[](nbOfFlows);
        alloc.claimedSeconds = new uint256[](nbOfFlows);
        alloc.claimedFlows = new bool[](nbOfFlows);
        // ------------------------------------------

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
  