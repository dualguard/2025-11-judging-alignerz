# [000293] NFT ownership does not guarantee correct vesting allocation association
  
  
**Summary:** The `claimTokens()` function assumes that `nftId` always belongs to the correct `projectId`, but the contract never validates this relationship. As a result, a caller can submit a valid `nftId` they own together with an unrelated or mismatched `projectId`, allowing unauthorized claims from allocations that were never intended for that NFT.


**Root Cause** The function retrieves allocation data using:
```javascript
(biddingProjects[projectId].allocations[nftId])
```
or
``` javascript
(rewardProjects[projectId].allocations[nftId])
```
without checking that:
1. The NFT actually belongs to that specific project.
2. The nftId exists in the project’s allocation mapping.
3. The project/NFT pairing was ever configured together.

This missing validation breaks the assumption that NFT ownership equals entitlement to the associated vesting schedule.

**Vulnerability Details:** There's no check in the `claimTokens` function that the `nftId` acually belongs to the `projectId` an attacker can just pass a valid `nftId` they own with any `projectId`
```javascript
 function claimTokens(uint256 projectId, uint256 nftId) external {
        address nftOwner = nftContract.extOwnerOf(nftId);
        require(msg.sender == nftOwner, Caller_Should_Own_The_NFT());
        bool isBiddingProject = NFTBelongsToBiddingProject[nftId];
@->    //@audit Allocation is pulled directly from projectId → nftId without validation
        (Allocation storage allocation, IERC20 token) = isBiddingProject ? 
        (biddingProjects[projectId].allocations[nftId], biddingProjects[projectId].token) : 
        (rewardProjects[projectId].allocations[nftId], rewardProjects[projectId].token);
        uint256 nbOfFlows = allocation.vestingPeriods.length;
       
       ////Rest of the code...

```
**Impact:**
A malicious NFT holder can:
1) Claim tokens from a different project’s allocation
If another project uses the same nftId index (or has uninitialized entries), the attacker may drain tokens from that project.
2. Bypass project-level vesting rules
Because the NFT-project linkage isn’t enforced, attackers can claim from vesting schedules they were never assigned.
3. Trigger unintended burns
The NFT may be burned when all flows in the mismatched project are marked claimed.


**Tool Used:** Manual Review

**POC:**

**Recommended Mitigation:** Add strict validation ensuring that an NFT belongs to the project before accessing its allocation:
```javascript
require(
    allocation.exists,
    "NFT does not belong to the specified project"
);
```
Additionally:
1. Validate that the project type (bidding vs reward) matches what the NFT is assigned to.
2. Reject calls where the allocation is uninitialized.
  