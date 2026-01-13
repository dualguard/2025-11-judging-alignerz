# [000156] Incorrect split logic causes loss of funds
  
  The split function reuses the same modified allocation data in a loop, causing subsequent calculations to be based on already reduced values instead of the original amount.

Assume a originalNftId with an allocation of 1000 USDC, where the user wants to split it in two of same percentage. The user 

First Loop Iteration:

- Loop processes the first 50% split (percentages[0]).
- It calculates the new amount: 1000 * 50% = 500.
- The code updates the allocation of the original NFT in storage, setting its amounts[0] to 500. The original allocation has now been overwritten.

Second Loop Iteration:

- Loop proceeds to the second 50% split (percentages[1]).
- When it reads the amount to be split, it doesn't read the original 1000. Instead, it reads the current value from the now-modified original allocation, which is 500.
- The function calculates the amount for the second NFT (splitNftId2): 500 * 50% = 250.
- This new allocation with an amount of 250 is stored for the newly minted NFT.

The Result is that the sum of them will be worth 750 rather than 1000, making 25% of their value permanently lost.

  