# seda-hub-relay

Deliver [SEDA](https://www.seda.xyz/) oracle results to CosmWasm contracts on the Cosmos Hub. The only trust assumption is the Hub's existing IBC light client of `seda-1`.

A relayer posts a data request on SEDA, then submits the resulting `DataResult` to the Hub with its ICS23 proof. A relay contract on the Hub checks the proof against the light client with `/ibc.core.client.v1.Query/VerifyMembership`, then hands the result to the consumer contract. Nothing SEDA-specific is deployed on the Hub, and relaying is permissionless.

This builds on the light client verification primitive from [interchain-event](https://github.com/Victor118/interchain-event). On the Hub, it requires [cosmos/gaia#4023](https://github.com/cosmos/gaia/issues/4023), which adds `VerifyMembership` to Gaia's wasm gRPC accept list.

## Contents

| Path | What |
|---|---|
| [`seda-relay-contract-spec.md`](seda-relay-contract-spec.md) | Specification of the relay contract, the relayer flow, and a push price feed variant |
| [`demos/seda-verify`](demos/seda-verify) | Off-chain, read-only check against mainnet: proves a real seda-1 `DataResult` against the Hub's light client |

## Status

- Verification checked off-chain against mainnet (`demos/seda-verify`).
- Gaia change implemented and tested, waiting on [cosmos/gaia#4023](https://github.com/cosmos/gaia/issues/4023).
- Next: oracle programs with reproducible builds (starting with Polymarket outcomes), then the relay contract and the relayer.
