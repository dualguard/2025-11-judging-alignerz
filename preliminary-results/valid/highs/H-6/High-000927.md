# [000927] [M-1] Gas Intensive Loops Without Bounds (DoS in Setup Functions)
  
  ### Summary

The getTotalUnclaimedAmounts() and _setDividends() functions iterate over all minted NFTs without any upper limits. Each iteration invokes external contracts and loops over allocation arrays, so as the number of NFTs grows, the total gas required can exceed block limits. This can prevent the owner from distributing dividends, effectively causing a self-imposed DoS at scale.



### Root Cause

The functions loop over all minted NFTs (`nft.getTotalMinted()`) and their allocations without any bounds or batching mechanism. This unbounded iteration, combined with external calls (`vesting.allocationOf()` and `safeOwnerOf()`), causes gas consumption to grow linearly with the number of NFTs, eventually exceeding block gas limits as the launchpad scales.

Code Link  : https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L127-L136
### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. Owner calls setUpTheDividends() (or any function relying on getTotalUnclaimedAmounts()).

2. Function iterates over all minted NFTs and performs external calls.

3. As NFT count grows, gas consumption exceeds block limit.

4. Transaction fails, preventing dividend setup.

### Impact

 If the number of minted NFTs grows significantly, the owner may be unable to distribute dividends because functions like getTotalUnclaimedAmounts() could exceed block gas limits. This creates a self-DoS scenario where the protocol’s dividend distribution is stalled. While users cannot directly exploit this, it can temporarily block the protocol’s operations and reward mechanisms, especially on chains with tighter gas constraints like Polygon or Arbitrum.

### PoC

1. Navigate to test/AlignerzVestingProtocolTest.t.sol.

2.  Run the test using:

```solidity
 forge test --match-test test_GasDoS --match-path test/AlignerzVestingProtocolTest.t.sol -vvv
```

```solidity

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {ERC721} from "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "forge-std/Test.sol";

interface IAlignerzVesting {
    struct Allocation {
        IERC20 token;
        uint256[] amounts;
        uint256[] claimedSeconds;
        uint256[] vestingPeriods;
        bool[] claimedFlows;
    }
    function allocationOf(uint256 nftId) external view returns (Allocation memory);
}

interface IAlignerzNFT {
    function extOwnerOf(uint256 tokenId) external view returns (address);
    function getTotalMinted() external view returns (uint256);
}

contract A26ZDividendDistributorMock {
    IAlignerzVesting public vesting;
    IAlignerzNFT public nft;
    IERC20 public token;

    constructor(address _vesting, address _nft, address _token) {
        vesting = IAlignerzVesting(_vesting);
        nft = IAlignerzNFT(_nft);
        token = IERC20(_token);
    }

    function getTotalUnclaimedAmounts() external returns (uint256 _total) {
        uint256 len = nft.getTotalMinted();
        for (uint i = 0; i < len;) {
            (, bool exists) = safeOwnerOf(i);
            if (exists) _total += getUnclaimedAmounts(i);
            unchecked { ++i; }
        }
    }

    function getUnclaimedAmounts(uint256 nftId) public returns (uint256) {
        IAlignerzVesting.Allocation memory alloc = vesting.allocationOf(nftId);
        if (address(alloc.token) != address(token)) return 0;
        return 1 ether;  
    }

    function safeOwnerOf(uint256 nftId) public view returns (address, bool) {
        try nft.extOwnerOf(nftId) returns (address o) { return (o, true); }
        catch { return (address(0), false); }
    }
}

// Minimal mocks
contract MockERC20 is ERC20 {
    constructor(string memory n, string memory s) ERC20(n, s) {}
}

contract MockNFT is ERC721, IAlignerzNFT {
    uint256 public totalMinted;
    constructor() ERC721("MockNFT", "MNFT") {}
    function mint(address to, uint256 id) external { _mint(to, id); totalMinted++; }
    function extOwnerOf(uint256 id) external view returns (address) { return ownerOf(id); }
    function getTotalMinted() external view returns (uint256) { return totalMinted; }
}

contract MockVesting is IAlignerzVesting {
    IERC20 public tokenAddr;
    constructor(address _token) { tokenAddr = IERC20(_token); }
    function allocationOf(uint256) external view returns (Allocation memory alloc) {
        alloc.token = tokenAddr;           // Match distributor token

        alloc.amounts = new uint256[](1);
        alloc.amounts[0] = 1 ether;

        alloc.claimedSeconds = new uint256[](1);
        alloc.claimedSeconds[0] = 0;

        alloc.vestingPeriods = new uint256[](1);
        alloc.vestingPeriods[0] = 1;

        alloc.claimedFlows = new bool[](1);
        alloc.claimedFlows[0] = false;
    }
}

contract A26ZDividendDistributorPoC is Test {
    A26ZDividendDistributorMock distributor;
    MockERC20 token;
    MockNFT nft;
    MockVesting vesting;
    address dummy = makeAddr("dummy");

    function setUp() public {
        token = new MockERC20("A26Z", "A26Z");
        nft = new MockNFT();
        vesting = new MockVesting(address(token));
        distributor = new A26ZDividendDistributorMock(address(vesting), address(nft), address(token));
    }

    function test_GasDoS() public {
        uint256 numNFTs = 500;
        for (uint i=0; i<numNFTs; i++) nft.mint(dummy, i);

        uint256 gasStart = gasleft();
        uint256 total = distributor.getTotalUnclaimedAmounts();
        uint256 gasUsed = gasStart - gasleft();

        console.log("Num NFTs:", numNFTs);
        console.log("Total unclaimed:", total / 1e18, "A26Z");
        console.log("Gas used:", gasUsed);

        console.log("Estimated gas for 10k NFTs:", (gasUsed * 10000) / numNFTs);
    }
}

```

### Mitigation

To prevent gas-related DoS, implement batching or pagination when looping over NFTs and allocations. Instead of processing all tokens in a single transaction, distribute dividends in smaller chunks across multiple transactions. Alternatively, consider off-chain computation of totals with on-chain verification, such as using Merkle proofs, to minimize on-chain iteration and gas consumption.
  