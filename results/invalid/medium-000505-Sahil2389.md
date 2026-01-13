# [000505] Whitelist Bypass Allows Non-Whitelisted Users to Gain Vesting Allocations Through NFT Transfer and Split Operations
  
  ### Summary

The whitelist mechanism only enforces access control during the `placeBid()` function. After a whitelisted user receives an NFT representing their vesting allocation, they can freely transfer or split the NFT to non-whitelisted addresses, completely bypassing the whitelist restrictions. This undermines the project's access control and allows unauthorized users to participate in whitelisted-only projects.

### Root Cause

See Summary & ...
In `AlignerzVesting.sol` contract Incomplete Access Control logic leads to Allows Non-Whitelisted Users to Gain Vesting Allocations Through NFT Transfer and Split Operations of white lisiting enabled project's nft's.

1. Whitelist check only exists in `placeBid()`:
```solidity
function placeBid(uint256 projectId, uint256 amount, uint256 vestingPeriod) external {
    if (isWhitelistEnabled[projectId]) {
        require(isWhitelisted[msg.sender][projectId], User_Is_Not_whitelisted());
    }
    // ... rest of function
}
```

2. No whitelist tracking on NFTs -> NFTs don't record which project they belong to or if that project has whitelist enabled or not(No tracking of original project whitelist status on NFT metadata):
```solidity
In `AlignerzVesting.sol`
mapping(uint256 => bool) public NFTBelongsToBiddingProject; // Only tracks bidding vs reward

// Missing: mapping to track which projectId the NFT belongs to  <<--------
//Missing: NFT belongs to Whitelist Enabled projects or not      <<--------
In `WhitelistManager.sol`
/// @notice Mapping to track projects' whitelisting status (project ID => isWhitelistEnabled)
mapping(uint256 => bool) public isWhitelistEnablet;

//Missing: No tracking of nft belongs to whitelisted project or not  <<--------
 /// @notice Represents an allocation
    /// @dev Tracks the allocation status and vesting progress
    struct Allocation {
        uint256[] amounts; // Amount of tokens committed for this allocation for all flows
        uint256[] vestingPeriods; // Chosen vesting duration in seconds for all flows
        uint256[] vestingStartTimes; // start time of the vesting for all flows
        uint256[] claimedSeconds; // Number of seconds already claimed for all flows
        bool[] claimedFlows; // Whether flow is claimed
        bool isClaimed; // Whether TVS is fully claimed
        IERC20 token; // The TVS token
        uint256 assignedPoolId; // Relevant for bidding projects: Id of the Pool (poolId=0; pool#1 / poolId=1; pool#2 /...) - for reward projects it will be 0 as default
    }
```
3. No whitelist checks in critical functions:
   - Standard ERC721 `transferFrom()` (In `AlignerzNFT.sol` )- No whitelist verification on transfers based on NFT's (because no tracking that nft belongs to Whitelist Enabled projects or not).
   - `claimTokens()` - No whitelist verification when claiming vested tokens.

Code Snippet-
https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/nft/ERC721A.sol#L297
https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L941-L975


### Internal Pre-conditions

1. Project launches with `whitelistStatus = true`.
2. `isWhitelistEnabled[projectId] = true` is set.
3. Attacker address is whitelisted for the project.


### External Pre-conditions

1. Malicious whitelisted user has accomplice non-whitelisted addresses.
2. Bidding project has closed and allocations are finalized.


### Attack Path


Step 1: Whitelisted User Places Large Bid with Long Vesting Period
```solidity
// Attacker (whitelisted) places bid
placeBid(projectId, 1000000, vestingPeriod); // Large amount, long vesting
```

Step 2: Project Closes and NFT is Claimed
```solidity
// After project closes, attacker claims NFT
claimNFT(projectId, poolId, 1000000, merkleProof);
// Receives nftId = 123
```

Step 3: Split NFT into Multiple Parts
```solidity
// Split into 10 parts (10% each)
uint256[] memory percentages = [1000, 1000, 1000, 1000, 1000, 1000, 1000, 1000, 1000, 1000];
splitTVS(projectId, percentages, 123);
// Creates 9 new NFTs: 124, 125, 126, 127, 128, 129, 130, 131, 132
```

Step 4: Transfer NFTs to Non-Whitelisted Addresses
```solidity
// Transfer via standard ERC721 function (no whitelist check)
nftContract.transferFrom(attacker, nonWhitelistedUser1, 124);
nftContract.transferFrom(attacker, nonWhitelistedUser2, 125);
// ... transfer remaining NFTs to other non-whitelisted users
```

Step 5: Non-Whitelisted Users Claim Tokens
```solidity
// Non-whitelisted users can now claim vested tokens
// Called by nonWhitelistedUser1
claimTokens(projectId, 124); // No whitelist check!
```


### Impact

Non-whitelisted users gain unauthorized access to whitelisted-only projects. Whitlisting system bypass. Defeats purpose of selective participation (e.g., early supporters, strategic partners). 


### PoC

N/A

### Mitigation

N/A
  