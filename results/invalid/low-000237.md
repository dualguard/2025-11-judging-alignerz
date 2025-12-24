# [000237] Missing Access Control on `AlignerzVesting::distributeStablecoinAllocation()` Function Allows Unauthorized Distribution Triggering
  
  - **Description**

The `::distributeStabelcoinALlocation()` function is missing the `onlyOwner` modifier, despite its NATSPEC stating *"Allows the owner to distribute the stablecoin tokens that have not been claimed yet to the KOLs"*. 

- **Impact**

Loss of administrative control over distribution timing and co-ordination.

- **Recommended Mitigation**

Add the `onlyOwner` modifier to the `::distributeStablecoinAllocation()` function
  