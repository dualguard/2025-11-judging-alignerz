# [000239] Vesting Period Validation Logic Permits Impractical 1-Second Vesting Duration
  
  - **Description**

The vesting period validation in `AlignerzVesting::placeBid()` and `AlignerzVesting::updateBid()` allows users to select a vesting period of exactly 1 second. And while this passes validation checks, it creates a vesting schedule that defeats the purpse of vesting and may cause unexpected behaviour in claiming logic.

- **Recommended Mitigation**

Require vesting period to be a reasonable minimum
  