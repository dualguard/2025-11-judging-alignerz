# [000762] Last minted NFT never receives dividends
  
  
### Summary

The incorrect NFT enumeration in the dividend distributor will cause the last minted TVS NFT to never receive dividends as the loop skips the highest token ID.

### Root Cause

In `A26ZDividendDistributor.sol:126-135` and `214-222` the contract uses a `for` loop starting from `i = 0` and going up to `i < len` over `nft.getTotalMinted()`, while the underlying `AlignerzNFT` (ERC721A) starts token IDs at 1, so token ID `len` is never visited and token ID 0 is always non-existent.

```solidity
    /// @notice USD value in 1e18 of all the unclaimed tokens of all the TVS
    function getTotalUnclaimedAmounts() public returns (uint256 _totalUnclaimedAmounts) {
        uint256 len = nft.getTotalMinted();
>>      for (uint i; i < len;) {
>>          (, bool isOwned) = safeOwnerOf(i); 
            if (isOwned) _totalUnclaimedAmounts += getUnclaimedAmounts(i);
            unchecked {
                ++i;
            }
        }
    }
```

```solidity
    /// @notice Internal logic that allows the owner to set the dividends for each TVS holder
    function _setDividends() internal {
        uint256 len = nft.getTotalMinted();
>>      for (uint i; i < len;) {
>>          (address owner, bool isOwned) = safeOwnerOf(i);
            if (isOwned) dividendsOf[owner].amount += (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts); 
            unchecked {
                ++i;
            }
        }
        emit dividendsSet();
    }
```

Because `AlignerzNFT` (via `ERC721A`) starts token IDs at 1 and mints them sequentially, valid token IDs are `1..len`, but the loops only visit `i = 0..len-1`; `safeOwnerOf(0)` always returns `(0,false)` and is ignored, while `safeOwnerOf(len)` is never called. This means the last minted NFT is never included in `totalUnclaimedAmounts` and never gets any share assigned in `_setDividends`, so its holder will not receive dividends even though they hold a valid TVS NFT.

### Internal Pre-conditions

1. There must be at least one TVS NFT minted by `AlignerzNFT` such that `nft.getTotalMinted() > 0` and the valid token IDs are `1..len`.  
2. The dividend distributor must rely on `getTotalUnclaimedAmounts` and `_setDividends` to compute and distribute dividends, which both use the incorrect loop `for (uint i; i < len;)` over `safeOwnerOf(i)`.

### External Pre-conditions

1. The system must have run the dividend setup flow (calling `setUpTheDividends` or `setAmounts` + `setDividends`) after at least one TVS NFT has been minted.  


### Attack Path

1. Multiple TVS NFTs are minted (IDs 1, 2, 3, ..., N) and represent claimable vesting positions that should receive dividends.  
2. The owner calls the dividend setup functions: first computing `totalUnclaimedAmounts` using `getTotalUnclaimedAmounts`, then calling `_setDividends` to assign `dividendsOf[owner].amount` based on `unclaimedAmountsIn[i]`.  
3. In both functions, the contract loops over `i = 0..len-1` where `len = nft.getTotalMinted()`, calling `safeOwnerOf(i)`.  
4. For `i = 0`, `safeOwnerOf(0)` calls `nft.extOwnerOf(0)`, which reverts because token ID 0 does not exist, and `safeOwnerOf` catches this and returns `(address(0), false)`, so token ID 0 is ignored.  
5. For `i = 1..len-1`, the existing NFTs (IDs 1..len-1) are correctly detected and their unclaimed amounts and dividend shares are included.  
6. For token ID `len` (the last minted NFT), `safeOwnerOf(len)` is never called because the loop stops at `i = len-1`, so this NFT’s unclaimed amount is not counted in `totalUnclaimedAmounts`, and `_setDividends` never assigns any `dividendsOf[owner]` for its holder, causing them to receive zero dividends even though they hold a valid TVS.

### Impact

The holder of the last minted TVS NFT never receives any dividends from the distributor because their token is skipped by both the total-unclaimed accounting and the dividends assignment, leading to a systematic underpayment where the final NFT holder’s fair share is lost from their perspective (and effectively redistributed among earlier holders or left undistributed, depending on how the protocol uses these values).

### PoC

Insert below under `test` folder and run with `forge test --mt test_dividends_enumeration_skips_last_token_id -vvv`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {A26ZDividendDistributor} from "../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";

contract DividendEnumerationBugTest is Test {
    AlignerzNFT public nft;
    A26ZDividendDistributor public distributor;

    address public owner;
    address public user1;
    address public user2;
    address public user3;

    function setUp() public {
        owner = address(this);
        user1 = makeAddr("user1");
        user2 = makeAddr("user2");
        user3 = makeAddr("user3");

        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");


        distributor = new A26ZDividendDistributor(
            address(1),        
            address(nft),     
            address(2),        
            block.timestamp,   
            30 days,           
            address(3)         
        );


        nft.mint(user1); // tokenId 1
        nft.mint(user2); // tokenId 2
        nft.mint(user3); // tokenId 3
    }

    function test_dividends_enumeration_skips_last_token_id() public {
        uint256 totalMinted = nft.getTotalMinted();
        assertEq(totalMinted, 3);


        (, bool exists1) = distributor.safeOwnerOf(1);
        (, bool exists2) = distributor.safeOwnerOf(2);
        (, bool exists3) = distributor.safeOwnerOf(3);
        assertTrue(exists1);
        assertTrue(exists2);
        assertTrue(exists3);


        uint256 len = totalMinted;
        uint256 seenExisting;
        for (uint256 i; i < len;) {
            (, bool exists) = distributor.safeOwnerOf(i);
            if (exists) {
                seenExisting++;
            }
            unchecked {
                ++i;
            }
        }

        // Only tokenIds 1 and 2 are ever seen; tokenId 3 (the last minted) is skipped
        assertEq(seenExisting, 2);
    }
}
```


Result:
```bash
Ran 1 test for test/DividendEnumerationBug.t.sol:DividendEnumerationBugTest
[PASS] test_dividends_enumeration_skips_last_token_id() (gas: 78947)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 856.25µs (152.88µs CPU time)

Ran 1 test suite in 84.16ms (856.25µs CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

### Mitigation

Update the loops in `getTotalUnclaimedAmounts` and `_setDividends` to iterate over the actual minted token ID range used by `AlignerzNFT` (ERC721A), starting at `_startTokenId()` (which is 1 in this code) and including `len`.


  