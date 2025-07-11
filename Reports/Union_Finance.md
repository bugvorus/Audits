*Platform : Sherlock*

Status: Confirmed

Severity: High

### VouchFacet::claimTokens lacks to update the state after operation causes caller to claim token much more than they should.

### Summary
The claimTokens function allows users to request a specific amount of tokens. However, it needs to ensure that the requested amount is not more than the maximum amount the user is allowed to claim. This is to prevent users from withdrawing more tokens than they are entitled to. The function checks how many tokens the user has already claimed using the claimedTokens mapping but currently does not update the mapping after operation, which ensures malicious caller to call this token again and again and end up with much more token then they should have.


### Root Cause:
Root cause of this vulnerability is the missing mapping state update before the token is transferred to the caller.

### Code-Snippet:
https://github.com/sherlock-audit/2024-06-union-finance-update-2/blob/main/union-v2-contracts/contracts/peripheral/VouchFaucet.sol#L93

### Impact
This can potentially leads to caller to claim more token than what they actually own.

### Mitigation
I recommed the following changes in the function:

```diff
function claimTokens(address token, uint256 amount) external {
        require(claimedTokens[token][msg.sender] <= maxClaimable[token], "amount>max"); 
+       require(amount <= maxClaimable[token], "amount>max");
+       claimedTokens[token][msg.sender] = claimedTokens[token][msg.sender] + amount;
        IERC20(token).transfer(msg.sender, amount); 
        emit TokensClaimed(msg.sender, token, amount);
```
