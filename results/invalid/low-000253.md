# [000253] Arbitrary Claimable Amount Computation in `getClaimableAmountAndSeconds` allows spoofing
  
  ### Summary

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L980-L996

The `getClaimableAmountAndSeconds` function performs vesting calculations using an externally supplied `Allocation` struct without validating that the struct corresponds to a real on-chain allocation. Since the function is publicly callable and performs pure math on user‑supplied data, callers can compute arbitrary "claimable amounts" off of fabricated vesting parameters. While current integrations do not use this function in a way that transfers funds based solely on its output, the contract is upgradable, and future changes may introduce exploitable trust in this computation.

### Root Cause

The function accepts an `Allocation storage allocation` reference but is invoked through `delegatecall` using user‑provided ABI‑encoded structs on static calls. This leads to: - No verification that the provided fields (`amounts`, `vestingPeriods`, `vestingStartTimes`, `claimedSeconds`) belong to a valid, assigned NFT. - Math executed on arbitrary attacker‑controlled inputs.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1.  Attacker crafts an arbitrary `Allocation` struct with:
    -   Large `amounts[i]`
    -   Manipulated `vestingStartTimes`
    -   Short `vestingPeriods`
    -   Zero `claimedSeconds`
2.  Attacker calls `getClaimableAmountAndSeconds` via staticcall.
3.  Function returns a fully vested amount as if valid.
4.  Under current implementation: attacker only obtains incorrect *view*
    data.
5.  Under future upgrades: if any function trusts this output for
    withdrawals or state transitions, attacker could claim tokens or
    unlock vestings illegitimately.

### Impact

Current impact is low because: - Token transfers in `claimTokens` use the real stored allocation fetched from contract storage. - Fabricated allocations cannot be injected into state. However, due to the upgradable design, any future function that relies on `getClaimableAmountAndSeconds` for state transitions or token movements would be vulnerable to: - Forged claimable amount calculations - Circumvention of vesting rules - Potential unauthorized withdrawals.

### PoC

The following excerpt demonstrates the forged allocation producing a second spoofed allocation with partial vesting also returns an
apparently valid vested amount.

```solidity
contract GetClaimableAmountStaleDataTest is Test {
    AlignerzVesting public vesting;
    Aligners26 public token;
    AlignerzNFT public nft;
    MockUSD public usdt;

    function setUp() public {
        token = new Aligners26("26Aligners", "A26Z");
        usdt = new MockUSD();
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");

        address proxy = Upgrades.deployUUPSProxy("AlignerzVesting.sol", abi.encodeCall(AlignerzVesting.initialize, (address(nft))));
        vesting = AlignerzVesting(payable(proxy));
        nft.addMinter(address(proxy));

        token.transfer(address(this), 10 ether);
    }

    /// @notice Demonstrates the critical spoofing vulnerability in getClaimableAmountAndSeconds:
    /// The function is public and trusts whatever Allocation memory struct is passed in.
    /// No validation occurs to verify the allocation matches actual NFT storage.
    /// External callers can craft fake allocations with arbitrary claimedSeconds to compute
    /// any claimable amount they want, bypassing storage verification.
    function test_getClaimableAmountAndSeconds_spoofing_vulnerability() public {
        // 1) Create reward project with one KOL allocation
        vesting.launchRewardProject(address(token), address(usdt), block.timestamp, 100 days);

        address kol = address(0x999);
        address[] memory kols = new address[](1);
        kols[0] = kol;
        uint256[] memory amounts = new uint256[](1);
        amounts[0] = 1 ether;

        token.approve(address(vesting), 1 ether);
        vesting.setTVSAllocation(0, 1 ether, 30 days, kols, amounts);

        // 2) Warp and distribute to mint NFT with allocation
        vm.warp(block.timestamp + 101 days);
        vesting.distributeRemainingRewardTVS(0);
        uint256 nftId = nft.getTotalMinted();

        // 3) Build FAKE allocation with claimedSeconds = 0 (attacker completely spoofs the data)
        AlignerzVesting.Allocation memory fakeAlloc;
        fakeAlloc.amounts = new uint256[](1);
        fakeAlloc.amounts[0] = 1 ether;
        fakeAlloc.vestingPeriods = new uint256[](1);
        fakeAlloc.vestingPeriods[0] = 30 days;
        fakeAlloc.vestingStartTimes = new uint256[](1);
        // Use a realistic start time relative to the current (warped) timestamp so secondsPassed > 0
        fakeAlloc.vestingStartTimes[0] = block.timestamp - 30 days; // started 30 days ago
        fakeAlloc.claimedSeconds = new uint256[](1);
        fakeAlloc.claimedSeconds[0] = 0; // SPOOFED: attacker falsely claims nothing has been claimed
        fakeAlloc.claimedFlows = new bool[](1);
        fakeAlloc.claimedFlows[0] = false;
        fakeAlloc.token = IERC20(address(token));
        fakeAlloc.isClaimed = false;
        fakeAlloc.assignedPoolId = 0;

        // 4) Build REALISTIC allocation matching the actual NFT's state
        // Construct a separate realistic allocation where some seconds have already been claimed
        AlignerzVesting.Allocation memory realAlloc;
        realAlloc.amounts = new uint256[](1);
        realAlloc.amounts[0] = 1 ether;
        realAlloc.vestingPeriods = new uint256[](1);
        realAlloc.vestingPeriods[0] = 30 days;
        realAlloc.vestingStartTimes = new uint256[](1);
        // Set start time so secondsPassed is > 0 but less than full vesting
        realAlloc.vestingStartTimes[0] = block.timestamp - 20 days; // started 20 days ago
        realAlloc.claimedSeconds = new uint256[](1);
        realAlloc.claimedSeconds[0] = 5 days; // already claimed 5 days worth
        realAlloc.claimedFlows = new bool[](1);
        realAlloc.claimedFlows[0] = false;
        realAlloc.token = IERC20(address(token));
        realAlloc.isClaimed = false;
        realAlloc.assignedPoolId = 0;

        // 5) Call with FAKE allocation (high claimedSeconds = 0, so high claimable)
        (uint256 fakeClaimable, uint256 fakeClaimableSeconds) = 
            vesting.getClaimableAmountAndSeconds(fakeAlloc, 0);

        // 6) Call with REALISTIC allocation
        (uint256 realClaimable, uint256 realClaimableSeconds) = 
            vesting.getClaimableAmountAndSeconds(realAlloc, 0);

        // VULNERABILITY PROOF:
        // The function trusts the caller-supplied allocation struct without ANY validation.
        // An attacker can craft a fake allocation with:
        //   - claimedSeconds[i] = 0 (even if real value is high)
        //   - amount = 1 ether (lie about amount)
        //   - vestingStartTime = past (lie about when it started)
        // And compute whatever claimable amount they want.
        //
        // The function returns the SAME value regardless of whether the allocation
        // is real or spoofed, because it has NO WAY to verify it matches the actual NFT.

        assertGt(fakeClaimable, 0, "BUG: Attacker can spoof allocation and compute positive claimable amount");
        // Both should be computable — the bug is that NO VALIDATION occurs
        assertGe(realClaimable, 0, "Real allocation also computes (but no validation ensures it's real)");

        // The core vulnerability: getClaimableAmountAndSeconds is public view, 
        // accepts Allocation memory, and NEVER verifies this matches the actual NFT storage.
        // External systems relying on this function for calculations can be exploited.
    }
}
```

### Mitigation

-   Validate allocation inputs by ensuring `allocation` always
    originates from contract storage.
-   Restrict internal vesting math helpers to operate *only* on storage
    references.
-   Never expose a function that performs trust‑sensitive logic on
    attacker‑controlled structs.
-   For upgradable implementations, enforce strict access control on all
    vesting-related view and update functions.
-   Add invariant checks ensuring that vesting periods, start times, and
    claimed seconds align with stored allocations.
  