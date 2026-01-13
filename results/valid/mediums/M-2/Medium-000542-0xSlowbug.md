# [000542] Incorrect bidFee limit enforces 0.1 USDC instead of documented 1 USDC, preventing admins from setting valid configuration
  
  ### Summary

The mismatch between the [README](https://github.com/dualguard/2025-11-alignerz/tree/main#q-are-there-any-limitations-on-values-set-by-admins-or-other-roles-in-protocols-you-integrate-with-including-restrictions-on-array-lengths) specification (maximum bidFee = **1 USDC = 1e6 units**) and the [contract’s validation](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L84-L92) (`newBidFee < 100001`) will cause **protocol misconfiguration** as admins will be unable to set a bid fee up to the documented maximum. The contract currently restricts the fee to ~0.1 USDC, contradicting the intended design.

### Root Cause

In the code:

```solidity
require(newBidFee < 100001, "Bid fee too high");
```

This enforces a maximum of:

```
100,001 units ≈ 0.100001 USDC (USDC has 6 decimals)
```

But the README clearly states:

> “bidFee and updateBidFee can have a maximum of **1 USDC/USDT**”

1 USDC = **1,000,000 units (1e6)**, not 100,001.

**Root cause:** The contract enforces the wrong maximum bid fee, limiting it to ~0.1 USDC instead of the intended 1 USDC.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. Admin attempts to set fee to a valid value like `300,000` (0.3 USDC).
2. Contract rejects it due to `require(newBidFee < 100001)`.
3. Admin cannot configure the protocol according to specifications.


### Impact

Admin **cannot set legitimate fee values** up to the documented maximum (1 USDC).
This causes:
Loss of fee for the protocol as they will only be able to charge maximum of 0.1 USDC instead of 1USDC as read me says

* **Protocol misconfiguration**
* **Unexpected operational limits**

### PoC

_No response_

### Mitigation

Replace placeholder limit with correct USDC max:

```solidity
require(newBidFee <= 1_000_000, "Bid fee too high"); // 1 USDC
```

Or define:

```solidity
uint256 constant MAX_BID_FEE = 1e6; // 1 USDC for 6-decimal tokens
```
  