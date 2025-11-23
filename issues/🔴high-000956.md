# [000956] Call to setUpDividends() fails if someone split his TVS and then merge before  setUpDividends() getting called
  
  ### Summary

The A26ZDividendDistributor::setUpTheDividends() call fails due to flaw in getTotalUnclaimedAmounts(). If someone splits his TVS & then merge again, and after that if owner calls the setUpDividends() then the call will be reverted by evm. 

### Root Cause

The [for loop](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L129) expects the nftId starts with 0.
```solidity
uint256 len = nft.getTotalMinted();
        for (uint i; i < len;) {    // i = 0 for the 1st iteration
            (, bool isOwned) = safeOwnerOf(i);
            if (isOwned) _totalUnclaimedAmounts += getUnclaimedAmounts(i);
            unchecked {
                ++i;
            }
        }
```
But when there is more than 1 nft was minted but few of them were burned for any reason then this logic reverts. 

### Internal Pre-conditions

None.

### External Pre-conditions

None.

### Attack Path

None. 

### Impact

Dividend can't be set up. 

### PoC

Before the running the test successfully we need to update 2 functions, replace the [FeesManager::calculateFeeAndNewAmountForOneTVS()](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169) with this:
```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        newAmounts = new uint[](amounts.length);  // @audit-info this line added
        for (uint256 i; i < length; i++) {   // @audit-info i++ added
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);            
            newAmounts[i] = amounts[i] - feeAmount;        
        }
    }
```
and replace the [AlignerzVesting::_computeSplitArrays()](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1113) with this:
```solidity
    function _computeSplitArrays(
        Allocation storage allocation,
        uint256 percentage,
        uint256 nbOfFlows
    )
        internal
        view
        returns (
            Allocation memory alloc
        )
    {
        uint len = allocation.amounts.length; // added
        alloc.amounts = new uint256[](len); // added
        alloc.vestingPeriods = new uint256[](len); // added
        alloc.vestingStartTimes = new uint256[](len); // added
        alloc.claimedSeconds = new uint256[](len); // added
        alloc.claimedFlows = new bool[](len); // added

        uint256[] memory baseAmounts = allocation.amounts;
        uint256[] memory baseVestings = allocation.vestingPeriods;
        uint256[] memory baseVestingStartTimes = allocation.vestingStartTimes;
        uint256[] memory baseClaimed = allocation.claimedSeconds;
        bool[] memory baseClaimedFlows = allocation.claimedFlows;
        alloc.assignedPoolId = allocation.assignedPoolId;
        alloc.token = allocation.token;
        for (uint256 j; j < nbOfFlows;) {
            alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;
            alloc.vestingPeriods[j] = baseVestings[j];
            alloc.vestingStartTimes[j] = baseVestingStartTimes[j];
            alloc.claimedSeconds[j] = baseClaimed[j];
            alloc.claimedFlows[j] = baseClaimedFlows[j];
            unchecked {
                ++j;
            }
        }
    }
```
The reason behind this update is these 2 functions has bugs, and these are the fixed version, if we don't fix the functions I can't show the working PoC. I have submitted reports related to these issues, you can find those with these titles: Uninitialized array results in DoS, Infinite loop will result in out of gas error i.e DoS and Flawed logic in FeesManager::calculateFeeAndNewAmountForOneTVS() will result DoS. For now just update those functions.
Now create a test file under ./test directory and paste this code:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "./AlignerzVestingProtocolTest.t.sol";
import "../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";

contract RewardProjectTest is AlignerzVestingProtocolTest {
        A26ZDividendDistributor dividendDistributor;
        function test_dividendSetupFails() public {
        // Deploying the dividendDistributor
        vm.prank(projectCreator);
        dividendDistributor = new A26ZDividendDistributor(
            address(vesting),
            address(nft),
            address(usdt),
            block.timestamp,
            60 days,
            address(token)
        );
        // Providing 1000e18 stablecoin to dividend distributor contract.
        // This amount is now reserved for TVS holders. 
        usdt.mint(address(dividendDistributor),1000e18);
        assertEq(usdt.balanceOf(address(dividendDistributor)),1000e18);

        uint currentTime = block.timestamp;
        vm.prank(projectCreator);
        // Launching reward project
        vesting.launchRewardProject(
            address(token),
            address(usdt),
            currentTime,
            30 days // claimWindow
        );

        // Setting TVS allocation
        address[] memory kolTVS = new address[](1);
        uint[] memory TVSamounts = new uint[](1);

        kolTVS[0] = makeAddr("KOL-1");

        TVSamounts[0] = 1000e18;

        vm.prank(projectCreator);
        vesting.setTVSAllocation(0, 1000e18, 60 days, kolTVS, TVSamounts);

        vm.warp(currentTime + 25 days);

        vm.prank(makeAddr("KOL-1"));
        vesting.claimRewardTVS(0);

        assertEq(nft.extOwnerOf(1), makeAddr("KOL-1"));

        uint[] memory percentages = new uint[](5);
        percentages[0] = 2000;
        percentages[1] = 2000;
        percentages[2] = 2000;
        percentages[3] = 2000;
        percentages[4] = 2000;
        assertEq(vesting.splitFeeRate(), 0, "Split fee error");
       
        vm.prank(makeAddr("KOL-1"));
        vesting.splitTVS(0, percentages, 1);

        // Lets merge now
        uint[] memory projectIds_ = new uint[](4);
        projectIds_[0] = 0;
        projectIds_[1] = 0;
        projectIds_[2] = 0;
        projectIds_[3] = 0;

        uint[] memory nftIds_ = new uint[](4);
        nftIds_[0] = 2;
        nftIds_[1] = 3;
        nftIds_[2] = 4;
        nftIds_[3] = 5;

        vm.prank(makeAddr("KOL-1"));
        uint newNftId = vesting.mergeTVS(0, 1, projectIds_, nftIds_);
        
        //Setting up dividends
        vm.prank(projectCreator);
        dividendDistributor.setUpTheDividends();

}
```
Run this test:
```solidity
forge clean && forge build && forge test --mt test_mergeTVS -vvvv
```
The output is this:
```solidity
├─ [0] VM::prank(projectCreator: [0x3e221db247A1Dd4A0724cf57b6b05304A2DcA513])
    │   └─ ← [Return]
    ├─ [43844] A26ZDividendDistributor::setUpTheDividends()
    │   ├─ [1284] MockUSD::balanceOf(A26ZDividendDistributor: [0x1422476607EB46CFb4925de78e2915E2A15701e9]) [staticcall]
    │   │   └─ ← [Return] 1000000000000000000000 [1e21]
    │   ├─ [1018] AlignerzNFT::getTotalMinted() [staticcall]
    │   │   └─ ← [Return] 5
    │   ├─ [1680] AlignerzNFT::extOwnerOf(0) [staticcall]
    │   │   └─ ← [Revert] OwnerQueryForNonexistentToken()
    │   ├─ emit iteration(: 1051300839 [1.051e9])
    │   ├─ [4318] AlignerzNFT::extOwnerOf(1) [staticcall]
    │   │   └─ ← [Return] KOL-1: [0xe1EE2F45A6eA5e850032a987Cd1Ef8a57BFbce30]
    │   ├─ [3714] ERC1967Proxy::fallback(1) [staticcall]
    │   │   ├─ [2977] AlignerzVesting::allocationOf(1) [delegatecall]
    │   │   │   └─ ← [Return] false, Aligners26: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], 0
    │   │   └─ ← [Return] false, Aligners26: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], 0
    │   └─ ← [Revert] EvmError: Revert
    └─ ← [Revert] EvmError: Revert
```

### Mitigation

I tried a lot but could not figure out why exactly evm reverting this. Waiting for expert's opinion on this. 
  