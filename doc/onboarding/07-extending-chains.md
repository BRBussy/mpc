# §7 — Extending to a new chain (light — pointers only)

[← §6 Local bootstrap](06-local-bootstrap.md) · [Index](README.md) · Next: [§8 Experiments →](08-experiments.md)

---

When you come back to this, you'll want three things.

## ① A chain-specific contract that emits a `SignatureRequested`-shaped event

Semantic fields `(sender, payload[32], key_version, deposit, chain_id, path, algo, dest, params)`. Reference implementations:

- [ChainSignatures.sol](../../chain-signatures/contract-eth/contracts/ChainSignatures.sol) — a minimal plumbing contract (196 lines total; that's the whole bar).
- [contract-sol/src/lib.rs](../../chain-signatures/contract-sol/src/lib.rs) — an Anchor version for Solana.

## ② A new indexer in [chain-signatures/node/src/](../../chain-signatures/node/src/)

Match the shape of the existing ones. The seam is the `ChainEvent` enum in [stream/mod.rs](../../chain-signatures/node/src/stream/mod.rs). Your indexer's job:

1. Detect events on the source chain.
2. Validate (the four checks in [§4.3](04-event-flow.md#checkpoint-2--indexer-side-every-mpc-node)).
3. Derive `epsilon` via a new `derive_epsilon_<chain>` in the `mpc_crypto::kdf` module.
4. Produce an `IndexedSignRequest`.
5. Send it into the shared channel.

The Hydration indexer ([indexer_hydration.rs](../../chain-signatures/node/src/indexer_hydration.rs)) is the most complete, well-annotated reference because it also handles proofs.

## ③ A `try_publish_<chain>` function in [rpc.rs](../../chain-signatures/node/src/rpc.rs)

Takes a validated `PublishAction` and calls `respond()` on the new chain.

## Wiring points (search these when you're ready)

- `Chain` enum in [chain-signatures/primitives/](../../chain-signatures/primitives/) `[UNVERIFIED]` — add a variant.
- `derive_epsilon_*` in [chain-signatures/crypto/](../../chain-signatures/crypto/) `[UNVERIFIED]` — add the derivation.
- CLI flags: add an `*Args` group in a new `indexer_<chain>.rs`, then `clap(flatten)` it in [cli.rs](../../chain-signatures/node/src/cli.rs).
- Spawn in the node's main loop where the existing indexers are started.

## Minimum viable "mock" extension for experimenting

Stand up a mock that emits a `SignatureRequested`-shaped message on any transport (a local HTTP-pushed log, a file watch, whatever), wire a new indexer that reads from it, and verify the signature comes back on-chain via an existing chain's `respond()`. You don't need to add a new `respond` target to prove the indexer path end-to-end.

---

[← §6 Local bootstrap](06-local-bootstrap.md) · [Index](README.md) · Next: [§8 Experiments →](08-experiments.md)
