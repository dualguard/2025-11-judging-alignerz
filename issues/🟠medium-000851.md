# [000851] Cross-Project Capital Recycling Enables Unfair Capital Efficiency Advantages
  
  ## Summary

The protocol has no restrictions preventing users from recycling the same capital across multiple projects through strategic timing of refunds and new bids, creating significant unfair advantages over committed long-term investors.

## Vulnerability Detail

Users can strategically recycle capital by exploiting timing differences between project lifecycles and immediate refund availability:

**Location**: [`AlignerzVesting.sol:708-735`](https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L708-L735) - placeBid

```solidity
function placeBid(uint256 projectId, uint256 amount, uint256 vestingPeriod) external {
    require(amount > 0, Zero_Value());
    // ❌ No check on capital source or holding period
    biddingProject.stablecoin.safeTransferFrom(msg.sender, address(this), amount);
    biddingProject.bids[msg.sender] = Bid({amount: amount, vestingPeriod: vestingPeriod});
}
```

**Location**: [`AlignerzVesting.sol:835-853`](https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L835-L853) - claimRefund

```solidity
function claimRefund(uint256 amount, uint256 projectId, bytes32[] calldata merkleProof) external {
    // ❌ No restrictions on immediate capital reuse after refund
    refundedBids[projectId][msg.sender] = true;
    biddingProject.stablecoin.safeTransfer(msg.sender, amount);
}
```

**Root Cause**: The protocol allows immediate capital reuse without any cooldown periods or restrictions on cross-project participation.

### Attack Mechanism - Capital Recycling Cycle

1. **Initial Capital**: User starts with limited capital (e.g., $100K USDT)
2. **Multi-Project Deployment**: Place same $100K across 5-10 projects with staggered timelines
3. **Strategic Refund Claiming**: As projects fail/succeed, immediately claim refunds
4. **Capital Redeployment**: Use refunded capital for new project bids
5. **Multiplicative Exposure**: Achieve 5-10x capital exposure compared to holdings
6. **Unfair Advantage**: Compete against users who lock capital per project

**Example Execution Timeline**:
```solidity
// User starts with $100K USDT actual capital

// Week 1: Projects A, B, C launch
placeBid(projectA, 100000, vestingPeriod); // Uses full capital
placeBid(projectB, 100000, vestingPeriod); // Same capital 
placeBid(projectC, 100000, vestingPeriod); // Same capital

// Week 2: Project A fails
claimRefund(100000, projectA, proofA); // Get capital back

// Week 3: Use same $100K for new projects
placeBid(projectD, 100000, vestingPeriod);
placeBid(projectE, 100000, vestingPeriod);

// Result: Participated in 5 projects with only $100K actual capital
// Honest users can only participate in 1 project at a time with $100K
```

**Code References**:
- [`AlignerzVesting.sol:860-891`](https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L860-891) - claimNFT function
- [`AlignerzVesting.sol:914-935`](https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L914-935) - withdrawPostDeadlineProfit function

## Impact

**Medium Severity**: Economic unfairness undermining project bidding integrity

- **Capital Multiplication**: Users achieve 5-10x effective capital compared to actual holdings
- **Unfair Competition**: Strategic recyclers compete against committed long-term investors
- **Market Distortion**: Project bidding dominated by capital efficiency rather than genuine investment commitment
- **Economic Inequity**: Sophisticated users gain systematic advantages over regular participants

**Capital Advantage Example**:
```solidity
// Strategic User (with capital recycling):
// $100K → simultaneous exposure to 7 projects → 7x capital efficiency

// Honest User (single project commitment):  
// $100K → exposure to 1 project → waits for outcome → moves to next
// Effective advantage: Strategic user gets 7x more opportunities
```

## Code Snippet

**Unrestricted bidding allowing capital recycling**:
```solidity
function placeBid(uint256 projectId, uint256 amount, uint256 vestingPeriod) external {
    require(amount > 0, Zero_Value());
    require(block.timestamp <= biddingProject.deadline, Deadline_Has_Passed());
    require(vestingPeriod <= biddingProject.vestingPeriod, Invalid_Vesting_Period());
    // ❌ No validation of capital source or concurrent project limits
    
    biddingProject.stablecoin.safeTransferFrom(msg.sender, address(this), amount);
    biddingProject.bids[msg.sender] = Bid({amount: amount, vestingPeriod: vestingPeriod});
}
```

**Immediate refund availability enabling recycling**:
```solidity
function claimRefund(uint256 amount, uint256 projectId, bytes32[] calldata merkleProof) external {
    // ... merkle proof validation ...
    // ❌ No cooldown or restrictions on capital reuse
    refundedBids[projectId][msg.sender] = true;
    biddingProject.stablecoin.safeTransfer(msg.sender, amount); // Immediate availability
}
```

  