# 18 - MagicNumber

## Challenge

To solve this level, you only need to provide the Ethernaut with a `Solver`, a contract that responds to `whatIsTheMeaningOfLife()` with the right number.

Easy right? Well... there's a catch.

The solver's code needs to be really tiny. Really reaaaaaallly tiny. Like freakin' really really itty-bitty tiny: 10 opcodes at most.

Hint: Perhaps its time to leave the comfort of the Solidity compiler momentarily, and build this one by hand O_o. That's right: Raw EVM bytecode.

Good luck!

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract MagicNum {

  address public solver;

  constructor() {}

  function setSolver(address _solver) public {
    solver = _solver;
  }

  /*
    ____________/\\\_______/\\\\\\\\\_____        
     __________/\\\\\_____/\\\///////\\\___       
      ________/\\\/\\\____\///______\//\\\__      
       ______/\\\/\/\\\______________/\\\/___     
        ____/\\\/__\/\\\___________/\\\//_____    
         __/\\\\\\\\\\\\\\\\_____/\\\//________   
          _\///////////\\\//____/\\\/___________  
           ___________\/\\\_____/\\\\\\\\\\\\\\\_ 
            ___________\///_____\///////////////__
  */
}
```

## Summary

In this level, we essentially needs to deploy a contract that returns the magic number `42`. However, difficult part is it should be in 10 opcodes or less, so we have to look in depth into EVM opcodes.

I have no knowledge of this before starting this level, so no idea how to solve it. I started with these two articles to give myself some background:

- [Solidity Bytecode and Opcode Basics](https://medium.com/@blockchain101/solidity-bytecode-and-opcode-basics-672e9b1a88c2)
- [EVM bytecode programming](https://hackmd.io/@e18r/r1yM3rCCd#7-Deploying-our-first-bytecode-smart-contract-for-real)

The first article gives some basic knowledge of different opcodes and Solidity EVM intro. The second one teaches you how to deploy contract creation bytecode, including initialization code and runtime code.

In this level, we want runtime opcodes to return `42` and it needs to be within 10 bytes.

1. Put `42` in memory. 

```
602a // PUSH1 0x2a -> value is 42
6000 // PUSH1 0x00 -> memory location is 0x00
52   // MSTORE
```

2. Return the value in memory.

```
6020 // PUSH1 0x20 -> value is 32 bytes
6000 // PUSH1 0x00 -> memory location is 0x00
f3   // RETURN
```

This gives us the runtime opcodes `602a60005260206000f3`. Now we need to create the initialization code. We can follow how [Section 13 of EVM bytecode programming](https://hackmd.io/@e18r/r1yM3rCCd#13-Deploying-%E2%80%9Chola-mundo%E2%80%9D) does to create the initialization code. 

> A way to copy code into memory. That’s called 39. Things that start with 3 tend to be related to the user input. In this case, we are the user, and we are inputting the code. So 39 copies code into memory, and we must tell it where in memory we want to copy it, and the lenght and offset of the code we wanna copy.
>
> So we need to push the following things into the stack in order to use 39:
> 
> `[memoryPosition codePosition length]`

```
600a // PUSH1 0x0a -> runtime code is 10 bytes
60?? // PUSH1 0x?? -> runtime code starts at 0x??
6000 // PUSH1 0x00 -> memory location is 0x00
39   // CODECOPY
600a // return 10 bytes
6000 // which are in memory position 0
f3  // RETURN
```

Since we have in total 12 bytes in this initialization code, `??` is then `0c` in hex. Our final payload is `0x600a600c600039600a6000f3602a60005260206000f3`.

## Walkthrough

```py
>>> from cheb3 import Connection
>>> from cheb3.utils import compile_sol
>>> conn = Connection("https://goerli.infura.io/v3/<key>")
>>> account = conn.account("<priv_key>")
```

Send transaction to deploy the bytecode:

```py
>>> account.send_transaction(None, 0, '0x600a600c600039600a6000f3602a60005260206000f3')
2023-06-27 23:29:28.051 | INFO     | cheb3.account:send_transaction:99 - Transaction to None: 0x4c0a6ddfada20dae74702b583064e0f93498a117e168110eeb2e6bd76b435803
AttributeDict({'blockHash': HexBytes('0x9e8c796253fa13a50a8f9364efedba3dc87e21f9b4ed9c4a758ebd4e52609cfe'), 'blockNumber': 9253621, 'contractAddress': '0xC819ABb52Db95e7FF8BE7c98f6C5248b705761a6', 'cumulativeGasUsed': 14781889, 'effectiveGasPrice': 2541, 'from': '0x0b26C24d538e3dfF58F7c733535e65a6674FB3aB', 'gasUsed': 55330, 'logs': [], 'logsBloom': HexBytes('0x000..00'), 'status': 1, 'to': None, 'transactionHash': HexBytes('0x4c0a6ddfada20dae74702b583064e0f93498a117e168110eeb2e6bd76b435803'), 'transactionIndex': 49, 'type': 0})
```

In Devtool, call:

```js
> await contract.setSolver("0xC819ABb52Db95e7FF8BE7c98f6C5248b705761a6")
{tx: '0x5ddfe2b60cb605c7ccf2f99fa3407a951e7d01d8f8976d6204058e2b1c28dd02', receipt: {…}, logs: Array(0)}
```

As such, we set that contract to the solver. Finally, submit the instance to pass the level.
