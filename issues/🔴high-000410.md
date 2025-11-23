# [000410] TVS Distribution Denial of Service via Malicious ERC721 Receiver Rejection
  
  ### Summary

The TVS (Token Vesting Schedule) distribution functions `distributeRewardTVS` and `distributeRemainingRewardTVS` mint NFTs to KOL addresses using `nftContract.mint()`, which internally calls `_safeMint`. If a KOL address is a smart contract that deliberately reverts in its `onERC721Received` callback (returning false or reverting), the entire distribution batch fails, preventing all subsequent legitimate KOLs from receiving their allocations.

### Root Cause

The `_claimRewardTVS` internal function calls `nftContract.mint(kol)`, which uses ERC721A's `_safeMint` that enforces the ERC721 receiver check via `_checkContractOnERC721Received`. When this check fails (contract returns wrong selector or reverts), the entire mint transaction reverts without try/catch handling, propagating the failure up through the distribution loop.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

Relevant code from TVS distribution functions:
https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L554-L577

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L525-L535

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L601-L623

The mint flow in `AlignerzNFT.sol`:
```solidity
function mint(address to) external whenNotPaused onlyMinter returns (uint256) {
    require(to != address(0), "ALIGNERZ: Address cannot be 0");
    uint256 startToken = _currentIndex;
    _safeMint(to, 1); // Calls ERC721A's _safeMint
    emit Minted(to, startToken);
    return startToken;
}
```

From `ERC721A.sol`:
```solidity
function _safeMint(address to, uint256 quantity, bytes memory _data) internal {
    _mint(to, quantity, _data, true);
}

function _mint(address to, uint256 quantity, bytes memory _data, bool safe) internal {
    // ... minting logic ...
    if (safe && isContract(to)) {
        if (!_checkContractOnERC721Received(from, to, tokenId, _data)) {
            revert TransferToNonERC721ReceiverImplementer();
        }
    }
}

function _checkContractOnERC721Received(...) private returns (bool) {
    try IERC721Receiver(to).onERC721Received(...) returns (bytes4 retval) {
        return retval == IERC721Receiver.onERC721Received.selector;
    } catch (bytes memory reason) {
        // ... revert handling
    }
}
```

### Attack scenario:
1. Malicious KOL deploys a contract address with the following code:
```solidity
contract MaliciousKOL {
    function onERC721Received(...) external pure returns (bytes4) {
        revert("DoS attack"); // or return wrong selector
    }
}
```

2. Malicious KOL registers `MaliciousKOL` contract address during TVS allocation setup
3. After claim deadline, anyone calls `distributeRewardTVS` or owner calls `distributeRemainingRewardTVS`
4. Distribution loop processes legitimate KOLs successfully
5. When loop reaches `MaliciousKOL` address:
   - `nftContract.mint(MaliciousKOL)` is called
   - `_safeMint` checks `isContract(MaliciousKOL)` → true
   - `_checkContractOnERC721Received` calls `MaliciousKOL.onERC721Received`
   - Malicious contract reverts or returns wrong selector
   - `_safeMint` reverts with `TransferToNonERC721ReceiverImplementer`
6. **Entire distribution transaction reverts** → all subsequent KOLs blocked





### Impact

## Impact
- **Complete distribution halt**: Single malicious contract KOL blocks entire TVS distribution
- **Griefing attack surface**: Attacker can deliberately register contract addresses to DoS distribution
- **Funds locked**: Legitimate KOLs after the malicious entry cannot receive NFT certificates or claim tokens
- **No partial progress**: Loop-wide revert means no state changes saved, retry attempts fail at same point

### PoC

_No response_

### Mitigation

### Use try/catch to skip failing mints (recommended)
Wrap the mint call in error handling to continue on failure:

```diff
 function _claimRewardTVS(uint256 rewardProjectId, address kol) internal {
     RewardProject storage rewardProject = rewardProjects[rewardProjectId];
     uint256 amount = rewardProject.kolTVSRewards[kol];
     require(amount > 0, Caller_Has_No_TVS_Allocation());
     rewardProject.kolTVSRewards[kol] = 0;
-    uint256 nftId = nftContract.mint(kol);
+    uint256 nftId;
+    try nftContract.mint(kol) returns (uint256 _nftId) {
+        nftId = _nftId;
+    } catch {
+        // Mint failed (non-ERC721 receiver), skip this KOL
+        emit RewardTVSDistributed(rewardProjectId, kol, 0, 0, 0); // Indicate skip
+        // Remove from pending list
+        uint256 index = rewardProject.kolTVSIndexOf[kol];
+        uint256 arrayLength = rewardProject.kolTVSAddresses.length;
+        rewardProject.kolTVSIndexOf[kol] = arrayLength - 1;
+        address lastIndexAddress = rewardProject.kolTVSAddresses[arrayLength - 1];
+        rewardProject.kolTVSIndexOf[lastIndexAddress] = index;
+        rewardProject.kolTVSAddresses[index] = rewardProject.kolTVSAddresses[arrayLength - 1];
+        rewardProject.kolTVSAddresses.pop();
+        return; // Exit early
+    }
     rewardProject.allocations[nftId].amounts.push(amount);
     // ... rest of allocation setup
 }
  