# §3 — Key files tour (ranked by integrator relevance)

[← §2 Architecture](02-architecture.md) · [Index](README.md) · Next: [§4 Event flow →](04-event-flow.md)

---

| # | File | What it is & why it matters for you |
|---|---|---|
| 1 | [chain-signatures/contract-eth/contracts/ChainSignatures.sol](../../chain-signatures/contract-eth/contracts/ChainSignatures.sol) | The entire Ethereum surface. `sign()`/`respond()`/events. Read the whole file — it's 196 lines. |
| 2 | [chain-signatures/contract-sol/src/lib.rs](../../chain-signatures/contract-sol/src/lib.rs) | Entire Solana surface in one Anchor file. Four entry points: `sign`, `respond`, `sign_bidirectional`, `respond_bidirectional`. |
| 3 | [chain-signatures/contract/src/lib.rs](../../chain-signatures/contract/src/lib.rs) (lines 128-243) | NEAR `sign()`/`respond()` + derivation helpers. NEAR uses **Promise yield/resume** — integrators get the signature as a normal return value. |
| 4 | [doc/ACCOUNT_DERIVATION.md](../ACCOUNT_DERIVATION.md) | Derivation path spec. Essential for deriving a user's foreign-chain address off-chain. |
| 5 | [chain-signatures/node/src/indexer_sol.rs](../../chain-signatures/node/src/indexer_sol.rs) | Solana indexer. Shows exactly what the nodes look for in the tx log stream. Read the `SignatureEvent` impl and validation around [indexer_sol.rs:164-207](../../chain-signatures/node/src/indexer_sol.rs#L164-L207) to understand pre-participation validation. |
| 6 | [chain-signatures/node/src/indexer_eth/mod.rs](../../chain-signatures/node/src/indexer_eth/mod.rs) | Ethereum indexer. Has two backends: direct RPC and Helios light client (see siblings `indexer_eth_direct_rpc.rs`, `indexer_eth_helios.rs`). |
| 7 | [chain-signatures/node/src/indexer.rs](../../chain-signatures/node/src/indexer.rs) | NEAR indexer (integration-test flavour). Small; uses contract-state polling. Production NEAR indexing is actually done via the NEAR contract's internal pending-requests list rather than log scraping. |
| 8 | [chain-signatures/node/src/indexer_hydration.rs](../../chain-signatures/node/src/indexer_hydration.rs) | Hydration/Substrate indexer, including Merkle-proof verification. Nice reference for "how to add a new chain". |
| 9 | [chain-signatures/node/src/stream/mod.rs](../../chain-signatures/node/src/stream/mod.rs) + [stream/ops.rs](../../chain-signatures/node/src/stream/ops.rs) | The unified `ChainEvent` enum and the loop that funnels every chain's events into the sign queue. The seam between "chain-specific" and "chain-agnostic". |
| 10 | [chain-signatures/node/src/rpc.rs](../../chain-signatures/node/src/rpc.rs) | Where the node calls `respond()` back on-chain. Per-chain `try_publish_*` functions. Signature-verification gate before publishing. |
| 11 | [chain-signatures/node/src/cli.rs](../../chain-signatures/node/src/cli.rs) | All node CLI flags + env vars. When something isn't working, start here. |
| 12 | [chain-signatures/node/src/web/mod.rs](../../chain-signatures/node/src/web/mod.rs) | HTTP routes the node exposes. Integrators: these are **not** for requesting signatures — only health/state/metrics + internal peer RPCs. |
| 13 | [chain-signatures/node/src/protocol/signature.rs](../../chain-signatures/node/src/protocol/signature.rs) | Where `IndexedSignRequest` is defined (lines 46-75) and where the cait-sith sign protocol is driven. |
| 14 | [integration-tests/tests/cases/solana.rs](../../integration-tests/tests/cases/solana.rs) | The cleanest, shortest end-to-end example in the repo — `test_solana_signature_basic` is 42 lines. Start your tracing here. |
| 15 | [integration-tests/src/actions/sign.rs](../../integration-tests/src/actions/sign.rs) | Helper builders the tests use (`cluster.sign().solana().payload(...)...`). Good model for an SDK API. |
| 16 | [integration-tests/src/containers.rs](../../integration-tests/src/containers.rs) | How the integration harness builds the raw instructions/transactions for each chain. Particularly useful on Solana where there's no equivalent Sol SDK yet. |
| 17 | [integration-tests/src/cluster/spawner.rs](../../integration-tests/src/cluster/spawner.rs) | How the test cluster is composed — sandbox, redis, anvil, solana-test-validator, n nodes. Read this if local bootstrap misbehaves. |
| 18 | [integration-tests/src/mpc_fixture/builder.rs](../../integration-tests/src/mpc_fixture/builder.rs) | Fast in-process MPC fixture (no docker). Useful for SDK unit tests. |
| 19 | [chain-signatures/node/src/sign_bidirectional.rs](../../chain-signatures/node/src/sign_bidirectional.rs) + [respond_bidirectional.rs](../../chain-signatures/node/src/respond_bidirectional.rs) | The "cross-chain execute" flow: sign on chain A, execute the resulting tx on chain B. Solana + Hydration only today. |
| 20 | [chain-signatures/contract/EXAMPLE.md](../../chain-signatures/contract/EXAMPLE.md) | Generated examples of near-cli commands against the NEAR contract. Regenerate with `cargo run --bin integration-tests -- contract-commands` ([integration-tests/src/main.rs:127-132](../../integration-tests/src/main.rs#L127-L132)). |

---

[← §2 Architecture](02-architecture.md) · [Index](README.md) · Next: [§4 Event flow →](04-event-flow.md)
