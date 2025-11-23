# [000238] Missing Access Control on `AlignerzVesting::distributeRewardTVS()` Function Allows Unauthorized Distribution Triggering
  
  - **Description**

The `::distributeRewardTVS()` function is missing the `onlyOwner` modifier, depsite its NATSPEC stating *"Allows the owner to distribute the TVS that have not been claimed yet to the KOLs"*

- **Impact**

Loss of administrative control over distribution timing and co-ordination.

- **Recommended Mitigation**

Add the `onlyOwner` modifier to the `::distributeRewardTVS()` function.
  