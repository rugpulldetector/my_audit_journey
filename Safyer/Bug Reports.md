# General
### 1) High - Unchecked returned value of transfer of ERC20 token, Should use `SafeERC20.safeTransfer`.
```
-       collateralToken.transfer(msg.sender, amount); // @audit -unchecked return value of transfer
+       collateralToken.safeTransfer(msg.sender, amount);
```
### 2) Low - Incompatibility with fee-on-transfer or rebasing ERC20 tokens
### 3) Low - Lack of zero address parameter validation check for constructor or `initialize()` 
### 4) Informational - Lack of `__Ownable_init()` and `__ReentrancyGuard_init()` at initialize(), constructor with `_disableInitializer()`, and it should inherit upgradeable counterpart of base contracts like `OwnableUpgradeable` for upgradeable contracts.

# Governance.sol

### 5) Critical - Malfunctioning vote counting mechanism, anyone with large balance can execute proposal irrespective of number of vote for proposal.
```
        if (balanceOf[msg.sender] >= quorumVotes) { // @audit - invalid vote counting, should not use executor's balance
```

### 6) High - Number of votes cast for proposal are not counted in `vote` or `revokeVote`.

### 7) High - Total supply is not updated when balanceOf is updated at `createProposal()`, `vote()`, `revokeVote()`
```
balanceOf[msg.sender] = balanceOf[msg.sender].sub(proposalDeposit);
balanceOf[msg.sender] = balanceOf[msg.sender].sub(1);
balanceOf[msg.sender] = balanceOf[msg.sender].add(1);
```
### 8) Low - Can revoke vote after voting duration has passed.

### 9) Informational - Unreachable code
```
        if (block.timestamp >= proposal.votingEndTime) {
            // If voting ends, reset hasVoted status
            hasVoted[msg.sender] = false;
        }
```
### 10) Informational - Lack of storage gap

# LendingPool.sol
### 11) Critical - All borrowing token reserves can be drained by calling `borrow()` time and again with amount smaller than `collateralBalance / 1.5`.

- CR should be calculated as `Collateral / Debt` not `Collateral / Borrowing Amount`.

```
-       require(collateralBalance[msg.sender] >= amount.mul(minCollateralRatio).div(100), "Insufficient collateral"); 
        uint256 interest = amount.mul(interestRate).div(100);
        uint256 totalRepayment = amount.add(interest);
        borrowBalance[msg.sender] = borrowBalance[msg.sender].add(totalRepayment); 
+       require(collateralBalance[msg.sender] >= borrowBalance.mul(minCollateralRatio).div(100), "Insufficient collateral"); 
```

### 12) Critical - Collateral can be removed without repaying remaining debt.
```
    function removeCollateral(uint256 amount) public nonReentrant {
        ...
        collateralBalance[msg.sender] = collateralBalance[msg.sender].sub(amount); // @audit - use unchecked to save gas
+       require(collateralBalance[msg.sender] >= borrowBalance.mul(minCollateralRatio).div(100), "Insufficient collateral"); 
```

### 13) Medium - Wrong assumption that collateral and borrowing tokens's prices are pegged to each other.
- To calculate collateralization ratio, there should be an oracle that feeds prices of collateral and borrowing token 

### 14) Medium - Wrong assumption that collateral and borrowing tokens's decimals are same.
To caluclate collateralization ratio, decimals difference should be taken into consideration.

### 15) Medium - `addCollateralAllowance`,`removeCollateralAllowance` can be frontrun by `addCollateral()` to spend remaining allowance and bypass lowered allowance. It's like `apporve` / `transferFrom`.
- `increaseAllowance`, `decreaseAllowance` functions are preferred.

### 16) Meidum - Important operations like `addCollateral()`, `removeCollateral()`, `borrow()`, `repay()` are not pausable because of missing `whenNotPaused` modifier.

```
-    function removeCollateral(uint256 amount) public nonReentrant {
+    function removeCollateral(uint256 amount) public whenNotPaused nonReentrant {
```

### 17) Medium - No liquidation mechanism to keep TCR healthy
There should be an incentive for liquidators who liquidates undercollateralization position.

### 18) Low - No upper limit for interest rate, owner can rug user by sandwiching `borrow()` with `setInterestRate()`.

### 19) Informational - Theres' no incentive for borrower to repay earlier because interest to be paid is calculated only once and does not grow over time.

```
        uint256 interest = amount.mul(interestRate).div(100);
        uint256 totalRepayment = amount.add(interest);
```

### 20) Informational - Unused `collateralizationBonus` maybe for liquidator.

# LiquidityPool.sol

### 21) High - Wrong logic for cooldown and withdrawal window check. 
- Cooldown end time should be mapping storage variable per each address.
- block.timestamp should be between [cooldownEndTime, cooldownEndTime + withdrawalWindow]
- It should be updated after successful withdrawal.
```
    mapping(address => uint256) public cooldownEndTime;

    function withdraw(uint256 amount) public nonReentrant whenNotPaused {
        ...
-       uint256 cooldownEndTime = block.timestamp + withdrawalCooldown;
-       if (block.timestamp < cooldownEndTime) {
-           require(block.timestamp + withdrawalWindow >= cooldownEndTime, "Withdrawal window closed");
-       }
+       require(block.timestamp > cooldownEndTime[msg.sender], "Cooldown period");
+       require(block.timestamp <= cooldownEndTime[msg.sender] + withdrawalWindow, "Withdrawal window closed");
+       cooldownEndTime[msg.sender] = block.timestamp + withdrawalCooldown;
        ...
    }
```
### 22) High - Unable to withdraw if `withdrawalCooldown` >= `withdrawalWindow` which is true by default
```
        uint256 cooldownEndTime = block.timestamp + withdrawalCooldown;
        if (block.timestamp < cooldownEndTime) {
            // @audit - unable to withdraw, if withdrawalCooldown >= withdrawalWindow 
            require(block.timestamp + withdrawalWindow >= cooldownEndTime, "Withdrawal window closed"); 
        } 
```
### 23) Medium - Wrong access control for `setWithdrawCooldown()`

```
    function setWithdrawalCooldown(uint256 cooldown) public {
-        require(msg.sender == owner() && isadmin(msg.sender), "Not the owner or admin"); // @audit - should be ||
+        require(msg.sender == owner() || isadmin(msg.sender), "Not the owner or admin"); 
        withdrawalCooldown = cooldown;
    }
```
### 24) Informational - Lack of storage gap
### 25) Informational - Missing event for important parameter changes like `minCollateralRatio`, `interestRate`, `maxLoanAmount`, `collateralizationBonus`


# Token.sol
### 26) Medium - If `burnFee` is greater than `transferFee`, it might revert because of lack of token balance.

- Let's say amount = balanceOf(msg.sender).

- After transfer to `to`, remaining balance will be feeAmount.
```
        super.transfer(to, netAmount);
```
- If burnFee is greater than tranferFee, burnAmount will be greater than remaining balance, thus `_burn()` will revert.
```
            uint256 burnAmount = amount.mul(burnFee).div(100);
            _burn(msg.sender, burnAmount); // @audit if fee amount is smaller than burn fee, it will create insolvency.
```

### 27) Medium - Transfer fee to be collected can be lost, as it is not deducted from `msg.sender`.

 - `transfer()` should send tranfer fee to treausry address, otherwise it can be transferred and fee to be collected are lost.
```
+       _transfer(msg.sender, feeTreasury, feeAmount);
        totalFees[msg.sender] = totalFees[msg.sender].add(feeAmount); // @audit - collected fee can be transfered.
```

### 28) Medium - Net transfer amount should be subtracted by burn amount.

```
-       net amount  = total amount - fee amount
+       net amount  = total amount - fee amount - burn amount
```
For every transfer, amount should be divided to three addresses.

- net amount for `to` address
- fee amount for `feeTreasury` address
- burn amount for `zero` address


```
    function transfer(address to, uint256 amount) public override returns (bool) {
        uint256 feeAmount = amount.mul(transferFee).div(100);
-       uint256 netAmount = amount.sub(feeAmount);
-       super.transfer(to, netAmount);
         ...
+       uint256 netAmount = amount.sub(feeAmount).sub(burnAmount);
+       super.transfer(to, netAmount);

        emit FeeCollected(msg.sender, feeAmount);
        return true;
    }
```

### 29) Informational - Lack of transfer fee collection mechanism
- Transfer fee could be either escrowed to current contract or directly sent to treasury address.
- In case transfer fee is escrowed to current current. there should be function to collect fee to treasury address.

