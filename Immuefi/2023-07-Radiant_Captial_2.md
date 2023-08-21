## Bug Description
Aave v3 deployed in Arbitrum L2 chain does not allow borrow and liquidation when L2 sequencer is down or in grace period.

https://docs.aave.com/developers/core-contracts/priceoraclesentinel

https://github.com/aave/aave-v3-core/blob/27a6d5c83560694210849d4abf09a09dec8da388/contracts/protocol/configuration/PriceOracleSentinel.sol#L57C1-L81C1
```solidity
  /// @inheritdoc IPriceOracleSentinel
  function isBorrowAllowed() public view override returns (bool) {
    return _isUpAndGracePeriodPassed();
  }


  /// @inheritdoc IPriceOracleSentinel
  function isLiquidationAllowed() public view override returns (bool) {
    return _isUpAndGracePeriodPassed();
  }


  /**
   * @notice Checks the sequencer oracle is healthy: is up and grace period passed.
   * @return True if the SequencerOracle is up and the grace period passed, false otherwise
   */
  function _isUpAndGracePeriodPassed() internal view returns (bool) {
    (, int256 answer, , uint256 lastUpdateTimestamp, ) = _sequencerOracle.latestRoundData();
    return answer == 0 && block.timestamp - lastUpdateTimestamp > _gracePeriod;
  }


  /// @inheritdoc IPriceOracleSentinel
  function setSequencerOracle(address newSequencerOracle) public onlyPoolAdmin {
    _sequencerOracle = ISequencerOracle(newSequencerOracle);
    emit SequencerOracleUpdated(newSequencerOracle);
  }

```

Aave v2 does not implement such emergency protection mechanism, coincidentally never deployed to Arbitrum L2 chain.

Unfortunately Radiant Capital which is Aavve v2 fork is prone to such an issue.

https://github.com/radiant-capital/v2/blob/main/contracts/lending/AaveOracle.sol#L93
https://github.com/aave/protocol-v2/blob/ce53c4a8c8620125063168620eba0a8a92854eb8/contracts/misc/AaveOracle.sol#L96C1-L97C1

```solidity
	function getAssetPrice(address asset) public view override returns (uint256) {
		IChainlinkAggregator source = assetsSources[asset];

		if (asset == BASE_CURRENCY) {
			return BASE_CURRENCY_UNIT;
		} else if (address(source) == address(0)) {
			return _fallbackOracle.getAssetPrice(asset);
		} else {
			int256 price = IChainlinkAggregator(source).latestAnswer();
			if (price > 0) {
				return uint256(price);
			} else {
				return _fallbackOracle.getAssetPrice(asset);
			}
		}
	}
```

When Arbitrum L2 sequncer is down or restared in less than grace period, price feed will stop being updated and become stale.

ETH/USD Chainlink Oracle feed has two trigger parameters: Deviation threshold and Heartbeat. That means that the price feed will update if the price moves by at least 0.5% or, at latest, every hour (Heartbeat for ETH/USD price feed).

If Sequencer’s downtime exceeds the delay time implemented on the emergency procedure to submit transactions to the network, users are theoretically able to use stale prices to execute transactions.

Furthermore, L2 sequencer down event happens occasionally.
- https://cryptotvplus.com/2023/06/bug-disrupts-eth-l2s-arbitrum-network/
- https://news.bitcoin.com/arbitrum-network-stalled-due-to-sequencer-downtime/
- https://coincu.com/54104-arbitrum-network-failed-again-due-sequencer/

## Impact
An innocent user might be a target of unfair massive liquidation by MEV searchers.

## Risk Breakdown
Difficulty to Exploit: Medium
Weakness: High
CVSS2 Score: 7.5

## Recommendation

1) Should include `PriceOracleSentiel.sol` https://github.com/aave/aave-v3-core/blob/27a6d5c83560694210849d4abf09a09dec8da388/contracts/protocol/configuration/PriceOracleSentinel.sol

2) Should validate L2 sequencer uptime status before borrow and liquidation.

https://github.com/radiant-capital/v2/blob/main/contracts/lending/libraries/logic/ValidationLogic.sol

```solidity
  function validateLiquidationCall(
    DataTypes.UserConfigurationMap storage userConfig,
    DataTypes.ReserveData storage collateralReserve,
    DataTypes.ValidateLiquidationCallParams memory params
  ) internal view {
	...

+    require(
+      params.priceOracleSentinel == address(0) ||
+        params.healthFactor < MINIMUM_HEALTH_FACTOR_LIQUIDATION_THRESHOLD ||
+        IPriceOracleSentinel(params.priceOracleSentinel).isLiquidationAllowed(),
+      Errors.PRICE_ORACLE_SENTINEL_CHECK_FAILED
+    );

    ...
  }

	function validateBorrow(
		address asset,
		DataTypes.ReserveData storage reserve,
		address userAddress,
		uint256 amount,
		uint256 amountInETH,
		uint256 interestRateMode,
		uint256 maxStableLoanPercent,
		mapping(address => DataTypes.ReserveData) storage reservesData,
		DataTypes.UserConfigurationMap storage userConfig,
		mapping(uint256 => address) storage reserves,
		uint256 reservesCount,
		address oracle)
	{
	...
		...
+		require(
+		params.priceOracleSentinel == address(0) ||
+			IPriceOracleSentinel(params.priceOracleSentinel).isBorrowAllowed(),
+		Errors.PRICE_ORACLE_SENTINEL_CHECK_FAILED
+		);
		...
    ...
  }
```
3) For long-term, should migrate to Aave v3 which is more stable than v2.

## References
https://docs.chain.link/data-feeds/l2-sequencer-feeds#arbitrum
https://medium.com/@lopotras/l2-sequencer-and-stale-oracle-prices-bug-54a749417277

## Proof of Concept
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.10;

import "forge-std/Test.sol";
import "./interface.sol";

interface LendingPool{
	function liquidationCall(
		address collateralAsset,
		address debtAsset,
		address user,
		uint256 debtToCover,
		bool receiveAToken
	) external;
}


contract RadiantExploit is Test {

    LendingPool lendingPool = LendingPool(0xF4B1486DD74D07706052A33d31d7c0AAFD0659E1);
    CheatCodes cheats = CheatCodes(0x7109709ECfa91a80626fF3989D68f67F5b1DD12D);
    address victim =  0xDA63F22BF4bDC0B88536bDf4375fc9E14862ABD8;

    function setUp() public {
        cheats.createSelectFork("arbitrum", 95144404);
        deal(address(this), 0);
    }

    function testExp() external {
        console.log("L2 Sequncer Down - Oracle price is stale");

        vm.startPrank(attacker);
        lendingPool.liquidationCall(collateralAsset,
				debtAsset,
				victim,
				debtToCover,
				receiveAToken);        
    }
}
```