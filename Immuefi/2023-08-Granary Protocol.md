No check for Chainlink price feed staleness and L2 sequencer uptime feed status. Massive liquidations possible in event of Chainlink price feed staleness and L2 sequencer down.

## Bug Description
1) It does not check if timestamp of price returned is outdated.

https://github.com/The-Granary/Granary-Protocol-v1/blob/main/contracts/misc/AaveOracle.sol#L88-L103
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

2) It does not check if L2 sequencer is down or in grace period.

Here's Aave v3's implemntation for L2 sequencer down event check.

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

Unfortunately Granary Finance which is Aave v2 fork is vulnerable to L2 sequencer down event.

When Arbitrum L2 sequncer is down or restared in less than grace period, price feed will stop being updated and become stale.

ETH/USD Chainlink Oracle feed has two trigger parameters: Deviation threshold and Heartbeat. That means that the price feed will update if the price moves by at least 0.5% or, at latest, every hour (Heartbeat for ETH/USD price feed).

If Sequencer’s downtime exceeds the delay time implemented on the emergency procedure to submit transactions to the network, users are theoretically able to use stale prices to execute transactions.

Furthermore, L2 sequencer down event happens occasionally.
- https://cryptotvplus.com/2023/06/bug-disrupts-eth-l2s-arbitrum-network/
- https://news.bitcoin.com/arbitrum-network-stalled-due-to-sequencer-downtime/
- https://coincu.com/54104-arbitrum-network-failed-again-due-sequencer/

## Impact
An innocent user might be a target of unfair massive liquidation by MEV searchers in event of Chainlink price feed staleness and L2 sequencer down.

## Recommendation

1) Should check oracle staleness.
```solidity
    function staleCheckLatestRoundData(AggregatorV3Interface priceFeed)
        public
        view
        returns (uint80, int256, uint256, uint256, uint80)
    {
        (uint80 roundId, int256 answer, uint256 startedAt, uint256 updatedAt, uint80 answeredInRound) =
            priceFeed.latestRoundData();

        uint256 secondsSince = block.timestamp - updatedAt;
        if (secondsSince > TIMEOUT) revert OracleLib__StalePrice();

        return (roundId, answer, startedAt, updatedAt, answeredInRound);
    }
```
2) Should include `PriceOracleSentiel.sol` https://github.com/aave/aave-v3-core/blob/27a6d5c83560694210849d4abf09a09dec8da388/contracts/protocol/configuration/PriceOracleSentinel.sol

3) Should validate L2 sequencer uptime status before borrow and liquidation.

https://github.com/The-Granary/Granary-Protocol-v1/blob/main/contracts/protocol/libraries/logic/ValidationLogic.sol
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