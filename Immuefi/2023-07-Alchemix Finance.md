## Bug Description
AlchemixToken v2 does not spend allowance when flash loan is returned and burnt.

It does not conform to [ERC3156 standard](https://eips.ethereum.org/EIPS/eip-3156#flash-lending-security-considerations).

According to the IERC3156 standard `IERC3156FlashBorrower` will approve `amount + fee` to `IERC3156FlashLender` before calling `IERC3156FlashLender.flashloan` and `IERC3156FlashLender.flashloan` should spend allowance given `amount + fee`.

So, at the end of flashloan, there should be no dust allowance remaining.

But `AlchemicTokenV2.flashloan` does not spend allowance after flashloan is over.

https://github.com/alchemix-finance/v2-foundry/blob/master/src/AlchemicTokenV2.sol#L232
https://github.com/alchemix-finance/v2-foundry/blob/master/src/AlchemicTokenV2Base.sol#L231

```solidity
  function flashLoan(
    IERC3156FlashBorrower receiver,
    address token,
    uint256 amount,
    bytes calldata data
  ) external override nonReentrant returns (bool) {
	...

    uint256 fee = flashFee(token, amount);

    _mint(address(receiver), amount);

    if (receiver.onFlashLoan(msg.sender, token, amount, fee, data) != CALLBACK_SUCCESS) {
      revert IllegalState();
    }

    _burn(address(receiver), amount + fee); // Will throw error if not enough to burn

    return true;
  }
```

## Impact
Even after flashloan is over, there will be dust approval remaining, but user will not be aware of it.

## Risk Breakdown
Difficulty to Exploit: Medium
Weakness: Medium
CVSS2 Score: 6.5

## Recommendation

Should include following ERC3156 standard and spendAllowance given by the user before buring token.

https://github.com/radiant-capital/v2/blob/main/contracts/lending/libraries/logic/ValidationLogic.sol

```solidity
https://github.com/alchemix-finance/v2-foundry/blob/master/src/AlchemicTokenV2.sol#L232
```solidity
  function flashLoan(
    IERC3156FlashBorrower receiver,
    address token,
    uint256 amount,
    bytes calldata data
  ) external override nonReentrant returns (bool) {
	...

    uint256 fee = flashFee(token, amount);

    _mint(address(receiver), amount);

    if (receiver.onFlashLoan(msg.sender, token, amount, fee, data) != CALLBACK_SUCCESS) {
      revert IllegalState();
    }

    _burn(address(receiver), amount + fee); // Will throw error if not enough to burn

    return true;
  }
```

## References
https://github.com/alchemix-finance/v2-foundry/blob/master/src/AlchemicTokenV2.sol#L232
https://github.com/alchemix-finance/v2-foundry/blob/master/src/AlchemicTokenV2Base.sol#L231
https://eips.ethereum.org/EIPS/eip-3156#flash-lending-security-considerations

## Proof of Concept
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.10;

import "forge-std/Test.sol";
import "./interface.sol";

interface IAlchemixTokenV2 is IERC20 {
	function flashFee(address token, uint256 amount) external view returns (uint256 fee);
	function flashLoan(
		IERC3156FlashBorrower receiver,
		address token,
		uint256 amount,
		bytes calldata data
	) external returns (bool success);
}


contract AlchemixTokenV2Exploit is Test {

    IAlchemixTokenV2 alcx = IAlchemixTokenV2(0xF4B1486DD74D07706052A33d31d7c0AAFD0659E1);
    CheatCodes cheats = CheatCodes(0x7109709ECfa91a80626fF3989D68f67F5b1DD12D);
    address victim =  0xDA63F22BF4bDC0B88536bDf4375fc9E14862ABD8;
	uint256 amount = 1 ether;

    function setUp() public {
        cheats.createSelectFork("arbitrum", 95144404);
        deal(address(this), 0);
    }

    function testExp() external {
        vm.startPrank(victim);

        uint256 _fee = alcx.flashFee(amount);
        uint256 _repayment = amount + _fee;
        IERC20(token).increaseAllowance(address(alcx), _repayment);

        console.log("Flashloan...");
        lender.flashLoan(this, token, amount, data);
		console.log("Approval after Flashloan - " + alcx.allowance(victim, alcx));
    }
}
```