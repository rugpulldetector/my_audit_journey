# `AlchemicTokenV2.flashloan()` and `AlchemicTokenV2Base.flashloan()` does not spend allowance approved by borrower after flashloan.

## Bug Description
After flashloan, those flash borrower will approve `amount + fee` token to lender and lender will have to spend allowance before burning, but unfortunately such logic is missing.

https://github.com/alchemix-finance/v2-foundry/blob/master/src/AlchemicTokenV2.sol#L210-L235
```
  function flashLoan(
    IERC3156FlashBorrower receiver,
    address token,
    uint256 amount,
    bytes calldata data
  ) external override nonReentrant returns (bool) {
    if (token != address(this)) {
      revert IllegalArgument();
    }

    if (amount > maxFlashLoan(token)) {
      revert IllegalArgument();
    }

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
After flashloan, allowance given by borrower will be increased by `amount+fee`.

## Recommendation

Should spend allowance approved by receiver before burning. or should use `burnFrom`.
```
  function flashLoan(
    IERC3156FlashBorrower receiver,
    address token,
    uint256 amount,
    bytes calldata data
  ) external override nonReentrant returns (bool) {
    ...

    if (receiver.onFlashLoan(msg.sender, token, amount, fee, data) != CALLBACK_SUCCESS) {
      revert IllegalState();
    }

+   _spendAllowance(address(receiver), address(this), amount + fee);
    _burn(address(receiver), amount + fee); // Will throw error if not enough to burn

    return true;
  }
```

## Proof of Concept
```
contract ERC3156FlashBorrowerMock is IERC3156FlashBorrower {
    bytes32 internal constant _RETURN_VALUE = keccak256("ERC3156FlashBorrower.onFlashLoan");

    function onFlashLoan(
        address, /*initiator*/
        address token,
        uint256 amount,
        uint256 fee,
        bytes calldata data
    ) public override returns (bytes32) {
        require(msg.sender == token);
        IERC20(token).approve(token, amount + fee);
        return _RETURN_VALUE;
    }
}


contract ContractTest is Test {
  function setUp() public {
    vm.createSelectFork("mainnet"); 
  }

  function testFlashloan() public {
    AlchemicTokenV2 alcx = new AlchemicTokenV2();
    deal(address(alcx), address(this), 1000 ether);

    alcx.flashloan(proxy, alcx, 900 ether, bytes(""));
    emit log_named_uint(alcx.allowance(address(this), address(alcx)));
  }
}
