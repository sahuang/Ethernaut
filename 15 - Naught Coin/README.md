# 15 - Naught Coin

## Challenge

NaughtCoin is an ERC20 token and you're already holding all of them. The catch is that you'll only be able to transfer them after a 10 year lockout period. Can you figure out how to get them out to another address so that you can transfer them freely? Complete this level by getting your token balance to 0.

Things that might help

- The [ERC20](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-20.md) Spec
- The [OpenZeppelin](https://github.com/OpenZeppelin/zeppelin-solidity/tree/master/contracts) codebase

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import 'openzeppelin-contracts-08/token/ERC20/ERC20.sol';

 contract NaughtCoin is ERC20 {

  // string public constant name = 'NaughtCoin';
  // string public constant symbol = '0x0';
  // uint public constant decimals = 18;
  uint public timeLock = block.timestamp + 10 * 365 days;
  uint256 public INITIAL_SUPPLY;
  address public player;

  constructor(address _player) 
  ERC20('NaughtCoin', '0x0') {
    player = _player;
    INITIAL_SUPPLY = 1000000 * (10**uint256(decimals()));
    // _totalSupply = INITIAL_SUPPLY;
    // _balances[player] = INITIAL_SUPPLY;
    _mint(player, INITIAL_SUPPLY);
    emit Transfer(address(0), player, INITIAL_SUPPLY);
  }
  
  function transfer(address _to, uint256 _value) override public lockTokens returns(bool) {
    super.transfer(_to, _value);
  }

  // Prevent the initial owner from transferring tokens until the timelock has passed
  modifier lockTokens() {
    if (msg.sender == player) {
      require(block.timestamp > timeLock);
      _;
    } else {
     _;
    }
  } 
} 
```

## Summary

As we can see, `NaughtCoin` is an ERC20 token which implements ERC20 [token standard](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-20.md). There is a `timeLock` of 10 years, only after that period can you pass the `lockTokens` modifier in `transfer`, so apparently we cannot call this function to transfer tokens.

This is more like an OSINT challenge now, since we need to find another way implemented in ERC20 to transfer funds outside. Just below `transfer` we see `transferFrom`:

> The `transferFrom` method is used for a withdraw workflow, allowing contracts to transfer tokens on your behalf. This can be used for example to allow a contract to transfer tokens on your behalf and/or to charge fees in sub-currencies. The function SHOULD `throw` unless the `_from` account has **deliberately authorized** the sender of the message via some mechanism.

Reading below, `approve` allows `_spender` to withdraw from your account and this will overwrite current `allowance` value. So we can call `approve` to set `allowance` to a large value, then call `transferFrom` to transfer tokens to our own account.

## Walkthrough

Let's first check balance:

```js
> await contract.balanceOf(player).then(v => v.toString())
'1000000000000000000000000'
```

Now, call `approve` to set `allowance` to a large value (same as above would actually just work fine). Remember we need to use current `player` account and approve one of your own account but it cannot be the same as `player`. To do this, we can create another MetaMask account, called `spender`, then call below from `player` connected browser.

```js
await contract.approve('0xbc347d947E698a2f81418BB8c25e8333bF1d34B2', '1000000000000000000000000')
```

Next, I used another browser that swictched to `spender` account, then deployed the following contract on Remix:

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface INaughtCoin {
    function transferFrom(
        address sender,
        address recipient,
        uint256 amount
    ) external returns (bool);
}
```

When deploying, use `contract` in level as `At Address`. Finally we call the `transferFrom` function with `sender = player`, `recipient = spender` and `amount = 1000000000000000000000000`. Upon completion,

```js
> await contract.balanceOf(player).then(v => v.toString())
0
```

Finally, submit the instance to pass the level.