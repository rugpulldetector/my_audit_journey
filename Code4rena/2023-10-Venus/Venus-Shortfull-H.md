## Summary

Lack of fund at risk fund might block auction to be closed. Successful bidder will lose gas fee because of tx revert.

## Severity

High

## Proof of Concept
- Shortfull.closeAuction()

https://github.com/VenusProtocol/isolated-pools/blob/develop/contracts/Shortfall/Shortfall.sol#L269-L272
```solidity
        address convertibleBaseAsset = riskFund.convertibleBaseAsset();

        uint256 transferredAmount = riskFund.transferReserveForAuction(comptroller, riskFundBidAmount);
        _transferOutOrTrackDebt(IERC20Upgradeable(convertibleBaseAsset), auction.highestBidder, riskFundBidAmount);
```

- RiskFund.transferReserveForAuction()

https://github.com/VenusProtocol/isolated-pools/blob/develop/contracts/RiskFund/RiskFund.sol#L2334
```
    function transferReserveForAuction(
        address comptroller,
        uint256 amount
    ) external override nonReentrant returns (uint256) {
        ...
        IERC20Upgradeable(convertibleBaseAsset).safeTransfer(shortfall_, amount);
        ...
    }
```

- Current convertible base asset for risk fund is BUSD.
https://bscscan.com/token/0x55d398326f99059ff775485246999027b3197955?a=0xdF31a28D68A2AB381D42b380649Ead7ae2A76E42


## Impact
If multiple auction are going on and previous auction drains out risk fund, next auction will not be closed and winner will waste gas fee if he tries to close auction.

## Tools Used
Manual Review

## Recommendation
Risk fund balance should be escrowed to `Shortfall` contract when starting an auction.
```
    function _startAuction(address comptroller) internal {

        auction.seizedRiskFund = riskFundBalance - remainingRiskFundBalance;
+       riskFund.transferReserveForAuction(comptroller, auction.seizedRiskFund);
    }
```