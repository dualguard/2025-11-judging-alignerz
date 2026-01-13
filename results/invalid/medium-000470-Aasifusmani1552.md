# [000470] The `calculateFeeAmount` function in `FeesManager` contract rounds down, effectively reducing the protocol fee and ultimately leading to loss of funds
  
  ### Summary

The `calculateFeeAmount` function in `FeesManager` contract calculates fees like this:-

```solidity
function calculateFeeAmount(uint256 feeRate, uint256 amount) public pure returns (uint256 feeAmount) {
@>        feeAmount = amount * feeRate / BASIS_POINT;
    }
```
Basically it uses integer division to calculate the fee and doesn't use any rounding protection libraries/functions like `mulDiv` from Openzeppelin. Now because integer division always rounds down, the fee will also round down effectively truncating the fractional part. This leads to **systematic undercharging**, where the protocol receives less fee than intended, especially when amounts are small because of frequent split/merge calls.

In some instances when the fee is low or the amount is low, the fee can even become 0. Because fee logic affects protocol revenue integrity and it is always advised to the  protocols to always round up while charging the fees, this can be severe and can be exploited by the attackers. 

### Root Cause

The root cause here is that the fee calculation uses floor division:
```solidity
feeAmount = amount * feeRate / BASIS_POINT;
```
This truncates fractional fees instead of rounding up as always happens when protocols charge fees. Because fees are the main source of protocol income, fee calculation always rounds up to favour protocol and to prevent rounding exploits. But here it is rounding down which is the main root cause.

### Internal Pre-conditions

N/A

### External Pre-conditions

N/A

### Attack Path

This is an issue in the core fee calculation formula, not an **Attack**. Though it can become an attack if a TVS in split and merged many times creating large number of flows with small amounts. In this condition, another split/merge can round down to 0.

### Impact

1. **Loss of Protocol Revenue** – Any fractional fee is rounded down to zero or a lower integer, reducing intended earnings.
2. **Exploitability** – Attackers can intentionally split amounts into smaller chunks to minimize fees across many operations.
3. **Inconsistent Effective Fee Rates** – Users with small amounts pay disproportionately lower fees, allowing manipulation.
4. **Cascading Rounding Errors** – Because merge/split calls reuse this function, rounding losses accumulate, magnifying the financial deviation.

**Severity - MEDIUM**

### PoC

NA

### Mitigation

User ceil division in fee calculation instead of integer division. Like :-
```diff
function calculateFeeAmount(uint256 feeRate, uint256 amount) 
    public 
    pure 
    returns (uint256 feeAmount) 
{
-    feeAmount = amount * feeRate / BASIS_POINT;
+    feeAmount = (amount * feeRate + BASIS_POINT - 1) / BASIS_POINT;
}

```
  