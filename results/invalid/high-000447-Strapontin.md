# [000447] NFTs can be split and merged with other projects
  
  ### Summary

The vesting contract allows users to earn NFTs based on bids they make on projects. If users strongly believe in a project, we can expect to have more users bid higher for it, and the value are expected to be sent to the project as funds to build their structure.

The vesting contract also allows users to split and merge their NFTs in order to update their allocation and potentially send/sells parts of it to other users.

The merging function allows users to merge NFTs from project 1 to NFTs from another project, which is an issue. If a user can do this, they will invest in the least interested project, ensuring best results (higher amount for lower vesting period), and merge them into other more interested project that distributes high amount of dividends. It will make these NFT holders able to receive the project's dividends, even though they didn't participate in the project's funding. 

They can even do this repeatedly on other projects, before a dividend is distributed, since the amount to distribute is only linked to the NFT during dividend distribution. That means they don't need to hold the funds in one NFT to abuse this behavior, but only to hold it during dividend distribution, before passing it into the next project.

### Root Cause

Users can merge from different projects.

### Attack Path

1. Alice has an 2 NFTs, one for project X, another for project Y.
2. Before dividends are distributed to project X, she can split/merge her NFT Y's amount to set them all into NFT X.
3. Once project X dividends are distributed, and before project Y dividends' distribution, she does the same to hold a bigger amount in NFT Y.
4. Since she inflates her real invested amounts, she will earn a bigger part of dividends than she's supposed to, effectivelly stealing from other NFT holders.

### Impact

Steal of funds.

### Mitigation

Do not allow merging from multiple projects:

```diff
-   function mergeTVS(uint256 projectId, uint256 mergedNftId, uint256[] calldata projectIds, uint256[] calldata nftIds)
+   function mergeTVS(uint256 projectId, uint256 mergedNftId, uint256[] calldata nftIds)
        external
        returns (uint256)
    {
        // [...]

        for (uint256 i; i < nbOfNFTs; i++) {
-           feeAmount += _merge(mergedTVS, projectIds[i], nftIds[i], token);
+           feeAmount += _merge(mergedTVS, projectId, nftIds[i], token);
        }

        // [...]
```
  