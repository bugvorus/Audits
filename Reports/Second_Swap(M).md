*Platform: Code4arena*

Status: Confirmed

Severity: Medium

### Title: Incomplete `ListingType` input validation during vesting listing could lead to unexpected outcomes

## Description: The SecondSwap_Marketplace::listVesting function includes a require statement designed to validate the minPurchaseAmt based on the ListingType. 
The expected behavior is as follows:

For SINGLE listings, the minPurchaseAmt must be strictly equal to the _amount.
For PARTIAL listings, the minPurchaseAmt must be > 0 and <= _amount.

However, the current logic is implemented as follows:

```solidity
require(
            _listingType != ListingType.SINGLE || (_minPurchaseAmt > 0 && _minPurchaseAmt <= _amount),
            "SS_Marketplace: Minimum Purchase Amount cannot be more than listing amount"
        );
```

This implementation has a flaw due to how the logical OR (||) operator works. For PARTIAL listings, minPurchaseAmt will never be validated becauset he first condition 
`(_listingType != ListingType.SINGLE)` evaluates to true. And for SINGLE listings, the logic does not enforce that `(minPurchaseAmt == _amount)` instead, it allows unintended partial purchases.
This flawed logic introduces the risk of treating SINGLE listings as PARTIAL ones, potentially allowing partial purchases in scenarios where they are not 
intended. This discrepancy could result in unexpected behavior, undermining the purpose of SINGLE listings.

https://github.com/code-423n4/2024-12-secondswap/blob/214849c3517eb26b31fe194bceae65cb0f52d2c0/contracts/SecondSwap_Marketplace.sol#L253

## Proof Of Concept:
Potential Scenario:
User Aciva lists their vesting tokens with the following parameters:

_amount: 1000

listingType: PARTIAL

minPurchaseAmt: 1002

Aciva expects that the PARTIAL listing type will allow prospective buyers to purchase the tokens in multiple transactions, based on their requirements. However, 
the listing logic permits the minPurchaseAmt to be greater than the total _amount, which is logically incorrect. As a result, the listing is created successfully despite the invalid configuration.

When user Bobxa tries to purchase some tokens from Aciva, the purchase is rejected due to the following validation logic:

```solidity
require(
            listing.listingType == ListingType.SINGLE ||
                (_amount >= listing.minPurchaseAmt || _amount == listing.balance),
            "SS_Marketplace: Invalid Purchase amount"
        );
```
The above logic prevents Bobxa from completing the purchase because the minPurchaseAmt (1002) exceeds the total _amount (1000), making it impossible for any 
buyer to meet the minPurchaseAmt condition. This renders Aciva's listing unsellable due to the incorrect minPurchaseAmt setting during the creation of the listing.

## Recommended Mitigation Steps:
This can be implemented to avoid this issue:

```solidity
require(
    (_listingType != ListingType.SINGLE || (_minPurchaseAmt == _amount)) && (_minPurchaseAmt > 0 && _minPurchaseAmt <= _amount)
);
```

