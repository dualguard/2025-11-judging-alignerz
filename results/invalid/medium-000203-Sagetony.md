# [000203] Missing KOL Allocation Revocation Mechanism Causes Permanent Loss of Stablecoins and Tokens
  
  ### Summary

The choice to make KOL reward allocations immutable with no revocation or address replacement mechanism is a mistake as any misconfigured, compromised, or inaccessible KOL address will cause permanent loss of allocated stablecoins and tokens, with no surgical recovery path available to the protocol owner. 

### Root Cause

The choice to design setTVSAllocation() and setStablecoinAllocation() without corresponding revocation functions is a mistake as once a KOL address is set in kolTVSRewards[kol], kolStablecoinRewards[kol], and the address arrays, there is no mechanism to:

Remove or zero-out a specific KOL's allocation
Replace a KOL address with a corrected one
Redirect unclaimed allocations back to treasury
Skip a specific KOL during post-deadline distribution

The only existing recovery option (withdrawStuckTokens) is a nuclear option that affects all funds, not surgical removal of one problematic allocation.

### Internal Pre-conditions

Owner needs to call setTVSAllocation() or setStablecoinAllocation() with at least one KOL address that becomes problematic (wrong address, lost keys, sanctioned, etc.)

### External Pre-conditions

KOL wallet needs to become inaccessible (lost keys, compromised, sanctioned, or simply a typo in the original address)

### Attack Path

1. Owner calls setStablecoinAllocation() allocating 50,000 USDC across 10 KOLs, including 0xBadAddress (typo) allocated 10,000 USDC
2. 50,000 USDC is transferred into the vesting contract
3. 9 legitimate KOLs claim their allocations (40,000 USDC distributed)
4. 0xBadAddress cannot claim (nobody controls it)
5. Claim deadline passes
6. Owner has two bad options:

6a. Call distributeRemainingStablecoinAllocation() → sends 10,000 USDC to 0xBadAddress (lost forever)
6b.  Call withdrawStuckTokens() → but this would have affected ALL KOLs if done earlier, not surgical


7 10,000 USDC is permanently lost to an inaccessible address

### Impact

The protocol suffers permanent loss of stablecoins (USDC/USDT) and project tokens allocated to any KOL address that becomes inaccessible. In realistic scenarios involving 10-50 KOLs with allocations of $1,000-$50,000 each, a single misconfigured address can result in $10,000+ permanent loss. The protocol has no way to recover these funds or redirect them to active contributors, directly impacting treasury reserves and growth budget.

### PoC

_No response_

### Mitigation

Add a revoke or replace mechanism .
  