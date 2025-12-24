# [000058] USDT Blacklist in Distribution Loop Causes Permanent Fund Lock Due to Broken Batching Fallback
  
  ## Summary

The [`distributeRemainingStablecoinAllocation()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L515) function loops through all unclaimed KOL stablecoin allocations and transfers funds using `safeTransfer()`. If any single address in the list is blacklisted by USDT, the entire transaction reverts, permanently locking funds for all other KOLs in that reward project. A malicious actor can weaponize this by intentionally getting blacklisted or registering a known blacklisted address, causing a "poison pill" DoS that harms innocent users.

## Vulnerability Details

### The Distribution Function

```solidity
function distributeRemainingStablecoinAllocation(uint256 rewardProjectId) external onlyOwner {
    RewardProject storage rewardProject = rewardProjects[rewardProjectId];
    require(block.timestamp > rewardProject.claimDeadline, Deadline_Has_Not_Passed());

    uint256 len = rewardProject.kolStablecoinAddresses.length;
    for (uint256 i = len - 1; rewardProject.kolStablecoinAddresses.length > 0;) {
        address kol = rewardProject.kolStablecoinAddresses[i];
        uint256 amount = rewardProject.kolStablecoinRewards[kol];
        rewardProject.kolStablecoinRewards[kol] = 0;
        rewardProject.stablecoin.safeTransfer(kol, amount);  // ← Reverts if blacklisted
        rewardProject.kolStablecoinAddresses.pop();
        emit StablecoinAllocationsDistributed(rewardProjectId, kol, amount);
        unchecked {
            --i;
        }
    }
}
```

### USDT Blacklist Mechanism

USDT implements a blacklist that prevents transfers to/from blacklisted addresses:

```solidity
// USDT contract
function transfer(address _to, uint _value) public whenNotPaused {
    require(!isBlackListed[msg.sender]);  // Sender check
    require(!isBlackListed[_to]);         // Recipient check
    // ... transfer logic
}
```

When `safeTransfer()` is called with a blacklisted recipient, the transaction reverts with no way to skip that address.

### The Poison Pill Attack

Malicious KOL

1. Attacker registers as a KOL for a reward project
2. Admin sets stablecoin allocations including the attacker
3. Attacker intentionally gets their address blacklisted by USDT (or uses a pre-blacklisted address)
4. Attacker doesn't claim before deadline (waits for distribution)
5. Admin calls `distributeRemainingStablecoinAllocation()`
6. Loop reaches attacker's address → `safeTransfer()` reverts
7. All other KOLs in that project lose access to their funds permanently

### Why There's No Recovery

The function has no mechanism to:

- Skip blacklisted addresses
- Remove addresses from the list without paying them
- Recover funds after a revert

## Impact

One blacklisted address causes all other KOLs in that reward project to lose their allocations:

### Weaponizable by Attackers

A malicious actor can intentionally cause this:

1. Register as a KOL with a blacklisted address (or get blacklisted after registration)
2. Don't claim before deadline
3. Force the distribution to fail
4. Harm all other KOLs in that project

This is not just a "user mistake" - it's an attack vector that harms the protocol and other users.

## Severity

### High

- Direct loss of funds: Innocent users permanently lose their allocations
- Weaponizable: Attacker can intentionally cause harm to others
- No recovery mechanism: Funds are permanently locked
- Affects core functionality: Reward distribution is a primary protocol feature

### Escalation from Lightchaser Low-21

Lightchaser identified this as Low-21 "Sending tokens in a for loop" with a generic description about failed transfers. However, they did not identify the specific USDT blacklist poison pill attack vector where:

1. A malicious actor can intentionally weaponize this
2. One blacklisted address causes all other users to lose funds (not just the blacklisted user)
3. The impact is permanent fund loss, not just a failed transaction

This escalates the finding from Low (generic loop issue) to High (weaponizable attack causing permanent loss for innocent users).

## Note

Note on Recovery/Mitigation: Normally, the administrator could mitigate this DoS by using the batching function distributeStablecoinAllocation() to skip the blacklisted address. However, that function contains a critical Array Out-of-Bounds error that causes it to revert whenever a partial list is provided.

Consequently, both the primary distribution method AND the fallback batching method are non-functional, leaving the protocol with no mechanism to recover the funds or pay the innocent users. This guarantees permanent fund lock.

See separate findings: 
Broken Batching in distributeStablecoinAllocation() Causes Array Out-of-Bounds Revert 

## Recommended Mitigation

Skip Blacklisted Addresses: Implement a try-catch pattern to skip addresses that fail
  