# [000049] Inconsistent use of input validation patterns.
  
  
## Summary

The `AlignerzNFT` contract mixes two different patterns for input validation and error handling:

1. **`require(condition, "message")`**
    
2. **`if (!condition) revert CustomError()`**
    

Example of inconsistency:

```solidity
if (!_exists(tokenId)) revert URIQueryForNonexistentToken();
```

While other parts of the contract use:

```solidity
require(_exists(tokenId), "ALIGNERZ: Token Id does not exist");
```

Using both patterns interchangeably increases code fragmentation and makes the error-handling style inconsistent across the contract. which affects readability, code style coherence, and maintainability.

Additionally, developers reviewing or extending this contract  struggle with mixed design decisions, especially where similar checks are implemented in different styles.


https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/nft/AlignerzNFT.sol#L96

https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/nft/AlignerzNFT.sol#L122

## **Impact**

- **Reduced readability:** Future auditors or contributors need to mentally switch between two validation styles.
    
- **Higher chance of future mistakes:** Mixed patterns often lead to uneven error handling or missed standardization.
    
- **Inconsistent gas usage:**
    
    - `require(string)` is more expensive.
        
    - Custom errors are cheaper.  
        Using both creates unpredictable gas patterns.
        

---

## **Recommendation**

Adopt a  single, consistent validation pattern  throughout the contract.

Two valid approaches exist:

#### Option A (preferred): Use custom `if (...) revert CustomError()` everywhere

- Cheaper gas.
    
- Modern Solidity style.
    
- Already used in `uri()` with `URIQueryForNonexistentToken()`.
    

#### Option B: Use `require()` everywhere

- Simpler to read for beginners.
    
- More verbose and more expensive in gas.
    

  