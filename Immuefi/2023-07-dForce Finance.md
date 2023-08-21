# `StatusOracle`, `PriceOracle` will fail to get status of an asset if timezone for status oracle is negative/western(UTC-x).

## `StatusOracle`
### Getting status oracle with negative timezone will revert.

https://github.com/dforce-network/LendingContractsV2/blob/master/contracts/StatusOracle.sol#L338-L345
```solidity
    function getAssetPriceStatus(address _asset) external view returns (bool) {
        return _getAssetStatus(_asset, block.timestamp);
    }

    function getAssetStatus(address _asset, uint256 _timestamp) external view returns (bool) {
        return _getAssetStatus(_asset, _timestamp);
    }
```

`StatusOracle._getAssetStatus()` will check asset's status given timestamp and timezone whether market is open/close or in holiday.

To do so, it will calculate local time.

https://github.com/dforce-network/LendingContractsV2/blob/master/contracts/StatusOracle.sol#L298-L303
```solidity
    function _getAssetStatus(address _asset, uint256 _timestamp) internal view returns (bool) {

        // Timestamp converted to the corresponding time zone.
        uint256 _timeZone = uint256(timeZone);
        uint256 _localTime = _timestamp + _timeZone; // @audit - might revert if timeZone < 0
```

If `timeZone` is `-1 hour`, `_timeZone` will become `type(uint256).max + 1 - 1 hour` by casting into `uint256`.

Then `_localTime = _timestamp + _timeZone` will overflow, thus reverted.

## `PriceOracle`
### Getting oracle status for an asset with negative timezone will revert.

https://github.com/dforce-network/LendingContractsV2/blob/master/contracts/PriceOracle.sol#L1546-L1557
```solidity
    function _getAssetPriceStatus(address _asset) internal view returns (bool) {

        IStatusOracle _statusOracle = statusOracle[_asset];
        if (_statusOracle == IStatusOracle(0))
            return true;

        return _statusOracle.getAssetPriceStatus(_asset);
    }

    function getAssetPriceStatus(address _asset) external view returns (bool) {
        return _getAssetPriceStatus(_asset);
    }
```

### Setting status oracle with negative timezone will revert.

https://github.com/dforce-network/LendingContractsV2/blob/master/contracts/PriceOracle.sol#L1338-L1358
```solidity
  function _setAssetStatusOracle(address _asset, IStatusOracle _statusOracle)
        public
        returns (uint256)
    {
...
        _statusOracle.getAssetPriceStatus(_asset);
        
        statusOracle[_asset] = _statusOracle;
        emit SetAssetStatusOracle(_asset, _statusOracle);

        return uint256(Error.NO_ERROR);
    }
```

## Impact
1) `StatusOracle` will fail to deliver status of oracle if timezone is negative.

2) `PriceOracle` will fail to get status of oracle or set status oracle for given asset if timezone is negative.

## Recommendation
Local time calculation should be changed as following.
https://github.com/dforce-network/LendingContractsV2/blob/master/contracts/StatusOracle.sol#L298-L303
```solidity
    function _getAssetStatus(address _asset, uint256 _timestamp) internal view returns (bool) {

        // Timestamp converted to the corresponding time zone.
-       uint256 _timeZone = uint256(timeZone);
-       uint256 _localTime = _timestamp + _timeZone;
+       uint256 _localTime = uint256(int256(_timestamp) + timezone);
```

## Proof of Concept
```solidity

// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.10;

import "forge-std/Test.sol";

interface PriceOracle {
    function _setAssetStatusOracle(address _asset, address _statusOracle) external;
}
interface StatusOracle {
    function _setTimeZone(int256 _timeZone) external;
}

contract ContractTest is DSTest {
    CheatCodes    cheat = CheatCodes(0x7109709ECfa91a80626fF3989D68f67F5b1DD12D);
    PriceOracle   priceOracle = (PriceOracle)0x34BAf46eA5081e3E49c29fccd8671ccc51e61E79;
    StatusOracle  statusOracle = (StatusOracle)0x0965BD5C993a012C7A5f2212E0c95fD1B45e3506
    address       asset = 0x2f956b2f801c6dad74E87E7f45c94f6283BF0f45;
    address       owner = 0x6F43161E3A56501ea14B2901132A4d9F0945E179;

  function setUp() public {
    cheats.createSelectFork("mainnet"); 
  }

  function testExploit() public {
    vm.prank(owner);
    statusOracle._setTimeZone(-1 hour);
    
    cheats.expectRevert();
    statusOracle.getAssetPriceStatus(asset);

    cheats.expectRevert();
    priceOracle._setAssetStatusOracle(asset, statusOracle);
  }
}

```