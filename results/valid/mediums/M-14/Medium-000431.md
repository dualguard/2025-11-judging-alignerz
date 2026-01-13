# [000431] A malicious user can ``honeypot`` others by using the ``splitNFT()`` function.
  
  ### Summary

A malicious user can ``honeypot`` others by using the ``splitNFT()`` function.

### Root Cause

As per the ``whitepaper``, 
The ``TVS NFTs``:
> can be bought, sold, or traded on secondary markets (NFT marketplaces), even
before the vesting matures.
Investors can exit anytime by selling their TVS instead of dumping unlocked tokens.
TVSs are splitable and mergeable, offering investors even more flexibility.

https://drive.google.com/file/d/1xN5bYUPd_BkBMtoRruHEO1CBUx0vBiit/edit?pli=1

But a malicious actor can easily honeypot ``buyers`` using the ``split()`` functionality.

```solidity
./AlignerzVesting.sol

 function splitTVS(
        uint256 projectId,
        uint256[] calldata percentages,
        uint256 splitNftId
    ) external returns (uint256, uint256[] memory) {
        address nftOwner = nftContract.extOwnerOf(splitNftId);
        require(msg.sender == nftOwner, Caller_Should_Own_The_NFT());

        bool isBiddingProject = NFTBelongsToBiddingProject[splitNftId];
        (Allocation storage allocation, IERC20 token) = isBiddingProject ?
        (biddingProjects[projectId].allocations[splitNftId], biddingProjects[projectId].token) :
        (rewardProjects[projectId].allocations[splitNftId], rewardProjects[projectId].token);

        uint256[] memory amounts = allocation.amounts;
        uint256 nbOfFlows = allocation.amounts.length;
        (uint256 feeAmount, uint256[] memory newAmounts) = calculateFeeAndNewAmountForOneTVS(splitFeeRate, amounts, nbOfFlows);
        allocation.amounts = newAmounts;
        token.safeTransfer(treasury, feeAmount);

        uint256 nbOfTVS = percentages.length;

        // new NFT IDs except the original one
        uint256[] memory newNftIds = new uint256[](nbOfTVS - 1);

        // Allocate outer arrays for the event
        Allocation[] memory allAlloc = new Allocation[](nbOfTVS);

        uint256 sumOfPercentages;
        for (uint256 i; i < nbOfTVS;) {
            uint256 percentage = percentages[i];
            sumOfPercentages += percentage;

            uint256 nftId = i == 0 ? splitNftId : nftContract.mint(msg.sender);
            if (i != 0) newNftIds[i - 1] = nftId;
...
```
The ``splitTVS()`` function is used to ``split`` a ``nft`` into ``multiple smaller nft position`` as per the ``percentage``. For example:

``NFT1`` can be split into 4 nfts per ``25% splitting``:

``NFT1, NFT2, NFT3 and NFT4`` where each ``NFT`` has now ``25%`` value of ``NFT1``.

The main problem with this approach is ``NFT1`` is not ``burned`` while ``NFT2``, ``NFT3`` and ``NFT4`` are ``minted new`` to the user.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. A malicious user ``Alice`` sets up a sale of his ``TVS NFT1`` at ``20%`` off.
2. ``Bob`` decides to buy the ``NFT1``.
3. Before transferring the ``NFT1``, ``Alice`` front-runs the transaction with ``splitTVS`` for two ``NFTs`` with percentages ``1``(can't be lower than this) and ``9999``.  Thus, ``NFT1`` has ``1%`` value and ``NFT2`` has ``99%`` value.
4. ``Bob`` is transferred ``NFT1`` with ``1%`` value.
5. ``Alice`` gets to keep ``NFT2`` with ``99%`` value and the ``amount`` from the ``sale``.

### Impact

The buyer will ``lose`` funds and will be the victim of `honeypot` for as less as `1%` of total value promised actually being provided to them.
The ``Attacker`` will get to keep as much as ``99%`` of original value + ``amount`` from the sale. This can be pretty huge ``amount`` as per the ``NFT``.

### PoC

_No response_

### Mitigation

``splitTVS()`` function should burn the original ``NFT``.
  