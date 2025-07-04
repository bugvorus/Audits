*Platform : Sherlock*

Status: Valid finding

Severity: High

### Title: Incorrent state update for the caller instead of from in ABONDToken::transferFrom function can lead to unexpected outcomes

## Description: The transferFrom function in the ABONDToken contract is designed to transfer tokens from the from address to the to address. It first updates the State 
structs of both the from and to addresses before transferring the tokens. However, during the state update, it mistakenly assigns the caller's (msg.sender) state to the
from address's updated state. This leads to an incorrect state change, where the caller's state is overwritten and the from address's state remains unmodified

https://github.com/sherlock-audit/2024-11-autonomint/blob/0d324e04d4c0ca306e1ae4d4c65f0cb9d681751b/Blockchain/Blockchian/contracts/Token/Abond_Token.sol#L165

## Root Cause:

```solidity
userStates[msg.sender] = fromState;  
```

## Impact: This results in the caller's state being updated to the modified from state, which is not the intended behavior.


## Attack Path:
Consider the aBondBalance of the from, to, and msg.sender (caller) before the transfer. For simplicity, cumulativeRate and ethBacked are not considered here, as they are updated differently:
to's state is adjusted based on the transferred amount, while from's cumulativeRate and ethBacked remain unchanged.

Initial Balances:
aBondBalance of from: 2000
aBondBalance of to: 2500
aBondBalance of msg.sender: 2800

The caller has an allowance of 1000 tokens from the from address and transfers these tokens to the to address.

Expected Balances After Transfer:

aBondBalance of from: 1000 (debited 1000 tokens)
aBondBalance of to: 3500 (credited 1000 tokens)
aBondBalance of msg.sender: 2800 (unchanged)

Actual Outcome with Current Logic:
The msg.sender's state is mistakenly overwritten with the updated state of the from address. As a result, the msg.sender loses 1800 tokens (their aBondBalance becomes 1000, the 
same as the updated from balance), which is incorrect.

## Mitigation:

```diff
-     userStates[msg.sender] = fromState;  
+    userStates[from] = fromState;
```
