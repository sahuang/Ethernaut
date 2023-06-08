# 00 - Hello Ethernaut

## Summary

This is a warmup challenge where I get to install various tools and learn how to interact with ABI. Below are the stuff needed throughout the Ethernaut series:

- [Metamask](https://metamask.io/) - A browser extension that allows me to interact with the Ethereum blockchain.
- [cheb3](https://github.com/YanhuiJessica/cheb3) - CTF tool based on web3.py.
- [Remix](https://remix.ethereum.org/) - An online IDE for Solidity.
- HTTP Endpoint - I am using [QuickNode](https://dashboard.quicknode.com/endpoints) but other options such as [infura](https://infura.io/) are also available. Eventually we get an endpoint such as `https://bitter-purple-general.ethereum-goerli.discover.quiknode.pro/<priv_api_key>/`.

A few other toolkits might be helpful, such as [Foundry](https://github.com/foundry-rs/foundry).

## Walkthrough

First we can follow the guide to get some information about the game:

```js
> player
'0x0b26C24d538e3dfF58F7c733535e65a6674FB3aB'
> getBalance(player)
Promise {<pending>}[[Prototype]]: Promise[[PromiseState]]: "fulfilled"[[PromiseResult]]: "4.116180836839137544"
> await ethernaut.owner()
'0x09902A56d04a9446601a0d451E07459dC5aF0820'
```

The we need to interact with the contract to complete the level. Note that on Chrome *Developer Tools*'s Console, there are auto-complete suggestions for the call which makes it much easier.

```js
> await contract.info()
'You will find what you need in info1().'
> await contract.info1()
'Try info2(), but with "hello" as a parameter.'
> await contract.info2("hello")
'The property infoNum holds the number of the next info method to call.'
> await contract.infoNum()
i {negative: 0, words: Array(2), length: 1, red: null}length: 1negative: 0red: nullwords: (2) [42, empty]0: 42length: 2[[Prototype]]: Array(0)[[Prototype]]: Object
> await contract.info42()
'theMethodName is the name of the next method.'
> await contract.theMethodName()
'The method name is method7123949.'
> await contract.method7123949()
'If you know the password, submit it to authenticate().'
> await contract.password()
'ethernaut0'
> await contract.authenticate('ethernaut0')
{tx: '0x48cd939d45447991f7f38b0fb93ae162953c3941bbd28a15a7f2ba07a81252c7', receipt: {…}, logs: Array(0)}
```

Submit the instance to pass the level.