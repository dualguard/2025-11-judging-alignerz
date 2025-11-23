# [000075] Deploying Mock USDC to Production Networks
  
  ### Summary

The deployment script a26zPolygon.s.sol deploys a MockUSD contract to Polygon mainnet, which is intended only for testing purposes and should not be used in production environments.  The MockUSD contract is explicitly designed as a mock token for development and testing, not as a production stablecoin.


### Root Cause

The deployment script includes MockUSD deployment at lines [24-26 of a26zPolygon.s.sol](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/script/a26zPolygon.s.sol#L24-L26). This mock contract is then transferred to the production owner address



### Internal Pre-conditions

1. Deployment script is executed with `--broadcast` flag to deploy to Polygon mainnet 
2. `MockUSD` contract gets deployed and ownership transferred to production address 

### External Pre-conditions

None

### Attack Path

1. Deployment script runs on Polygon mainnet with `--broadcast` flag
2. `MockUSD` is deployed as a production contract

### Impact

1. Deploying a mock token on mainnet undermines trust and signals improper production practices.
2. The mock token’s unrestricted minting makes it unsafe for any real economic activity.

### PoC

N/A

### Mitigation

Remove `MockUSD` deployment from mainnet scripts (Polygon, Arbitrum, MegaETH). Keep it only in testnet deployment scripts where mock tokens are appropriate:

```diff
/ For mainnet scripts (a26zPolygon.s.sol):  
// Remove these lines:  
// mockUSD = new MockUSD();  
// console.logString("MockUSD deployed at: ");  
// console.logAddress(address(mockUSD));  
// mockUSD.transferOwnership(0x64E6728D28D323Dd17b4232857B3A8e3AB9194d9);  
  
// For testnet scripts: Keep MockUSD deployment as-is
```
  