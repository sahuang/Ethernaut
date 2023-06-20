# 12 - Privacy

## Challenge

The creator of this contract was careful enough to protect the sensitive areas of its storage.

Unlock this contract to beat the level.

Things that might help:

- Understanding how storage works
- Understanding how parameter parsing works
- Understanding how casting works

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Privacy {

  bool public locked = true;
  uint256 public ID = block.timestamp;
  uint8 private flattening = 10;
  uint8 private denomination = 255;
  uint16 private awkwardness = uint16(block.timestamp);
  bytes32[3] private data;

  constructor(bytes32[3] memory _data) {
    data = _data;
  }
  
  function unlock(bytes16 _key) public {
    require(_key == bytes16(data[2]));
    locked = false;
  }

  /*
    A bunch of super advanced solidity algorithms...

      ,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`
      .,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,
      *.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^         ,---/V\
      `*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.    ~|__(o.o)
      ^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'  UU  UU
  */
}
```

## Summary

This level is actually very similar to [08 - Vault](../08%20-%20Vault/). In that level, we covered storage layout in EVM and mentioned that we can use `getStorageAt` to read storage even if the field is `private`. Here we can do the similar thing.

## Walkthrough

```js
> await web3.eth.getStorageAt(await contract.address, 0)
'0x0000000000000000000000000000000000000000000000000000000000000001'
> await web3.eth.getStorageAt(await contract.address, 1)
'0x00000000000000000000000000000000000000000000000000000000648ca98c'
```

The first two storage slots contain `locked` set to 1 and `ID` set to the current block timestamp. Recall that EVM will try to pack variables into the same slot if possible. If two or more variables fit into a single 32-byte slot, they are packed into the same slot, beginning on the right. The `locked` variable is a `bool` which is a single byte. The `ID` variable is a `uint256` which is 32 bytes. So they aren't packed together.

```js
> await web3.eth.getStorageAt(await contract.address, 2)
'0x00000000000000000000000000000000000000000000000000000000a98cff0a'
```

Starting from the right, we packed `flattening = 0x0a`, `denomination = 0xff`, and `awkwardness`. The `awkwardness` variable is a `uint16` which is 2 bytes.

```js
> await web3.eth.getStorageAt(await contract.address, 3)
'0x2fcb76d41641ad1632f611deff42aa5ec050c37ab2a64ada87bd2000b4596c41'
> await web3.eth.getStorageAt(await contract.address, 4)
'0xf59b1f533e0bb77b48c6b9ef2b7d4a416dbd809a49393fdb48ddfc72de7a27e2'
await web3.eth.getStorageAt(await contract.address, 5)
> '0xcc52b101d16c857f2c25dfd6cd994caa408bff7628c99612b0ec2757b497ab23'
```

Here it stores `bytes32[3] private data`, with each array element as one slot. Since we need `data[2]`, it would be `0xcc52b101d16c857f2c25dfd6cd994caa408bff7628c99612b0ec2757b497ab23`.

Now, the `_key` is actually a `bytes16` instead of `bytes32`. We should look at how Solidy casts bytes. According to [Solidity's documentation](http://solidity.readthedocs.io/en/develop/types.html#explicit-conversions), casting from `bytes32` to `bytes16` will truncate the lower bytes. `uint` conversion is actually the opposite:

```js
uint32 a = 0x12345678;
uint16 b = uint16(a); // b will be 0x5678 now

bytes2 a = 0x1234;
bytes1 b = bytes1(a); // b will be 0x12 now
```

This means `0xcc52b101d16c857f2c25dfd6cd994caa` is the `_key`. Now we just need to call `await contract.unlock('0xcc52b101d16c857f2c25dfd6cd994caa');` to set `locked` to `false`.

Finally, submit the instance to pass the level.

## Afterword

Nothing in the ethereum blockchain is private. The keyword private is merely an artificial construct of the Solidity language. Web3's `getStorageAt(...)` can be used to read anything from storage. It can be tricky to read what you want though, since several optimization rules and techniques are used to compact the storage as much as possible.