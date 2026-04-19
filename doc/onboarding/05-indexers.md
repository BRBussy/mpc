# §5 — Indexing across chains (deep)

[← §4 Event flow](04-event-flow.md) · [Index](README.md) · Next: [§6 Local bootstrap →](06-local-bootstrap.md)

---

## 5.1 Shared shape

Every indexer implements a loop that produces a stream of `ChainEvent`s and pushes them through the central dispatcher at [chain-signatures/node/src/stream/mod.rs](../../chain-signatures/node/src/stream/mod.rs). `ChainEvent` variants include `SignRequest(IndexedSignRequest)`, `Respond(SignatureRespondedEvent)`, `RespondBidirectional(...)`, `Block(u64)`, `CatchupCompleted`, `ExecutionConfirmed { ... }` `[UNVERIFIED]` (exact enum lines — see `stream/mod.rs` lines 19-100).

Every indexer normalizes to the same `IndexedSignRequest` type at [chain-signatures/node/src/protocol/signature.rs:46-75](../../chain-signatures/node/src/protocol/signature.rs#L46-L75) `[UNVERIFIED]` exact lines, with fields `{ id: SignId, args: SignArgs { entropy, epsilon, payload, path, key_version }, chain: Chain, unix_timestamp_indexed: u64, kind: SignKind }`.

All indexers are **individually optional**. The node starts whichever are configured; missing config silently disables the indexer. For Ethereum/Solana/Hydration this is visible in [cli.rs](../../chain-signatures/node/src/cli.rs) where the `*Args` groups are `clap(flatten)`'d in and each indexer's `::new()` returns `Result`/`Option` that falls through on missing config.

## 5.2 Ethereum

- **Source files**: [chain-signatures/node/src/indexer_eth/mod.rs](../../chain-signatures/node/src/indexer_eth/mod.rs), plus sibling backends `indexer_eth_direct_rpc.rs` and `indexer_eth_helios.rs`.
- **Detection**: polls latest block every 500ms; replays the `last_processed_block → current` range on each tick `[UNVERIFIED]`. Also subscribes to new-block notifications when available.
- **Event matching**: uses alloy-generated `SignatureRequested::SIGNATURE_HASH` (and similarly for `SignatureResponded`) as the log-topic filter, matched against `log.address() == contract_address`.
- **Finality**: see [§4.4](04-event-flow.md#44-finality--reorg-behavior). Enforces `finalized` block tag unless `optimistic_requests=true`.
- **Reorg detection**: block hash re-fetch after finality; mismatch logged and skipped `[UNVERIFIED]`.
- **Max catchup window**: 8,191 blocks when using Helios (light-client limit) `[UNVERIFIED]`.
- **Backends**: direct RPC (fast, trusts RPC) vs. Helios (light client, verifies consensus). Controlled by `--eth-light-client`.
- **Config** (from [cli.rs](../../chain-signatures/node/src/cli.rs) and [integration-tests/src/main.rs:26-42](../../integration-tests/src/main.rs#L26-L42)):

| Flag | Env var | Default |
|---|---|---|
| `--eth-account-sk` | `MPC_ETH_ACCOUNT_SK` | `5de4111afa1a4b94...` (dev default) |
| `--eth-contract-address` | `MPC_ETH_CONTRACT_ADDRESS` | `e7f1725E7734CE288F8367e1Bb143E90bb3F0512` (dev default) |
| `--eth-consensus-rpc-http-url` | — | `http://localhost:8545` |
| `--eth-execution-rpc-http-url` | — | `http://localhost:8545` |
| `--eth-network` | — | `sepolia` |
| `--eth-helios-data-path` | — | `/tmp/data` |
| `--eth-refresh-finalized-interval` | — | `10000` (ms) |
| `--eth-light-client` | — | `false` |
| `--eth-optimistic-requests` | — | `false` |

## 5.3 Solana

- **Source file**: [chain-signatures/node/src/indexer_sol.rs](../../chain-signatures/node/src/indexer_sol.rs).
- **Detection**: Solana PubSub WebSocket `logs_subscribe` with `RpcTransactionLogsFilter::Mentions([program_id])`, commitment `confirmed` ([indexer_sol.rs ~lines 340-395 range](../../chain-signatures/node/src/indexer_sol.rs)). No HTTP polling for sign events.
- **Event matching**: Anchor discriminators (8-byte prefixes) match `SignatureRequestedEvent::DISCRIMINATOR`, `SignBidirectionalEvent::DISCRIMINATOR`, `SignatureRespondedEvent::DISCRIMINATOR`, `RespondBidirectionalEvent::DISCRIMINATOR`. Events are emitted via Anchor's `emit_cpi!` ([contract-sol/src/lib.rs:33](../../chain-signatures/contract-sol/src/lib.rs#L33)) — parsed out of the transaction's inner instructions.
- **Dedup**: 30s TTL cache on tx signatures avoids refetching the same transaction `[UNVERIFIED]`.
- **Reconnect**: stalls beyond 60s trigger reconnect `[UNVERIFIED]`.
- **Entropy source**: first 32 bytes of transaction signature `[UNVERIFIED]`.
- **Config**:

| Flag / env | Notes |
|---|---|
| `--sol-account-sk` / `MPC_SOL_ACCOUNT_SK` | Required — node's Solana keypair for responding. |
| `--sol-rpc-http-url` / `MPC_SOL_RPC_HTTP_URL` | Required. |
| `--sol-rpc-ws-url` / `MPC_SOL_RPC_WS_URL` | Required — indexer is WebSocket-first. |
| `--sol-program-address` / `MPC_SOL_PROGRAM_ADDRESS` | Required — program id to monitor (also used by `respond()`). |

## 5.4 Hydration (Substrate)

- **Source file**: [chain-signatures/node/src/indexer_hydration.rs](../../chain-signatures/node/src/indexer_hydration.rs).
- **Detection**: Subxt `subscribe_finalized()` block stream (no reorgs possible because finalized-only).
- **Event matching**: pallet name `Signet` + variant name `SignatureRequested` / `SignatureResponded` / `SignBidirectionalRequested` / `RespondBidirectionalEvent` `[UNVERIFIED]` line refs.
- **Proof verification**: every event is re-derived via `state_get_read_proof` + `read_proof_check` against the block's Blake2-256 state root — unlike the other indexers, this one **requires** a Merkle proof. Good pattern for adding new Substrate chains.
- **Entropy source**: `blake2_256(event_bytes)`.
- **Config**:

| Flag / env | Notes |
|---|---|
| `--hydration-rpc-ws-url` / `MPC_HYDRATION_RPC_WS_URL` | Required. |
| `--hydration-signer-uri` / `MPC_HYDRATION_SIGNER_URI` | Required — signer URI (used for the `respond` path). |

## 5.5 NEAR

- **Source file**: [chain-signatures/node/src/indexer.rs](../../chain-signatures/node/src/indexer.rs).
- **Detection**: polls the NEAR contract's `pending_requests_data()` view call every 750ms — **not** log scraping. This is because the NEAR contract stores pending requests in on-contract state (see [contract/src/lib.rs:180](../../chain-signatures/contract/src/lib.rs#L180) `self.lock_request(...)`). The state is authoritative; logs are not needed.
- **Entropy**: `sha256(format!("{:?}", sign_id))` — deterministic but not blockchain-grounded `[UNVERIFIED]`.
- **Config**: `--mpc-contract-id` / `MPC_CONTRACT_ID` (default `dev.sig-net.testnet`), `--near-rpc` / `MPC_NEAR_RPC` (default `https://rpc.testnet.near.org`). See [cli.rs:37-45](../../chain-signatures/node/src/cli.rs#L37-L45).

## 5.6 Chain-specific quirks cheat-sheet

| Concern | Ethereum | Solana | Hydration | NEAR |
|---|---|---|---|---|
| Transport | Polling + subscriptions | WebSocket only | Subxt stream | HTTP polling of view fn |
| Finality waiting | Configurable | Confirmed commitment | Finalized only | Immediate |
| Event ID | keccak256 | keccak256 | keccak256 | `SignId::from_parts` |
| Can disable | Yes | Yes | Yes | Legacy only — NEAR is always needed (orchestration contract) |
| Max catchup | ~8191 blocks (Helios) | Unbounded | Unbounded | N/A (state, not log) |

---

[← §4 Event flow](04-event-flow.md) · [Index](README.md) · Next: [§6 Local bootstrap →](06-local-bootstrap.md)
