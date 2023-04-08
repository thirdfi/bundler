# Bundler


## Introduction

Bundler is a special class of actor that act as a relayer between account abstraction transaction (called `userOperation`) and actual transaction on-chain through EntryPoint contract. Bundler implement full EIP-4337 RPC calls by, fetch `userOperation` from alternate mempool, do validation and simulation before send it to EntryPoint contract, and return any response back through RPC.

Further read:

https://www.blocknative.com/blog/account-abstraction-erc-4337-guide#7

https://hackmd.io/@Vid201/aa-bundler-rust#ERC-4337-Account-Abstraction---Bundler-implementation-in-Rust


## Flow

![bundler](https://user-images.githubusercontent.com/48756752/230726043-d31b2c1a-6288-4773-b1a8-316102947f8f.png)

### RPC

Bundler receive methods from endpoint and return any result.

  - `eth_sendUserOperation`

  - `eth_getUserOperationByHash`

  - `eth_getUserOperationReceipt`

  - `eth_chainId`

  - `eth_supportedEntryPoints`

  - `eth_estimateUserOperationGas`

  - `web3_clientVersion`: return `/erc4337RuntimeVersion` (version of `@account-abstraction/utils` SDK) and `/unsafe` if bundler run with unsafe mode

For parameters required and return value, refer [EIP4337 article](https://eips.ethereum.org/EIPS/eip-4337#rpc-methods-eth-namespace).

### Validation

- validate EntryPoint input

- validate `userOperation` input parameters

- simulate validate `userOperation` by Geth traceCall (or by EntryPoint if input `--unsafe` when run bundler)

  - check if account/paymaster use forbidden opcodes if validate by traceCall

  - check if paymaster staked

  - (TODO: other check)

- validate if the signature is correct

- validate if `userOperation` is expired / going to expire soon

- add into mempool after all validation passed

### Auto-bundler

Auto-bundler create bundle for all `userOperation` from mempool every `autoBundleInterval` seconds set in `bundler.config.json`

## Create bundle

- sort `userOperation`s from mempool by highest priority fee

- skip (or remove if banned) `userOperation`s that use throttled factory/paymaster

- skip `userOperation`s that execute by same AA account (no duplicate AA account execution in one bundle, multiple execution within same AA account use batch execution in single `userOperation`)

- do validation check again (in case `userOperation` been skipped in first attempt bundle), and remove from mempool if failed

- check if Paymaster balance is enough to pay all `userOperation`s in the bundle

## Send bundle

- bundler signer sign and send bundle to EntryPoint

- if bundle failed on-chain, sender/paymaster/factory that cause it will be recorded in reputation system


## Usage:

1. Run `yarn && yarn preprocess`

2. Change parameters in `./packages/bundler/localconfig/bundler.config.json`

  - `network`: RPC to connect i.e. Arbitrum RPC

  - `beneficiary`: wallet to receive gas reimbursement

  - `minBalance`: minimum balance in bundler signer, if lower than this value, beneficiary will auto assign to bundler signer

  - `mnemonic`: mnemonic of bundler signer to execute AA transaction

  - `minStake`: Paymaster minimum stake in EntryPoint

  - `minUnstakeDelay`: minimum time Paymaster stake locked in EntryPoint before able to unstake in seconds, Paymaster will be rejected if lower than this value

  - `autoBundleInterval`: interval seconds to execute all `userOperation`s in mempool

  - `autoBundleMempoolSize`: maximum amount of `userOperation` per bundle

3. Fund bundler signer at least `minBalance` ETH

4. Run bundler with `yarn run bundler`, or `yarn run bundler --unsafe` if connect to RPC that doesn't support Geth trace call `debug_traceCall`. (i.e. Alchemy free RPC)

Debug: run debug mode with `DEBUG=* yarn run bundler`


## Other technical details

TODO: everything else here

### Forbidden opcodes

Why forbidden opcodes?
https://eips.ethereum.org/EIPS/eip-4337#forbidden-opcodes

### Signature aggregator

This bundler currently not support signature aggregator

## Mempool

How mempool work in bundler

## Reputation

How reputation work in bundler

## Safe and unsafe bundler

Different between safe and unsafe bundler

How preVerificationGas calculate
