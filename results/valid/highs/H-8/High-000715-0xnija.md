# [000715] Dividend loops never advance, so setAmounts() /setDividends()  revert
  
  

## summary
getUnclaimedAmounts() never increments its loop index when a flow is already claimed or newly created, so the entire dividend setup path reverts before computing any values.


## Finding Description
setAmounts() calls _setAmounts(), which loops over every minted NFT and calls getUnclaimedAmounts(nftId). That function is implemented as:

[https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L140-L161](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L140-L161)
```solidity
function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
    ...
    uint256 len = vesting.allocationOf(nftId).amounts.length;
    for (uint i; i < len;) {
        if (claimedFlows[i]) continue;
        if (claimedSeconds[i] == 0) {
            amount += amounts[i];
            continue;
        }
        ...
        unchecked {
            ++i;
        }
    }
    ...
}
```


Every time the loop hits a continue , which happens immediately when claimedSeconds[i] == 0 or claimedFlows[i] is true, the unchecked { ++i; } at the bottom is skipped. Because every new flow starts with claimedSeconds == 0 and claimedFlows == false, the first iteration hits the continue and i never increases, so the function loops forever and setAmounts() can never finish. The dividend setup path (setAmounts() + setDividends()) therefore always reverts and dividends cannot be distributed at all.


** Attack Path **
Any call to setAmounts()  or setUpTheDividends() (owner-only) will call getUnclaimedAmounts() for each NFT. The faulty loop never terminates because every flow is in its initial state, so deployers cannot progress past the dividend setup stage, effectively locking the dividend contract. No external input is needed beyond owning valid NFTs and calling the owner functions.
- Preconditions: A project with at least one minted TVS NFT whose allocation has unclaimed flows ( claimedSeconds = 0 , claimedFlows = false ) — this is the normal initial state after claimNFT() .
- Steps:
  - Owner calls setAmounts() or setUpTheDividends() .
  - getTotalUnclaimedAmounts() iterates NFTs and invokes getUnclaimedAmounts() for existing tokens.
  - getUnclaimedAmounts() enters the loop at i = 0 , hits claimedSeconds[i] == 0 , executes continue , and never increments i , causing an infinite loop and revert.
Highest-impact scenario:

- Immediately after deployment and initial NFT claims, the owner attempts to configure dividends via setUpTheDividends() ; the first allocation encountered has an unclaimed flow in its initial state, causing the dividend setup to revert and blocking all distribution.


## Impact
High. The dividend contract can never compute or distribute dividends, stalling one of its core use cases.


## Likelihood Explanation
High. The bug is triggered immediately when the owner calls setAmounts()  after deployment, since the arrays it iterates over are full of unclaimed flows.

## poc 

Run

- Commands:
  - forge clean
  - forge build
  - forge test --mt test_DividendSetupInfiniteLoopReverts_Simple -vvv
import
```diff
+ import {A26ZDividendDistributor} from "../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";
```

```solidity
    function test_DividendSetupInfiniteLoopReverts() public {
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);
        vesting.launchBiddingProject(address(token), address(usdt), block.timestamp, block.timestamp + 1_000_000, "0x0", true);
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.01 ether, true);
        vesting.addUserToWhitelist(bidders[0], PROJECT_ID);
        vm.stopPrank();

        vm.startPrank(bidders[0]);
        usdt.approve(address(vesting), BIDDER_USD);
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 90 days);
        vm.stopPrank();

        bytes32[] memory leaves = new bytes32[](2);
        leaves[0] = getLeaf(bidders[0], BIDDER_USD, PROJECT_ID, 0);
        leaves[1] = getLeaf(makeAddr("dummy"), BIDDER_USD, PROJECT_ID, 0);
        CompleteMerkle m = new CompleteMerkle();
        bytes32 root = m.getRoot(leaves);
        bytes32[] memory proof = m.getProof(leaves, 0);

        bytes32[] memory poolRoots = new bytes32[](1);
        poolRoots[0] = root;

        vm.prank(projectCreator);
        vesting.finalizeBids(PROJECT_ID, bytes32(0), poolRoots, 60);

        vm.prank(bidders[0]);
        uint256 nftId = vesting.claimNFT(PROJECT_ID, 0, BIDDER_USD, proof);

        Aligners26 otherToken = new Aligners26("Other", "OTH");
        A26ZDividendDistributor distributor = new A26ZDividendDistributor(address(vesting), address(nft), address(usdt), block.timestamp, 10 days, address(otherToken));
        usdt.mint(address(distributor), 100_000_000);

        vm.expectRevert();
        distributor.setAmounts();
    }
}

contract DividendDistributorLoopPoCTest is Test {
    AlignerzVesting public vesting;
    Aligners26 public token;
    AlignerzNFT public nft;
    MockUSD public usdt;

    address public bidder;

    uint256 constant BIDDER_USD = 1_000 ether;
    uint256 constant PROJECT_ID = 0;

    function setUp() public {
        usdt = new MockUSD();
        token = new Aligners26("26Aligners", "A26Z");
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");
        vesting = new AlignerzVesting();
        vesting.initialize(address(nft));
        nft.addMinter(address(vesting));
        vesting.setTreasury(address(1));
        bidder = makeAddr("poc_bidder");
        vm.deal(bidder, 50 ether);
        usdt.mint(bidder, BIDDER_USD);
        token.approve(address(vesting), 3_000_000 ether);
    }

    function test_DividendSetupInfiniteLoopReverts_Simple() public {
        vesting.setVestingPeriodDivisor(1);
        vesting.launchBiddingProject(address(token), address(usdt), block.timestamp, block.timestamp + 1_000_000, "0x0", true);
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.01 ether, true);
        vesting.addUserToWhitelist(bidder, PROJECT_ID);

        vm.startPrank(bidder);
        usdt.approve(address(vesting), BIDDER_USD);
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 90 days);
        vm.stopPrank();

        bytes32[] memory leaves = new bytes32[](2);
        leaves[0] = keccak256(abi.encodePacked(bidder, BIDDER_USD, PROJECT_ID, uint256(0)));
        leaves[1] = keccak256(abi.encodePacked(makeAddr("dummy"), BIDDER_USD, PROJECT_ID, uint256(0)));
        CompleteMerkle m = new CompleteMerkle();
        bytes32 root = m.getRoot(leaves);
        bytes32[] memory proof = m.getProof(leaves, 0);

        bytes32[] memory poolRoots = new bytes32[](1);
        poolRoots[0] = root;
        vesting.finalizeBids(PROJECT_ID, bytes32(0), poolRoots, 60);

        vm.prank(bidder);
        vesting.claimNFT(PROJECT_ID, 0, BIDDER_USD, proof);
        vm.prank(address(vesting));
        nft.mint(bidder);

        Aligners26 otherToken = new Aligners26("Other", "OTH");
        A26ZDividendDistributor distributor = new A26ZDividendDistributor(address(vesting), address(nft), address(usdt), block.timestamp, 10 days, address(otherToken));
        usdt.mint(address(distributor), 100_000_000);

        vm.expectRevert();
        distributor.setAmounts();
    }
```
How It Works

- Sets up a minimal bidding project and mints a TVS NFT via claimNFT , which creates an allocation with claimedSeconds = 0 and claimedFlows = false .
- Mints one additional NFT to ensure nft.getTotalMinted() returns 2 , making index 1 reachable in getTotalUnclaimedAmounts() ; index 0 is skipped via safeOwnerOf .
- Instantiates A26ZDividendDistributor with a different token than the vesting allocation to avoid the early return 0 .
- Calls setAmounts() and expects a revert due to the infinite loop at the first continue branch inside getUnclaimedAmounts(1) .
- Assertion:
  - Uses vm.expectRevert() before setAmounts() to capture the revert (gas exhaustion/non-terminating loop).

## Recommendation
Ensure the loop increments i  before every continue. One fix is to replace the for loop with:


```solidity
for (uint i; i < len; i++) {
    if (claimedFlows[i]) continue;
    if (claimedSeconds[i] == 0) {
        amount += amounts[i];
        continue;
    }
    ...
}
```



  