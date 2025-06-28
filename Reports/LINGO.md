*Platform: Hackenproof*

Status: Valid finding but does not lies in-scope attack vectors

Severity: Medium

### Title: Incorrect total supply validation check in `LingoToken::mint` function can leads to potentially minting more tokens than `MAX_SUPPLY`

## Description: The LingoToken::mint function is designed to mint tokens for the address to with the specified amount. Before minting, it checks whether the new tokens would 
exceed the MAX_SUPPLY limit. However, the function fails to account for the initialSupply of tokens that are minted during contract deployment to the deployer. Since these 
tokens are already in circulation, ignoring them allows the MINTER_ROLE address to mint additional tokens, potentially exceeding the MAX_SUPPLY limit.

https://github.com/avicenne-studio/lingo-token-contracts/blob/8c46c38ae2b9b71be093a882f2c01302747d0d3b/contracts/LingoToken.sol#L152-L157

## Proof Of Concept:

```solidity
pragma solidity 0.8.20;

import "forge-std/Test.sol";
import "../contracts/LingoToken.sol";

contract LingoTokenTest is Test {
    LingoToken lingoToken;

    address admin = address(uint160(uint256(keccak256("admin"))));
    address treasury = address(uint160(uint256(keccak256("treasury"))));
    address minter = address(uint160(uint256(keccak256("minter"))));
    address user = address(uint160(uint256(keccak256("user"))));

    uint256 initialSupply = 100_000 * 10 ** 18; // 100,000 tokens
    uint256 maxSupply = 1_000_000 * 10 ** 18;  // 1,000,000 tokens
    uint256 transferFee = 500;                // 5%

    function setUp() public {
        vm.startPrank(admin);
        lingoToken = new LingoToken(initialSupply, treasury, transferFee);
        lingoToken.grantRole(lingoToken.MINTER_ROLE(), minter);
        vm.stopPrank();
    }

    function testMintExceedsMaxSupply() public {
        vm.startPrank(minter);

        uint256 remainingSupply = maxSupply - initialSupply;
        uint256 mintAmount = remainingSupply + 1; // exceeds the max supply

        lingoToken.mint(user, mintAmount);

        uint256 balance = lingoToken.balanceOf(user);
        emit log_named_uint("Balance of user:", balance);

        vm.stopPrank();
    }

}
```
