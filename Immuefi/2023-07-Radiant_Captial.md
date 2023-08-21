## Bug Description
It doesn't check Arbitrum L2 sequncer is down or restared in less than grace period when getting ETH/USD price feed, thus innocent users might be bounty-hunted.

`PriceProvider.getLpTokenPriceUsd()` relies on Chainlink ETH/USD price feed.

https://github.com/radiant-capital/v2/blob/main/contracts/radiant/oracles/PriceProvider.sol#L90-L94
```solidity
	function getLpTokenPriceUsd() public view returns (uint256 price) {
		// decimals 8
		uint256 lpPriceInEth = getLpTokenPrice();
		// decimals 8
		uint256 ethPrice = uint256(baseTokenPriceInUsdProxyAggregator.latestAnswer());
		price = lpPriceInEth.mul(ethPrice).div(10 ** 8);
	}
```

`BountyManager._canBountyHunt()` relies on `BountyManager.mindLPBalance()` and `PriceProvider.getLpTokenPriceUsd()` to check if a user can be kicked out as bounty.

https://github.com/radiant-capital/v2/blob/main/contracts/radiant/eligibility/BountyManager.sol#L197-L201
```solidity
	function _canBountyHunt(address _user) internal view returns (bool) {
		(, , uint256 lockedLP, , ) = IMFDPlus(mfd).lockedBalances(_user);
		bool isEmissionsEligible = IEligibilityDataProvider(eligibilityDataProvider).isEligibleForRewards(_user);
		return lockedLP >= minDLPBalance() && isEmissionsEligible;
	}

	function minDLPBalance() public view returns (uint256 min) {
		uint256 lpTokenPrice = IPriceProvider(priceProvider).getLpTokenPriceUsd();
		min = minStakeAmount.mul(1e8).div(lpTokenPrice);
	}
```
But when Arbitrum L2 Sequencer is down or in grace period, price feed will stop being updated and become stale.

ETH/USD Chainlink Oracle feed has two trigger parameters: Deviation threshold and Heartbeat. That means that the price feed will update if the price moves by at least 0.5% or, at latest, every hour (Heartbeat for ETH/USD price feed).

If Sequencer’s downtime exceeds the delay time implemented on the emergency procedure to submit transactions to the network, users are theoretically able to use stale prices to execute transactions.

Furthermore, L2 sequencer down event happens occasionally.
- https://cryptotvplus.com/2023/06/bug-disrupts-eth-l2s-arbitrum-network/
- https://news.bitcoin.com/arbitrum-network-stalled-due-to-sequencer-downtime/
- https://coincu.com/54104-arbitrum-network-failed-again-due-sequencer/

## Impact
An innocent user might be a target for bounty hunting.


## Risk Breakdown
Difficulty to Exploit: Medium
Weakness: High
CVSS2 Score: 7.5

## Recommendation
Should following the best practice given by Chainlink.
1) `PriceProvider` should check if L2 sequencer is down or restarted in less than grace period when query oracle feed.
2) `IChainlinkAggregator.latestAnswer()` is deprecated. use `IChainlinkAggregator.latestRoundData()`.

https://github.com/radiant-capital/v2/blob/main/contracts/radiant/oracles/PriceProvider.sol

```solidity
	IChainlinkAggregator public baseTokenPriceInUsdProxyAggregator;
+	IChainlinkAggregator public sequencerUptimeFeed;
+   uint256 private constant GRACE_PERIOD_TIME = 3600;

+   error SequencerDown();
+   error GracePeriodNotOver();

	function initialize(
		IChainlinkAggregator _baseTokenPriceInUsdProxyAggregator,
+       IChainlinkAggregator _sequencerUptimeFeed,
		IPoolHelper _poolHelper
	) public initializer {
		require(address(_baseTokenPriceInUsdProxyAggregator) != (address(0)), "Not a valid address");
		require(address(_poolHelper) != (address(0)), "Not a valid address");
		__Ownable_init();

		poolHelper = _poolHelper;
		baseTokenPriceInUsdProxyAggregator = _baseTokenPriceInUsdProxyAggregator;
+		sequencerUptimeFeed = _sequencerUptimeFeed;
		usePool = true;
	}

+    function getLatestETHPrice() public view returns (int) {
+       if (sequencerUptimeFeed != address(0)) {
+			// prettier-ignore
+        	(
+           	/*uint80 roundID*/,
+	           	int256 answer,
+   	    	uint256 startedAt,
+      		    /*uint256 updatedAt*/,
+            	/*uint80 answeredInRound*/
+        	) = sequencerUptimeFeed.latestRoundData();

+        	// Answer == 0: Sequencer is up
+        	// Answer == 1: Sequencer is down
+        	bool isSequencerUp = answer == 0;
+        	if (!isSequencerUp) {
+           	 revert SequencerDown();
+        	}

+        	// Make sure the grace period has passed after the
+        	// sequencer is back up.
+        	uint256 timeSinceUp = block.timestamp - startedAt;
+        	if (timeSinceUp <= GRACE_PERIOD_TIME) {
+           	 revert GracePeriodNotOver();
+        	}
+		}

+       // prettier-ignore
+       (
+           /*uint80 roundID*/,
+           int data,
+           /*uint startedAt*/,
+           /*uint timeStamp*/,
+           /*uint80 answeredInRound*/
+       ) = dataFeed.latestRoundData();

+       return data;
+   }

	function getTokenPriceUsd() public view returns (uint256 price) {
		if (usePool) {
-			uint256 ethPrice = uint256(IChainlinkAggregator(baseTokenPriceInUsdProxyAggregator).latestAnswer());
+			uint256 ethPrice = uint256(getLatestETHPrice());
			uint256 priceInEth = poolHelper.getPrice();
			price = priceInEth.mul(ethPrice).div(10 ** 8);
		} else {
			price = oracle.latestAnswer();
		}
	}

	/**
	 * @notice Returns lp token price in ETH.
	 */
	function getLpTokenPriceUsd() public view returns (uint256 price) {
		// decimals 8
		uint256 lpPriceInEth = getLpTokenPrice();
		// decimals 8
-		uint256 ethPrice = uint256(baseTokenPriceInUsdProxyAggregator.latestAnswer());
+		uint256 ethPrice = uint256(getLatestETHPrice());
		price = lpPriceInEth.mul(ethPrice).div(10 ** 8);
	}

```
## References
https://docs.chain.link/data-feeds/l2-sequencer-feeds#arbitrum
https://medium.com/@lopotras/l2-sequencer-and-stale-oracle-prices-bug-54a749417277

## Proof of Concept
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.10;

import "forge-std/Test.sol";
import "./interface.sol";

interface BountyManager{
	function claim(
		address _user,
		uint256 _actionType
	) external;
}


contract RadiantExploit is Test {

    BountyManager bountyManager = BountyManager(0xcbbd19e676d9b66bf139e2728aa29de783442ac9);
    CheatCodes cheats = CheatCodes(0x7109709ECfa91a80626fF3989D68f67F5b1DD12D);
    address victim =  0xDA63F22BF4bDC0B88536bDf4375fc9E14862ABD8;

    function setUp() public {
        cheats.createSelectFork("arbitrum", 95144404);
        deal(address(this), 0);
    }

    function testExp() external {
        console.log("L2 Sequncer Down - Oracle price is stale");
        vm.startPrank(attacker);
        bountyManager.claim(victim);        
    }
}
```