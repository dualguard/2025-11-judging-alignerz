# [000458] The vesting contract implementation initialization is not deactivated in the constructor
  
  ### Summary

`AlignerzVesting` inherits `UUPSUpgradeable` from OpenZeppelin, but the constructor lacks the call to `_disableInitializers`. This may make the proxy vulnerable by attacks from an attacker. The following quote is from OpenZeppelin official documentation: https://docs.openzeppelin.com/upgrades-plugins/writing-upgradeable#initializing-the-implementation-contract

> Do not leave an implementation contract uninitialized. An uninitialized implementation contract can be taken over by an attacker, which may impact the proxy

### Root Cause

Missing constructor with `_disableInitializers`

### Impact

The proxy may be impacted by an attacker taking over the implementation.

### Mitigation

Add a constructor with `_disableInitializers`

```diff
+   /// @custom:oz-upgrades-unsafe-allow constructor
+   constructor() {
+       _disableInitializers();
+   }
```
  