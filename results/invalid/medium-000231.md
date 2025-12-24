# [000231] Missing Access Control on `distributeRewardTVS` and `distributeStablecoinAllocation` leads to permanent DoS
  
  ### Summary

The `distributeRewardTVS` and `distributeStablecoinAllocation` functions are intended for post-deadline distribution of unclaimed rewards by the project owner. However, they are declared as `external` without an `onlyOwner` modifier, allowing any user to call them. A malicious actor can exploit this by providing a crafted input array that causes the function to revert mid-execution. This effectively creates a permanent Denial of Service (DoS), preventing the legitimate owner from ever successfully distributing the remaining unclaimed tokens to users.

### Root Cause

The root cause of this vulnerability is a missing access control modifier. The functions `distributeRewardTVS`(L525) and `distributeStablecoinAllocation`(L540) are intended to be privileged, but they lack the `onlyOwner` modifier. This oversight makes them publicly callable, exposing them to manipulation by external, non-privileged users.

https://github.com/dualguard/2025-11-alignerz-Ayomiposi233/blob/7b05b7b1bbb71e3e6957270e83365a936945ea5d/protocol/src/contracts/vesting/AlignerzVesting.sol#L525-L550


### Internal Pre-conditions

- The functions `distributeRewardTVS` and `distributeStablecoinAllocation` are callable externally by anoyone.

### External Pre-conditions

- Attacker watches mempool for owner's legitimate call.

### Attack Path

A malicious user can craft an input array to ensure the function always reverts after partially processing, preventing it from ever completing successfully.

1. __Setup__:

- A `RewardProject` is active, and its `claimDeadline` has passed.
- The `kolTVSAddresses` array contains at least two addresses of KOLs who have not claimed their rewards e.g `[KOL_A, KOL_B, KOL_C]`.
2. __The Attack__:

- An attacker calls `distributeRewardTVS(rewardProjectId, [KOL_A, KOL_A])` front-running the owner's call. The input array contains the address of a single legitimate claimant repeated.
3. __Execution Flow__:

- The `for` loop begins.
- __First Iteration (i=0)__: The function calls `_claimRewardTVS` for `KOL_A`. The claim is processed successfully: KOL_A's reward amount is set to `0`, an NFT is minted, and `KOL_A` is removed from the `kolTVSAddresses` array using a swap-and-pop mechanism. The state of the array is now effectively `[KOL_C, KOL_B]`.
- __Second Iteration (i=1)__: The function proceeds to the next element in the attacker's input array, which is again `KOL_A`.
- It calls `_claimRewardTVS` for `KOL_A` a second time.
- Inside `_claimRewardTVS`, the check `require(amount > 0, Caller_Has_No_TVS_Allocation())` is executed. Since KOL_A's reward was set to `0` in the first iteration, this check fails.
4. __Outcome (Denial of Service)__:

- The entire `distributeRewardTVS` transaction reverts.
- Because the transaction reverts, the successful claim and state changes from the first iteration are rolled back. `KOL_A`'s reward is restored, and the `kolTVSAddresses` array is returned to its original state.
- The attacker can repeat this attack indefinitely, ensuring that any attempt by the legitimate owner to call this function (even with a correct array) can be front-run and made to fail. This holds the distribution process hostage, preventing `KOL_B`, `KOL_C`, and all other remaining claimants from receiving their tokens via this mechanism.


### Impact

The impact is a permanent Denial of Service for the unclaimed reward distribution functionality. A malicious actor can prevent the protocol owner from fulfilling their duty to distribute tokens after the claim deadline, effectively trapping those funds and preventing them from reaching the intended recipients. This undermines the protocol's reliability and fairness.

### PoC

_No response_

### Mitigation

Enforce strict access control by adding the `onlyOwner` modifier to both `distributeRewardTVS` and `distributeStablecoinAllocation`. This ensures that only the trusted owner can initiate the distribution process with a correctly formed list of recipients.
  