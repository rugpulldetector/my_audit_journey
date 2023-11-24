# General
1) Unchecked returned value of transfer of ERC20 token, Should use `SafeERC20.safeTransfer`.
```
-       collateralToken.transfer(msg.sender, amount); // @audit -unchecked return value of transfer
+       collateralToken.safeTransfer(msg.sender, amount);
```
2) Incompatible with fee-on-transfer or rebasing ERC20 tokens
3) Lack of `__Ownable_init()` and `__ReentrancyGuard_init()` at initialize(), constructor with `disableInitializer()`
4) Should inherit upgradeable counterpart of base contracts for upgradeability

# Governance.sol

1) Malfunctioning vote counting mechanism, anyone with large balance can execute proposal irrespective of number of vote for proposal.
```
        if (balanceOf[msg.sender] >= quorumVotes) { // @audit - invalid vote counting, should not use executor's balance
```

2) Number of votes cast for proposal is not tracked in `vote` or `revokeVote`.

2) Total supply is not updated when balanceOf is updated at `createProposal()`, `vote()`, `revokeVote()`
```
balanceOf[msg.sender] = balanceOf[msg.sender].sub(proposalDeposit);
balanceOf[msg.sender] = balanceOf[msg.sender].sub(1);
balanceOf[msg.sender] = balanceOf[msg.sender].add(1);
```

4) Unreachable code
```
        if (block.timestamp >= proposal.votingEndTime) {
            // If voting ends, reset hasVoted status
            hasVoted[msg.sender] = false;
        }
```

# LendingPool.sol
1) All borrowing token reserves can be drained by calling `borrow()` time and again with amount smaller than `collateralBalance / 1.5`.

- CR should be calculated as `Collateral / Debt` not `Collateral / Borrowing Amount`.

```
-       require(collateralBalance[msg.sender] >= amount.mul(minCollateralRatio).div(100), "Insufficient collateral"); 
        uint256 interest = amount.mul(interestRate).div(100);
        uint256 totalRepayment = amount.add(interest);
        borrowBalance[msg.sender] = borrowBalance[msg.sender].add(totalRepayment); 
+       require(collateralBalance[msg.sender] >= borrowBalance.mul(minCollateralRatio).div(100), "Insufficient collateral"); 
```

2) Collateral can be withdrawn without repaying remaining debt.
```
    function removeCollateral(uint256 amount) public nonReentrant {
        ...
        collateralBalance[msg.sender] = collateralBalance[msg.sender].sub(amount); // @audit - use unchecked to save gas
+       require(collateralBalance[msg.sender] >= borrowBalance.mul(minCollateralRatio).div(100), "Insufficient collateral"); 
```

3) Important operations like `addCollateral()`, `removeCollateral()`, `borrow()`, `repay()` are not pausable because of missing `whenNotPaused` modifier.

```
-    function removeCollateral(uint256 amount) public nonReentrant {
+    function removeCollateral(uint256 amount) public whenNotPaused nonReentrant {
```

4) Theres' no incentive for borrower to repay earlier because interest to be paid is calculated only once and does not grow over time.

```
        uint256 interest = amount.mul(interestRate).div(100);
        uint256 totalRepayment = amount.add(interest);
```

5) No upper limit for interest rate, owner can rug user by sandwiching `borrow()` with `setInterestRate()`.

6) Unused `collateralizationBonus` maybe for liquidator.

# LiquidityPool.sol
1) Unable to withdraw if `withdrawalCooldown` >= `withdrawalWindow` which is true by default
```
        uint256 cooldownEndTime = block.timestamp + withdrawalCooldown;
        if (block.timestamp < cooldownEndTime) {
            // @audit - unable to withdraw, if withdrawalCooldown >= withdrawalWindow 
            require(block.timestamp + withdrawalWindow >= cooldownEndTime, "Withdrawal window closed"); 
        } 
```
2) Wrong access control for `setWithdrawCooldown()`

```
    function setWithdrawalCooldown(uint256 cooldown) public {
-        require(msg.sender == owner() && isadmin(msg.sender), "Not the owner or admin"); // @audit - should be ||
+        require(msg.sender == owner() || isadmin(msg.sender), "Not the owner or admin"); 
        withdrawalCooldown = cooldown;
    }
```

# Token.sol
