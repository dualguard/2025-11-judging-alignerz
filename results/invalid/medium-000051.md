# [000051] Missing UUPSUpgradeable Initialization
  
  ### Summary

The `AlignerzVesting` contract inherits from `UUPSUpgradeable` but fails to call `__UUPSUpgradeable_init()` in its `initialize()` function. This prevents the UUPS proxy pattern from being properly initialized.

### Root Cause

```solidity
    /// @notice Initializes the vesting contract
    /// @param _nftContract Address of the NFT contract
    function initialize(address _nftContract) public initializer {//@audit missing uupsupgradable
        __Ownable_init(msg.sender);
        __FeesManager_init();
        __WhitelistManager_init();
        require(_nftContract != address(0), Zero_Address());
        nftContract = IAlignerzNFT(_nftContract);
        vestingPeriodDivisor = 2_592_000; // Set default vesting period multiples to 1 month (2592000 seconds)
    }
```

https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L18

https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L352

### Internal Pre-conditions


| Contract                           | Initializer Function               | Required? | Notes                                     |
| ---------------------------------- | ---------------------------------- | --------- | ----------------------------------------- |
| AccessControlUpgradeable           | `__AccessControl_init()`           | ✅ Yes     | Initializes role storage                  |
| AccessControlEnumerableUpgradeable | `__AccessControlEnumerable_init()` | ✅ Yes     | Adds enumerable roles                     |
| PausableUpgradeable                | `__Pausable_init()`                | ✅ Yes     | Enables pause/unpause                     |
| UUPSUpgradeable                    | `__UUPSUpgradeable_init()`         | ✅ Yes     | Required for upgrade authorization logic  |
| ERC721EnumerableUpgradeable        | `__ERC721Enumerable_init()`        | ✅ Yes     | Adds `totalSupply`, `tokenOfOwnerByIndex` |



### External Pre-conditions

_No response_

### Attack Path

The contract inherits from `UUPSUpgradeable` (line 18) but the `initialize()` function does not call `__UUPSUpgradeable_init()`. According to OpenZeppelin's UUPS pattern, all parent contract initializers must be called in the correct order during initialization.

### Impact

The inheritance chain initialization is incomplete, 

### PoC

https://solodit.cyfrin.io/issues/missing-initializer-calls-in-geniusvaultcores-initialization-function-halborn-shuttle-labs-genius-contracts-re-assessment-markdown

### Mitigation

```diff
function initialize(address _nftContract) public initializer {
    __Ownable_init(msg.sender);
+    __UUPSUpgradeable_init();  
    __FeesManager_init();
    __WhitelistManager_init();
    require(_nftContract != address(0), Zero_Address());
    nftContract = IAlignerzNFT(_nftContract);
    vestingPeriodDivisor = 2_592_000;
}
```
  