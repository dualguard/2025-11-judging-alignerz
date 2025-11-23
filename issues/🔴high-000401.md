# [000401] Swap‑and‑pop feature remove the pending KOL addresses from the kolTVSAddresses array.
  
  ### Summary

The `_claimRewardTVS()` function is used to claim TVS allocation. After minting the TVS NFT and transferring allocation data, the function removes the KOL from the `kolTVSAddresses` array using a swap-and-pop pattern.


### Root Cause

The intended logic:
1. Find KOL’s index
2. Swap it with the last array element
3. Update the swapped element’s index
4. Pop the last element (removing the KOL)

However, the implementation introduces a critical bug in the removal logic:
```solidity
rewardProject.kolTVSIndexOf[kol] = arrayLength - 1;
```

This line incorrectly overwrites the removed KOL’s index with the last index, even though the KOL is being removed.
This creates a stale mapping entry and causes index corruption, breaking the fundamental assumption that `kolTVSIndexOf[address]` always points to a valid array index.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

.

### Impact

- Stale / Phantom indices: Removed addresses still appear "indexed" even though they no longer exist in the array.
- State corruption when adding a KOL again
Re-adding a previously removed KOL will assign a new correct index but the stale mapping may cause: overwrite of a correct index, incorrect swaps and unpredictable behavior
- Hard-to-debug inconsistencies: Off-chain scripts or future contract logic may read incorrect index values and misinterpret array membership.

### PoC

```solidity
contract KolIndexHarness {
    // Here both are extracted from RewardProject for demo
    address[] public kolTVSAddresses;
    mapping(address => uint256) public kolTVSIndexOf;

    // Helper: seed addresses and mapping
    function seed(address[] memory addrs) external {
        delete kolTVSAddresses;
        // Clear mapping entries from prior runs
        for (uint256 i = 0; i < addrs.length; i++) {
            delete kolTVSIndexOf[addrs[i]];
        }
        for (uint256 i = 0; i < addrs.length; i++) {
            kolTVSIndexOf[addrs[i]] = i;
            kolTVSAddresses.push(addrs[i]);
        }
    }

    // Original buggy removal logic from AligerzVesting.sol for demo purpose
    function removeBuggy(address kol) external {
        uint256 index = kolTVSIndexOf[kol]; 
        uint256 arrayLength = kolTVSAddresses.length;

        // bug: incorrectly updates removed KOL index to last
        kolTVSIndexOf[kol] = arrayLength - 1;

        address lastIndexAddress = kolTVSAddresses[arrayLength - 1];
        kolTVSIndexOf[lastIndexAddress] = index;
        kolTVSAddresses[index] = kolTVSAddresses[arrayLength - 1];
        kolTVSAddresses.pop();

        // note: it does not delete kol's mapping entry (stale).
    }

    // Helpers for tests
    function addrs() external view returns (address[] memory) {
        return kolTVSAddresses;
    }

    function contains(address a) external view returns (bool) {
        for (uint256 i = 0; i < kolTVSAddresses.length; i++) {
            if (kolTVSAddresses[i] == a) return true;
        }
        return false;
    }
}

contract KolIndexRemovalTest is Test {
    KolIndexHarness h;

    address A = address(0xA1);
    address B = address(0xB2);
    address C = address(0xC3);
    address D = address(0xD4);
    address E = address(0xE5);

    function setUp() public {
        h = new KolIndexHarness();
    }

    // --- BUG DEMO ---

    function test_buggy_removal_last_leaves_stale_mapping() public {
        address[] memory init = new address[](4);
        init[0] = A; init[1] = B; init[2] = C; init[3] = D;
        h.seed(init);

        // Remove last (D)
        h.removeBuggy(D);

        // Array should be [A,B,C]
        address[] memory arr = h.addrs();
        assertEq(arr.length, 3, "length mismatch after pop");
        assertEq(arr[0], A);
        assertEq(arr[1], B);
        assertEq(arr[2], C);

        // BUG: mapping still says D has index 3 (stale phantom)
        uint256 staleIdx = h.kolTVSIndexOf(D);
        assertEq(staleIdx, 3, "expected stale index for removed element");
        // Also, array length is 3 → valid indices are [0..2]; 3 is out-of-bounds
    }

    function test_buggy_removal_middle_keeps_removed_in_mapping() public {
        address[] memory init = new address[](4);
        init[0] = A; init[1] = B; init[2] = C; init[3] = D;
        h.seed(init);

        // Remove middle (B at index 1)
        h.removeBuggy(B);

        // Array should become [A, D, C]
        address[] memory arr = h.addrs();
        assertEq(arr.length, 3, "length mismatch after pop");
        assertEq(arr[0], A);
        assertEq(arr[1], D);
        assertEq(arr[2], C);

        // BUG: mapping for B remains non-deleted and points to 3
        uint256 staleIdx = h.kolTVSIndexOf(B);
        assertEq(staleIdx, 3, "expected stale index for removed element");
        // This will break membership checks or future logic relying on kolTVSIndexOf
    }
}
```


### Mitigation

Remove this line:

```solidity
rewardProject.kolTVSIndexOf[kol] = arrayLength - 1;
```

And optionally, clear kol's entry after removal:

```solidity
delete rewardProject.kolTVSIndexOf[kol];
```

```solidity
uint256 index = rewardProject.kolTVSIndexOf[kol];
uint256 lastIndex = rewardProject.kolTVSAddresses.length - 1;
address lastAddress = rewardProject.kolTVSAddresses[lastIndex];
// Move last element into the removed slot
rewardProject.kolTVSAddresses[index] = lastAddress;
// Update its index
rewardProject.kolTVSIndexOf[lastAddress] = index;
// Remove last array element
rewardProject.kolTVSAddresses.pop();
// Delete removed KOL index
delete rewardProject.kolTVSIndexOf[kol];
```
  