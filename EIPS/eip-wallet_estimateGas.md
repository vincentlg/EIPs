---
eip: eth_estimateGasRequirement
title: New Rpc method for an optimized gas estimation and an opcode to require a minimal gas value
author: Ronan Sandford (@wighawag)
type: Standards Track
category: Core
discussions-to: https://ethereum-magicians.org/t/eip-2075-fixing-eth-estimategas-with-a-new-opcode/3313
status: Draft
created: 2019-05-24
---

## Simple Summary
Add a variant of estimateGas that relies on a new EVM mechanism to keep track of gas requirement
Add an opcode to require a specific amount of gas (similar to ```require(gasleft()>X)``` in solidity) that allow ```eth_estimateGasRequirement``` to return a value that will represent enough gas to make the transaction succeed, even though the actual gas used will be less.

## Abstract

Currently, because contract can be using code like ```require(gasleft()>X)```, a call to ```eth_estimateGas``` is basically binary searching for the minimum gas amount required for the call to succeed.
This is because the EVM do not keep track of gas requirement implemented by the smart contract.

This proposal add a new mechanism to the EVM that can be used for estimation
it also add a new opcode that perform the same check as ```require(gasleft()>X)``` but ensure that ```eth_estimateGasRequirement``` return the minimum amount of gas required for the call to succeed (the use case for ```eth_estimateGas```)

### Specification
Let specify `minimalGas` as a new variable that the EVM need to keep track for the purpose of `eth_estimateGasRequirement`. At the start of a tx, it is set to zero.

At the end of an `eth_estimateGasRequirement` call, the gas spent is compared to `minimalGas`. The bigger value of the two is returned as "estimate". It does not have any other role in the context of a transaction call, its only purpose is to optimize gas estimation

Adds a new opcode ```REQUIRE_GAS``` at `<TBD>`, which uses 1 stack argument : a 32 bytes value that represent the gas required for the contract code to proceed. It will check if the gas left at that point is greater or equal to that value.

If not it will revert the call. 

If there is enough gas, the gas amount specified will be added to the gas spent up to that point. If that amount is bigger than `minimalGas` it replaces it. In other words:
```
minimalGas = max(minimalGas, X + <gas spent so far including the gas used for the opcode>)
```
where X is the value passed to `REQUIRE_GAS`

As mentioned the result of an `eth_estimateGasRequirement` is 
```
max(<gas used>, minimalGas)
```

The operation costs `G_base` + `G_verylow` to execute.

### Rationale

`eth_estimateGas` as currently implemented can be required to call the transaction 20 times. This is because it can't know whethere the smart contract is rejecting the transaction  based on the gas left.

Indeed when a contract have code like this
```
require(gasleft() > X)
```
where X is greater than the gas actually used after that check, the gas required is greater than the gas used.


This is a common pattern for meta-transaction where it is not possible for the smart contract to know whether that value X is an over estimation of the gas actually required.

With the introduction of [EIP-1706](https://eips.ethereum.org/EIPS/eip-1706) such problem will also arise for contract that simply use `SSTORE`, even existing contract, unless a similar mechanism (using `minimalGas` tracking) is used to ensure `eth_estimateGasRequirement` continue to work as expected 

This mechanism will also be useful for [EIP-1930](https://eips.ethereum.org/EIPS/eip-1930) that also implicitly check for `gasleft()`


## Backwards Compatibility

Backwards compatible for smart contract as it introduce a new opcode.

`eth_estimateGasRequirement` is a new RPC method so application can continue using `eth_estimateGas` for contract that use ```require(gasleft()>X)```

## Test Cases

TBD

## Implementation

TBD

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
