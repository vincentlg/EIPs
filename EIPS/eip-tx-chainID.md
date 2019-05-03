---
eip: tx-chainID
title: flexible chainID verification for transaction
author: Ronan Sandford (@wighawag)
discussions-to: 
category: Core
type: Standards Track
status: Draft
created: 2019-04-28
---

## Abstract
This EIP specify a new behavior for transaction's chainID checking. It allows transaction signed for a previous chainID to be valid after an hardfork that changed the latest chainID. In other words, this allow hardfork that changed the chainID to be handled without downtime.

## Motivation
[EIP-155](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-155.md) proposes to use the chain ID to prevent replay attacks between different chains. Unfortunately, when a contentious hard fork happen, one of the fork will need to update its chainID. Transaction that were recently broadcasted will be invalid on that chain. This is undesirable as it provide a downtime for the fork forced to update its chainID.

Ideally, we want to preserve a freedom to fork so that whatever minority choose to fork, they should not be impacted either by downtime or replays.

## Specification

Add a blockNumber as value to hash as part of the transaction. That blockNumber represent the time at which the transaction was signed and it used to decide if it was meant for a particular fork

Transaction with a blockNumber specified are accepted if the chainID specified is a valid chainID at that block. 


## Rationale

When a contentious fork F1 happen,if nothing is done, transaction can be replayed on both fork indefinitely.

One side of the fork can decide to update its chainID via another fork F2 so new transaction targeted on that chain can't be replayed on the other one.

Unfortunately, this require user to be careful at the chainID fork transition F2 since if the mechanism in place is simply to reject transaction with different chainID, then they will have to wait the fork to happen before emitting the transaction.

The solution for "no down time" is to continue accepting transaction with older chainIDs.

Unfortunately, this means that users of the side of fork F1 that did not updated its chainID, will have their transaction replayed.

The solution is thus to add a blockNumnber specified as part of the transaction, so that old chainID are only accepted for transaction signed for a blockNumber where that chainID is valid.

## Backwards Compatibility
This EIP is introduced an extra parameter to each transaction ```blockNumber``` that represent the time arround at which the transaction was signed. As mentioned above, it allows transaction signed for a majority chain to not be replayed on a minority chain.

But to allow backward compatibility for transaction in the pool when the hardfork that implement this EIP happen, we can make such field optional. In which case, transaction specified with any past or present chainID will continue to be valid. The blockNumber 

## References
This was previously suggested as part of [EIP-1344](https://ethereum-magicians.org/t/eip-1344-add-chain-id-opcode/1131/89).

And this mimick the behavior specified as part of [EIP-1965](https://github.com/ethereum/EIPs/pull/1965) for off-chain message replay protection.

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
