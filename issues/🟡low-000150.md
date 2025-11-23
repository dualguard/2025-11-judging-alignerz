# [000150] withdrawPostDeadlineProfit deadline check does not prevent withdrawals for active bids
  
  This line of code is intended to be a safety check, ensuring that profits from a token sale can only be withdrawn by the owner after the user-facing deadline for claiming tokens or refunds has passed. The goal is to lock the funds in the contract during the bidding and claiming periods, providing security and trust to the participants.

However, the deadline variable is only given a non-zero value when the owner finalizes the project by calling finalizeBids. For any active project where bidding is open but not yet finalized, the deadline remains at its default value of zero. The check, therefore, becomes require(block.timestamp > 0), which is always true. 
  