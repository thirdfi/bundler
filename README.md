# Bundler


## Introduction

TODO: introduction of bundler, with some article reference


## Flow

TODO: flow chart + explanation on flow chart


## Usage:

1. Run `yarn && yarn preprocess`

2. Change parameters in ./packages/bundler/localconfig/bundler.config.json

  - `network`: RPC to connect i.e. Arbitrum RPC

  - `beneficiary`: wallet to receive gas reimbursement

  - `minBalance`: minimum balance in bundler signer, if lower than this value, beneficiary will auto assign to bundler signer

  - `mnemonic`: mnemonic of bundler signer to execute AA transaction

  - `minStake`: minimum stake in Paymaster

  - `minUnstakeDelay`: minimum time stake locked in EntryPoint required before able to unstake

3. Fund bundler signer at least `minBalance` ETH

4. Run bundler with `yarn run bundler`, or `yarn run bundler --unsafe` if connect to RPC that doesn't support Geth trace call `debug_traceCall`. (i.e. Alchemy free RPC)


## Other technical details

TODO: everything else here