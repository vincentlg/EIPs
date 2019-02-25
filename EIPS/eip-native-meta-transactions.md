
  
  
---  
eip: native-meta-transactions  
title: Native Meta Transactions  
author: Ronan Sandford (@wighawag)  
discussions-to: XXXX  
status: Draft  
type: Standards Track  
category: ERC  
created: 2019-02-25  
requires: 712, 20, 777, 1271
---  
  
## Introduction  
Native Meta transactions (a term first coined by Austin Thomas Griffith here : [https://medium.com/gitcoin/native-meta-transactions-e509d91a8482](https://medium.com/gitcoin/native-meta-transactions-e509d91a8482)) allows users that simply own [ERC20](https://eips.ethereum.org/EIPS/eip-20) or [ERC777](https://eips.ethereum.org/EIPS/eip-777) ([ERC1155](https://eips.ethereum.org/EIPS/eip-1155) could be added too) tokens to perform operations on the ethereum network without needing to own ether themselves by letting a third party, the relayer, the responsibility to execute a transaction on the ethereum network carrying the desired operations (the so called meta transaction) in exchange of a token reward to cover the relayer's gas cost. They differ from traditional meta transactions ([https://medium.com/uport/making-uport-smart-contracts-smarter-part-3-fixing-user-experience-with-meta-transactions-105209ed43e0](https://medium.com/uport/making-uport-smart-contracts-smarter-part-3-fixing-user-experience-with-meta-transactions-105209ed43e0)) in that they only require support from the token contract itself and do not need the user wallet to be contract based.  
  
The proposal here define the message format and token contract interface required for **web3 wallets and browsers** to provide a meaningful meta-transaction display when users are requested to approve. This could even allow wallets/browsers to not bother users by displaying an ether balance when such users do not have any ether or when the application being used does not require it.  
  
It does not dictate how such signed message get injected on the network except for the meaning of each message parameters and their security requirements. More precisely, it does not dictate the ABI signature for the token smart contract function executing the meta-transaction (the one signed and broadcast by the relayer).  This is left as work for another standard.
  
Nevertheless due to nature of ERC20 this proposal also need to define how smart contract recipient of such ERC20 based meta transaction need to behave when being called. In particular it specify how the ```from``` field is to be verified. 
  
On the other hand, for ERC777 based meta transaction, no modification is required for the receiver and thus, any ERC777 recipient will support such native meta transaction through the use of the ```tokensReceived``` hook (see [ERC777](https://eips.ethereum.org/EIPS/eip-777)).  
  
  
## Why native meta transactions?  
  
It is common today for users to own ERC20 tokens without owning any ether. More and more applications reward their users without requiring prior on-chain interactions. It is thus possible for users to have been given tokens without them ever owning ether.  With native meta transactions and a willing relayer (the company behind the token for example), such users can now interact with ethereum. 
  
Without meta transactions, it would be impossible for them to interact with the ethereum network when required. Indeed, unless they have ether they can't interact with the ethereum network, requiring them to go through a difficult and costly process to acquire it. They can't even exchange their token for ether without having ether to pay the gas associated with such transaction.  
  
While normal meta-transactions are possible using smart contract based wallet (like gnosis safe: https://safe.gnosis.io), there are currently more users with EOA (Externally Owned Account) based wallet and this might be the case for a while as there is an inherent cost to smart contract wallet. Native meta transactions also allow new tokens to be used without requiring generic relayer support on the part of the smart contract wallet. They thus offer native support for meta-transactions without any requirement from the users except to have a private key and enough tokens to pay for the gas.  
  
As mentioned, Native Meta Transactions are great for applications whose users do not necessarily own or even know about ether. This is also useful for applications that want to distribute their tokens to new non-crypto users. Indeed, with such meta-transactions they can then start interacting in ethereum without having to think about ether. Application with the support of web3 wallet and browsers can then provide a less confusing experience where users can simply operate on one currency, at least until their horizon expands.  
  
## Specification 
The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).
 
For operability and the ability for wallet to display a more meaningful UI the following need to be defined:  
  
1) a signed message format that include all the required information for performing a meta transaction and protecting the user  
2) a public getter on the token smart Contract for the current nonce  
  
For ERC20 based meta transaction the receiver also need  
3) a way to verifiy the meta transaction signer.  
  
### Message format  
  
The proposal is using a message format based on [ERC712](https://eips.ethereum.org/EIPS/eip-712) so that wallet that support ERC712 but do not support the proposal described here can still offer an approval display showing all the information albeit in a less than ideal presentation.  
  
We separate ERC20 and ERC777 message type since a token could implement both of these standard and the behaviour/implementation of meta-transaction would differ between the two (since ERC777 would rely on the data parameter of send and the ```tokensReceived``` hook while ERC20 would relies on temporary approval and ```from```parameter verification). The difference between the 2 would result in the transaction having different meaning. If the relayer were to chose which to execute, they could result in outcome not desired by the signer.  
  
For simplicity though, they MUST share the same nonce, allowing us to keep using a singular getter for it.  
  
Here is the ERC712 message format for ERC20 based meta transactions :  
```typeHash = keccak256("ERC20MetaTransaction(address from,address to,uint256 amount,bytes data,uint256 nonce,uint256 gasPrice,uint256 gasLimit,uint256 tokenGasPrice,address relayer)");```  
  
And here is the one for ERC777 based meta transaction  
  
```typeHash = keccak256("ERC777MetaTransaction(address from,address to,uint256 amount,bytes data,uint256 nonce,uint256 gasPrice,uint256 gasLimit,uint256 tokenGasPrice,address relayer)");```  
  
The meaning of each field is as follow:  
  
* **from**: the account from which the meta-transaction is executed for and from which the token will be deduced. It MUST be either equal to the resulting message signer or have been given execution rights to the signer (using for example [ERC1271](https://eips.ethereum.org/EIPS/eip-1271))  
* **to**: the target that will receive the token amount (if any, see below). If it is a contract's address the data(see below) passed will be executed on it. 
* **amount**: the amount in token to be sent to ```to``` (for ERC20 it'll take the form of an temporary approval)  
* **data**: the bytes to be executed at ```to``` (if empty, a simple transfer will be executed)  
* **nonce**: a nonce so that messages cannot be replayed. It also ensure the order of transactions. for it to be a valid message the nonce MUST be equal to the current nonce + 1
* **gasprice**: the gas price in ether to be used for the actual transaction (keeping the user in control) this also prevent attack from a malicious relayer where the logic of the call relies on gasprice (for whatever reason)  
* **gasLimit**: the exact amount of gas to be used to execute the meta-transaction. This field also prevents attacks that could happen if the logic of the call relies on gas provided for whatever reason. (note though that the total cost of the transaction executing the meta-transaction will obviously have a higher cost)
* **tokenGasPrice** the gasPrice set in token, (the relayer will decide to accept meta-transaction or not depending on the value of such). In other word the exchange rate used is defined by ```gasPrice / tokengasPrice``` This separation allow users to set the ether gasPrice to be used, deciding on the speed at which it desire the transaction to be mined
* **relayer** (optionally set to zero) used to protect the chosen relayer from front running attacks where the intended executor would lose its gas by someone inserting the transaction before. Letting it to be zero (acts as a wildcard) could still be worthwhile for situations where the users would benefit of having different relayers. This is out of scope of this proposal to define how such relayers network would work together. 
  
### nonce 
In order for the wallet or application to request a valid meta transaction it needs to be able to know the current nonce

The token contract MUST implement a getter for the current nonce as follow:  
  
```function meta_nonce(address from) external returns (uint256);```  
  
Nonces work similarly to normal ethereum transactions: a transaction can only be executed if it matches the last nonce + 1, and once a transaction has occurred, the `lastNonce` will be updated to the current one. This prevents transactions to be executed out of order or more than once.  
  
### execute transaction and receiver verification  
  
When executing the meta-transaction the contract must then verify the signature and if the nonce matches as specified above.  
  
Then for the case of ERC20 the implementation MUST temporarily change the approval before calling the destination to reflect the amount of token being sent. It MUST NOT emit Approval events for the temporary change of allowance. It also need to ensure that the first parameter of the call being made is equal to ```from```. The receiver will thus be able to accept calls from the token by knowing that the first parameter is indeed the ```from``` and not some arbitrary address. This means only such receiver will be able to accept such meta transactions securely. 
 
In order to do it, the receiver simply check if msg.sender is the token contract itself and if so can assume that first parameter is equal to the ```from``` specified as part of the meta transaction message,  

Remember, such ERC20 meta transaction receiver need to have as first parameters the "sender" whose token will be deduced.

for example:
```
function receiveERC20Meta(address sender, uint256 value) external {
  require(sender ==  msg.sender ||  msg.sender ==  address(token), "sender has to be the actual sender or the erc20 token contract itself");
  require(value == price, "value != price");
  token.transferFrom(sender, address(this), value);
  balance += value;
  ...
}
``` 
  
For ERC777 tokens, there is no need of such restriction and the receiver will simply use the standard ```tokensReceived(address, address, address, uint256, bytes, bytes)``` hook.   As such, ERC777 meta transactions are able to call any contract that can receive ERC777 tokens via ```tokensReceived``` hook.
  
 ### Balance checks 
To protect from malicious user the executor also need to ensure the user (```from```) has enough balance. While it is technically possible for the user to withdraw token just in time (between the meta-tx is send and mined), it is unlikely to happen since it is unlikely to benefit the user unless it wishes to cancel the last minute. They could achieve a similar feat anyway by publishing a different signed message with the same nonce (albeit at a higher gas cost than a simple transfer). For erc777 they can also conditionally block refund via tokensTosend callback.  This is a risk that need to be taken by the relayer.
  
### Gas accounting and refund  
The ```gasLimit``` set as part of the message represent the gas passed to the contract call made (the meta transaction). This ensure the signer that its call will be executed as intended with as many gas as it asked for. This means though that the total gas cost of the realyer's transaction will be higher than that.

The token contract SHOULD calculate the refund based on such total. Otherwise the relayer would have hard time to know if the amount of token received would cover the execution cost. 

As for the user, this means that the total token gas cost can only be know if the extra gas required to execute the meta transaction is known. 

This can be achieved by executing the transaction the relayer would execute. It is not part of this standard.
    
### Gas estimate  
In order to estimate the gasLimit to use for the meta transaction, the meta transaction call data can be used. The behaviour will be identical. As such there is no need to expose an estimateGas function for that.

As for the extra gas required by the relayer, the whole meta transaction call can be estimated as usual. 
  
  
## Example Implementation  
  
see https://github.com/pixowl/thesandbox-contracts/blob/master/src/Sand/erc20/ERC20MetaTxExtension.sol
  
## wallet / browser  
  
Web3 wallet and browsers MUST provide a meaningful interface when such signed message is being requested. They could for example show a similar UI to traditional ether transaction except it should clearly state the token being used as well as the other parameters. For the data field, they should also be able to use function signature registry to display the function being called and the arguments.  
  
  
## Copyright  
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).