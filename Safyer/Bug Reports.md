# Sayfer Audit Report

## 32 Issues Found
- 3 Critical
- 5 High
- 12 Medium
- 4 Low
- 8 Informational

# General
### 1) High - Unchecked returned value of transfer of ERC20 token, Should use `SafeERC20.safeTransfer`.
```
-       collateralToken.transfer(msg.sender, amount); // @audit -unchecked return value of transfer
+       collateralToken.safeTransfer(msg.sender, amount);
```
### 2) Medium - Incompatibility with fee-on-transfer or rebasing ERC20 tokens

### 3) Low - Lack of zero address parameter validation check for constructor or `initialize()` 

```
    function initialize(address _liquidityToken) public initializer() {
        require(liquidityToken != address(0), "Zero address");
        ...
    }
```
### 4) Informational - If upgradeability was intented, it should inherit upgradeable counterpart of base contracts
- Lack of `__Ownable_init()`, `__ReentrancyGuard_init()`, `__Pausable_init()` at initialize()
- constructor with `_disableInitializer()`
```
contract LiquidityPool is Initializable, OwnableUpgradeable, PausableUpgradeable, ReentrancyGuardUpgradeable {
    constructor () {
        _disableInitializer();
    }

    function initialize(address _liquidityToken) public initializer() {
        __Ownable_init();
        __ReentrancyGuard_init();
        __Pausable_init();
        ...
    }
```

# Governance.sol

### 5) Critical - Malfunctioning vote counting mechanism, anyone with large balance can execute proposal irrespective of number of vote for proposal.
```
        if (balanceOf[msg.sender] >= quorumVotes) { // @audit - invalid vote counting, should not use executor's balance
```

### 6) High - Number of votes cast for proposal are not counted in `vote` or `revokeVote`.

### 7) High - No checkpointing mechanism by block number.
- `createProposal()` should record block number at which proposal was created.
- Voter's checkpointed past balance and total supply at proposal creation block number should be used for vote counting given proposal.

### 8) Medium - It allows voter to vote multiple times, and it consume voter's balance each time, which is not required.
- If voter has changed mind, he can revoke vote, so there's no need to allow voting twice.

```
+       require(voted[msg.sender] == false, "No second voting"); // @audit - no double voting
```

### 9) Medium - Total supply is not updated when balanceOf is updated at `createProposal()`, `vote()`, `revokeVote()`
```
balanceOf[msg.sender] = balanceOf[msg.sender].sub(proposalDeposit);
balanceOf[msg.sender] = balanceOf[msg.sender].sub(1);
balanceOf[msg.sender] = balanceOf[msg.sender].add(1);
```

### 10) Low - No upper limit for `quorumPercentage`. If quorumPercentage is over 100%, no proposal can be executed.
```
    function setQuorumPercentage(uint256 percentage) public onlyOwner {
+       require(percentage < 100000, "Upper limit of quorum");
        ...
    }
```
### 11) Low - Can revoke vote after voting duration has passed.
```
    function revokeVote(uint256 proposalId) public nonReentrant onlyExistingProposal(proposalId) {
        Proposal storage proposal = proposals[proposalId];
+        require(block.timestamp < proposal.votingEndTime, "Voting has ended");
        ...
   }
```

### 12) Informational - Unreachable code because of require() at the start of function.
```
    function executeproposal(uint256 proposalId) public nonReentrant onlyExistingProposal(proposalId) {
        ...
        require(block.timestamp < proposal.votingEndTime, "Voting has ended");
        ...
        if (block.timestamp >= proposal.votingEndTime) {
            // If voting ends, reset hasVoted status
            hasVoted[msg.sender] = false;
        }
```
### 13) Informational - Lack of storage gap if upgradeability was inteneded

# LendingPool.sol
### 14) Critical - All borrowing token reserves can be drained by calling `borrow()` time and again with amount smaller than `collateralBalance / 1.5`.

- CR should be calculated as `Collateral / Debt` not `Collateral / Borrowing Amount`.

```
-       require(collateralBalance[msg.sender] >= amount.mul(minCollateralRatio).div(100), "Insufficient collateral"); 
        uint256 interest = amount.mul(interestRate).div(100);
        uint256 totalRepayment = amount.add(interest);
        borrowBalance[msg.sender] = borrowBalance[msg.sender].add(totalRepayment); 
+       require(collateralBalance[msg.sender] >= borrowBalance.mul(minCollateralRatio).div(100), "Insufficient collateral"); 
```

### 15) Critical - Collateral can be removed without repaying remaining debt.
```
    function removeCollateral(uint256 amount) public nonReentrant {
        ...
        collateralBalance[msg.sender] = collateralBalance[msg.sender].sub(amount); // @audit - use unchecked to save gas
+       require(collateralBalance[msg.sender] >= borrowBalance.mul(minCollateralRatio).div(100), "Insufficient collateral"); 
```

### 16) Medium - Wrong assumption that collateral and borrowing tokens's prices are pegged to each other.
- To calculate collateralization ratio, there should be an oracle that feeds prices of collateral and borrowing token 

### 17) Medium - Wrong assumption that collateral and borrowing tokens's decimals are same.
- To caluclate collateralization ratio, decimals difference should be taken into consideration.

```
+       uint256 debtInCollateralToken = debt * 10 ** collateralToken.decimals() / 10 ** borrowedToken.decimals();
```

### 18) Medium - `addCollateralAllowance`,`removeCollateralAllowance` can be frontrun by `addCollateral()` to spend remaining allowance and bypass lowered allowance. It's like `apporve` / `transferFrom`.
- `increaseAllowance`, `decreaseAllowance` functions are preferred.

### 19) Medium - Important operations like `addCollateral()`, `removeCollateral()`, `borrow()`, `repay()` are not pausable because of missing `whenNotPaused` modifier.

```
-    function removeCollateral(uint256 amount) public nonReentrant {
+    function removeCollateral(uint256 amount) public whenNotPaused nonReentrant {
```

### 20) Medium - No liquidation mechanism to keep TCR healthy
There should be an incentive for liquidators who liquidates undercollateralization position.

### 21) Low - No upper limit for interest rate, owner can rug user by sandwiching `borrow()` with `setInterestRate()`.

### 22) Informational - Theres' no incentive for borrower to repay earlier because interest to be paid is calculated only once and does not grow over time.

```
        uint256 interest = amount.mul(interestRate).div(100);
        uint256 totalRepayment = amount.add(interest);
```

### 23) Informational - Unused `collateralizationBonus` maybe for liquidator.

# LiquidityPool.sol

### 24) High - Wrong logic for cooldown and withdrawal window check. 
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
### 25) High - Unable to withdraw if `withdrawalCooldown` >= `withdrawalWindow` which is true by default
```
        uint256 cooldownEndTime = block.timestamp + withdrawalCooldown;
        if (block.timestamp < cooldownEndTime) {
            // @audit - unable to withdraw, if withdrawalCooldown >= withdrawalWindow 
            require(block.timestamp + withdrawalWindow >= cooldownEndTime, "Withdrawal window closed"); 
        } 
```
### 26) Medium - Wrong access control for `setWithdrawCooldown()`

```
    function setWithdrawalCooldown(uint256 cooldown) public {
-        require(msg.sender == owner() && isadmin(msg.sender), "Not the owner or admin"); // @audit - should be ||
+        require(msg.sender == owner() || isadmin(msg.sender), "Not the owner or admin"); 
        withdrawalCooldown = cooldown;
    }
```
### 27) Informational - Lack of storage gap if upgradeability was inteneded
### 28) Informational - Missing event for important parameter changes like `minCollateralRatio`, `interestRate`, `maxLoanAmount`, `collateralizationBonus`


# Token.sol
### 29) Medium - If `burnFee` is greater than `transferFee`, it might revert because of lack of token balance.

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

### 30) Medium - Transfer fee to be collected can be lost, as it is not deducted from `msg.sender`.

 - `transfer()` should send tranfer fee to treausry address, otherwise it can be transferred and fee to be collected are lost.
```
+       _transfer(msg.sender, feeTreasury, feeAmount);
        totalFees[msg.sender] = totalFees[msg.sender].add(feeAmount); // @audit - collected fee can be transfered.
```

### 31) Medium - Net transfer amount should be subtracted by burn amount.

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

### 32) Informational - Lack of transfer fee collection mechanism
- Transfer fee could be either escrowed to current contract or directly sent to treasury address.
- In case transfer fee is escrowed to current current. there should be function to collect fee to treasury address.

