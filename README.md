# Bundler


## Introduction

Bundler is a special class of actor that act as a relayer between account abstraction transaction (called `userOperation`) and actual transaction on-chain through EntryPoint contract. Bundler fetch `userOperation` from alternate mempool, do validation and simulation before send it to EntryPoint contract, and return any response back through RPC. This bundler implement EIP-4337 RPC calls without `debug_*` calls.

Further read:

https://www.blocknative.com/blog/account-abstraction-erc-4337-guide#7

https://hackmd.io/@Vid201/aa-bundler-rust#ERC-4337-Account-Abstraction---Bundler-implementation-in-Rust


## Flow

![bundler Medium](https://user-images.githubusercontent.com/48756752/230726201-6953f1f6-67a1-46ea-9dd6-0e24885069b2.png)

### RPC

Bundler receive RPC methods from endpoint and return any result. The methods including:

  - `eth_sendUserOperation`

  - `eth_getUserOperationByHash`

  - `eth_getUserOperationReceipt`

  - `eth_chainId`

  - `eth_supportedEntryPoints`

  - `eth_estimateUserOperationGas`

  - `web3_clientVersion`: return `aa-bundler` + `/erc4337RuntimeVersion` (version of `@account-abstraction/utils` SDK) + `/unsafe` if bundler run with unsafe mode

For parameter required and return value, refer [EIP4337 article](https://eips.ethereum.org/EIPS/eip-4337#rpc-methods-eth-namespace).

### Validation

- validate EntryPoint input

- validate `userOperation` input parameters

- simulate validate `userOperation` by Geth traceCall (or by EntryPoint if input `--unsafe` when run bundler)

  - check if `userOperation` is executable

  - check if account/paymaster use forbidden opcodes or storage if validate by traceCall 
  (refer [this](https://eips.ethereum.org/EIPS/eip-4337#specification-1) and [this](https://eips.ethereum.org/EIPS/eip-4337#forbidden-opcodes])

  - check if paymaster staked

- validate if signature is correct

- validate if `userOperation` is expired / going to expire soon

Add into mempool after all validation passed

### Auto-bundler

Auto-bundler create bundle for at most `autoBundleMempoolSize` amount of `userOperation` from mempool every `autoBundleInterval` seconds set in `bundler.config.json`

### Create bundle

- sort `userOperation`s from mempool descending by highest priority fee

- skip (or remove if banned) `userOperation`s that use throttled factory/paymaster

- skip `userOperation`s that execute by same AA account (no duplicate AA account execution in one bundle, multiple execution within same AA account can use batch execution in single `userOperation`)

- do validation check for `userOperation` again (in case `userOperation` been skipped in first attempt bundle), and remove from mempool if failed

- check if Paymaster balance is enough to pay all `userOperation`s that rely on Paymaster in the bundle

### Send bundle

Bundler signer sign and send bundle to EntryPoint. If bundle failed on-chain, sender/paymaster/factory that cause it will be recorded in reputation system


## Usage:

1. Run `yarn && yarn preprocess`

2. Change parameters in `./packages/bundler/localconfig/bundler.config.json`

  - `network`: RPC to connect i.e. Arbitrum RPC URL

  - `beneficiary`: wallet to receive gas reimbursement

  - `minBalance`: minimum balance in bundler signer, if lower than this value, `beneficiary` will auto assign to bundler signer

  - `mnemonic`: mnemonic of bundler signer to execute AA transaction

  - `maxBundleGas`: gas limit per bundle

  - `minStake`: Paymaster minimum stake in EntryPoint

  - `minUnstakeDelay`: minimum seconds Paymaster stake locked in EntryPoint before able to unstake, Paymaster will be rejected if lower than this value

  - `autoBundleInterval`: interval seconds to bundle at most `autoBundleMempoolSize` amount of `userOperation`s in mempool

  - `autoBundleMempoolSize`: maximum amount of `userOperation` per bundle

3. Fund bundler signer at least `minBalance` ETH

4. Run bundler with `yarn run bundler`, or `yarn run bundler --unsafe` if connect to RPC that doesn't support Geth trace call `debug_traceCall`. (i.e. Alchemy free RPC)

Debug: run debug mode with `DEBUG=* yarn run bundler`


## Advanced technical details

### Forbidden opcodes

Why forbidden opcodes?

> These opcodes are forbidden because their outputs may differ between simulation and execution, so simulation of calls using these opcodes does not reliably tell what would happen if these calls are later done on-chain.

Refer full paragraph [here](https://eips.ethereum.org/EIPS/eip-4337#forbidden-opcodes)

### Signature aggregator

This bundler currently not support signature aggregator

### Mempool

User send `userOperation` through bundler RPC. Bundler run some validation on `userOperation`, and only add it into mempool if the validation check passed. Then, bundler attempt to bundle all `userOperation`s that sit in mempool follow these criteria: every `autoBundleInterval` seconds or total `userOperation`s in mempool exceed `autoBundleMempoolSize`. When criteria met, bundler first remove previous `userOperation`s that had executed successfully on-chain, and record into reputation system (success + 1). Then follow by create a new bundle. In the process of bundle creation, if the `userOperation` use banned Factory or Paymaster contract, `userOperation` will be removed directly from mempool. Throttled Factory or Paymaster will not be included in bundle and keep in mempool until their reputation become better (as time goes by). `userOperation` with same AA wallet in bundle will also been skipped (which can be included in next bundle). After that, same validation check on `userOperation` again before finally add into bundler, and remove the `userOperation` from mempool if check failed. Other check including total gas used of `userOperation` in bundle cannot exceed `maxBundleGas` (skip subsequence `userOperation` if exceed), and Paymaster ETH balance that able to sponsor all `userOperation`s that rely on it in same bundle (skip subsequence `userOperation`s if exceed).

### Reputation

*TODO: explain in simple term*

Refer full explanation [here](https://eips.ethereum.org/EIPS/eip-4337#reputation-scoring-and-throttlingbanning-for-global-entities)

### Safe and unsafe bundler

*TODO: different between safe and unsafe bundler*

Why need safe bundler? Some explanation [here](https://eips.ethereum.org/EIPS/eip-4337#rationale)

TL:DR, to prevent success on validation but failed on execution.