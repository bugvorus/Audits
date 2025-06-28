*Platform : Immunefi*

Status: Out of Scope (only Critical and High was accepted)

Severity: Low

### Title: Incorrect minimum share calculation in `HourglassLiquidUSDLockDepositor` contract

## Brief Intro: The calculation method for minimum shares in the HourglassLiquidUSDLockDepositor contract introduces significant precision loss, potentially resulting in 
zero shares being minted even when depositing huge amount of assets, particularly for tokens with fewer decimals.

```solidity
        uint256 minShareAcceptable = (
            (((10 ** depositAssetDecimals) * amountToMintWith) / shareRate) * minMintReceivedSlippageBps
        ) / (10 ** (10 + depositAssetDecimals));
```

## Summary:
The mintLockedUnderlying function in the HourglassLiquidUSDLockDepositor contract is designed to mint Liquid USD tokens using a specified deposited asset. A critical 
part of this process involves calculating the minShareAcceptable value, which ensures that the minted shares meet a minimum threshold as per the user's slippage tolerance. 
This calculation plays a pivotal role in protecting liquidity providers (LPs) from unfavorable outcomes when minting tokens.
However, the current method for calculating minShareAcceptable has precision-related flaws that can significantly impact the minting process. These issues include scenarios
where liquidity providers receive far fewer shares than expected, potentially eroding their ownership of the underlying assets. For certain assets, particularly those with 
fewer decimals (e.g., USDT with 6 decimals), the calculation can result in a minShareAcceptable value of zero, causing LPs to receive no shares at all despite successfully depositing their assets.

In a system where minted shares represent ownership and value of the underlying assets, such inaccuracies can have severe implications for liquidity providers

## Proof Of Concept:
Consider a scenario where User A wants to deposit 1M USDC into the contract, with a shareRate of 100% (i.e., 1,000,000) and a slippage tolerance of 9800 (98%). Despite providing 
such a substantial amount of assets, the minShareAcceptable calculated by the contract would result in 0 due to the current formula.
Since slippage applies to the output asset and minShareAcceptable corresponds to the shares minted, setting this value to zero could potentially lead to users or liquidity 
providers (LPs) losing their funds in unfavorable scenarios.
To better illustrate this, I have attached a simplified contract and tested the output values under different input conditions in Remix. I've also attached an image showing the results for 
the scenario above.

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity 0.8.24;

contract TestShareOutput {

    function cloneReceipt(
        uint256 depositAssetDecimals, //6 for USDC
        uint256  amountToMintWith, //1M USDC
        uint256  shareRate, //100% (1_000_000)
        uint256 minMintReceivedSlippageBps //9800
        ) pure public returns (uint256 minShareAcceptable) {

     minShareAcceptable = (
            (((10 ** depositAssetDecimals) * amountToMintWith) / shareRate) * minMintReceivedSlippageBps
        ) / (10 ** (10 + depositAssetDecimals)); 

    }

}
```
