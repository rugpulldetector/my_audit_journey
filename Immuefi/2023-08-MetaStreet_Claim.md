# Liquidation will not be finalized if currency token transfer to borrower failed. Lock of funds and collateral.

## Bug Description
`Pool.onCollateralLiquidated()` uses push pattern to return surplus fund to borrower after auction is over.

https://github.com/metastreet-labs/metastreet-contracts-v2/blob/master/contracts/Pool.sol#L1086
```solidity
    function onCollateralLiquidated(bytes calldata encodedLoanReceipt, uint256 proceeds) external nonReentrant {
		...

        /* Transfer surplus to borrower */
        if (borrowerSurplus != 0) IERC20(_currencyToken).safeTransfer(loanReceipt.borrower, borrowerSurplus);

        /* Emit Collateral Liquidated */
        emit CollateralLiquidated(loanReceiptHash, proceeds, borrowerSurplus);
    }
```

If currency token transfer is failed, liquidation will not be finalized and collateral and funds will be locked.

There might be several reasons why currency token transfer might fail.

- Blacklisted by currency token. Some stablecoins have freezing or blacklisting feature that might block transfer to certain address.

- Paused in case of emergency

Here's code for Binance USD stablecoin which checks if contract is paused or sender or receiver is frozen address.

https://etherscan.io/address/0x5864c777697bf9881220328bf2f16908c9afcd7e#code
```
    string public constant name = "Binance USD";
    string public constant symbol = "BUSD";
    uint8 public constant decimals = 18;

	...

    function transfer(address _to, uint256 _value) public whenNotPaused returns (bool) {
        require(_to != address(0), "cannot transfer to address zero");
        require(!frozen[_to] && !frozen[msg.sender], "address frozen");
        require(_value <= balances[msg.sender], "insufficient funds");

        balances[msg.sender] = balances[msg.sender].sub(_value);
        balances[_to] = balances[_to].add(_value);
        emit Transfer(msg.sender, _to, _value);
        return true;
    }
```


## Impact

### Risk
- Permeanent locking of funds and collateral if borrower is blacklisted.

- Temporary locking of funds and collateral if contract is unpaused.

### Weakness
As pool creation is permissionless as mentioned following, any user can create pool by using `PoolFactory.create()` with any currency token as long as its decimal is 18, e.g. `BUSD`.

https://github.com/metastreet-labs/metastreet-contracts-v2/blob/master/docs/DESIGN.md#L19-L20
```
**Permissionless**: Allow users to instantiate a lending pool for any collection permissionlessly
```

## Risk Breakdown
Difficulty to Exploit: Medium
Weakness: High
CVSS2 Score: 8

## Recommendation
Please follow pull over push pattern so that recipient of fund has to claim funds.

```solidity
+	mapping(address => uint256) private claimable;
+    function claim(address recipient) {
+		require(claimable[recipient] > 0, "Nothing to claim");
+		claimable[recipient] = 0;
+        IERC20(liquidation_.currencyToken).safeTransfer(recipient, claimable);
+	}
```

https://github.com/metastreet-labs/metastreet-contracts-v2/blob/master/contracts/Pool.sol#L1086
```solidity
    function onCollateralLiquidated(bytes calldata encodedLoanReceipt, uint256 proceeds) external nonReentrant {
		...

        /* Transfer surplus to borrower */
-       if (borrowerSurplus != 0) IERC20(_currencyToken).safeTransfer(loanReceipt.borrower, borrowerSurplus); // @audit - blacklisted transfer
+       if (borrowerSurplus != 0) claimable[loanReceipt.borrower] += borrowerSurplus; 

        /* Emit Collateral Liquidated */
        emit CollateralLiquidated(loanReceiptHash, proceeds, borrowerSurplus);
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

    function testMetaStreetClaim() external {
        uint128 tick = 1e26 << 8;
        uint128[] memory ticks = new uint128[](1);
        ticks[0] = tick;

        console.log("Step 1: Depositing...");
        cheats.startPrank(depositor);
        busd.approve(address(pool), type(uint256).max);
		pool.deposit(tick, 5 ether, 0);

        console.log("Step 2: Borrowing...");
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

        console.log("Step 3.2 Freezing BUSD asset of borrower...");
		cheats.startPrank(busd.assetProtectionRole());
		busd.freeze(borrower);

        console.log("Step 3.3 Successful bidder cannot claim NFT if borrower is frozen.");
        cheats.warp(block.timestamp + 2 days);
		cheats.startPrank(bidder);
		cheats.expectRevert();
        liquidator.claim(coolCats, coolCatsId, encodedLoanReceipt);
    }
}
```

## Output

```
[PASS] testMetaStreetClaim() (gas: 750065)
Logs:
  New pool created with BUSD...
  Step 1: Depositing...
  Step 2: Borrowing...
  Step 3: Liquidating...
  Step 3.1: Bidding...
  Step 3.2 Freezing BUSD asset of borrower...
  Step 3.3 Successful bidder cannot claim collateral if borrower is frozen. 
```