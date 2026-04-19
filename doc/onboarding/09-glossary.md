# §9 — Glossary

[← §8 Experiments](08-experiments.md) · [Index](README.md)

---

| Term | Meaning in this codebase |
|---|---|
| **cait-sith** | External Rust crate implementing the threshold-ECDSA protocols (triples, pre-signatures, signatures). The node drives it; you never call it. |
| **Triple** | Pre-generated shared randomness input to the pre-signature protocol. Stockpiled ahead of time. |
| **Pre-signature (Presignature)** | Intermediate artifact consumed by the signature protocol. Also stockpiled. One signature costs one pre-signature. |
| **Root public key** | The network's canonical secp256k1 public key. Exposed as `public_key()` on NEAR. All derived keys bend off it. |
| **Derived public key** | `root_pk + epsilon * G`. The "address" a user has on a foreign chain. Computable off-chain. |
| **Epsilon** | Deterministic scalar derived from `(key_version, sender_identity, path)` per chain. Drives the derived public key. See [doc/ACCOUNT_DERIVATION.md](../ACCOUNT_DERIVATION.md). |
| **Path** | Free-form string the integrator provides; part of the epsilon derivation. Think of it like a BIP-32 path but application-defined. |
| **Key version** | `u32` incremented when the network rotates / updates key material. Currently `LATEST_MPC_KEY_VERSION = 0`. `[UNVERIFIED]` constant defined in `mpc_primitives`. |
| **SignId** | Internal 32-byte id for a sign request. On NEAR: computed from `(predecessor, payload, path, key_version)` via `SignId::from_parts`. On other chains: `keccak256(abi_encode(...))` — the **request_id** field you see in events. |
| **request_id** | The cross-chain, integrator-visible id. `keccak256(abi_encode(sender, payload, path, key_version, chain_id, algo, dest, params))`. Computed identically in every non-NEAR indexer. |
| **IndexedSignRequest** | Internal, chain-agnostic representation after normalization. Defined at [protocol/signature.rs:46-75](../../chain-signatures/node/src/protocol/signature.rs#L46-L75) `[UNVERIFIED]`. |
| **SignatureRequested / SignatureResponded** | The two canonical event names, emitted on every chain. Shapes differ but semantics are fixed. |
| **Proposer / Owner** | The node driving a given protocol invocation. Other nodes follow. Deterministically selected per request (hash-based). |
| **Threshold** | Minimum nodes required to cooperate. Today: 5-of-8 ([doc/SCALING_AND_SECURITY.md](../SCALING_AND_SECURITY.md)). |
| **Participants** | The nodes actually selected for one protocol invocation. A subset of size ≥ threshold. |
| **Orchestrating contract** | The NEAR contract at [chain-signatures/contract/](../../chain-signatures/contract/). Source of truth for membership, config, key versions. Not involved in non-NEAR sign requests' happy path. |
| **Signing contract** | Chain-specific contract that accepts user `sign()` calls. One per chain. |
| **Bi-directional** | On Solana/Hydration, a flow where the MPC both signs a serialized destination-chain tx **and** broadcasts it, then returns the destination-chain execution result back to the caller. See [§4.2(b)](04-event-flow.md#42-the-bi-directional-pattern--what-it-actually-is). Not used by the vanilla `sign()` flow. |
| **Backlog** | Node-local persistent storage (in Redis) of in-flight sign requests, keyed by `(Chain, SignId)`. See [chain-signatures/node/src/backlog/](../../chain-signatures/node/src/backlog/). |
| **Mesh** | The node's view of which peers are online, who holds which triples/presignatures. See [doc/mpc_node_specification.md](../mpc_node_specification.md) §"Node Connection Status". Not integrator-facing. |
| **CAIP-2** | Chain identifier standard — e.g. `eip155:1`, `solana:<genesis>`, `bip122:<genesis>`. Used for cross-chain derivation consistency. See [doc/ACCOUNT_DERIVATION.md](../ACCOUNT_DERIVATION.md#chain-identification). |
| **`algo`, `dest`, `params` fields** | Free-form `string` fields carried through the event. The contract doesn't interpret them; the MPC doesn't interpret them; **they influence the `request_id` hash**. Your SDK decides what to put here, but once chosen it's fixed for a given request. |
| **Event-sourcing (within a node)** | Local technique used by nodes for crash recovery during a running protocol. Not a cross-node pattern. |

---

[← §8 Experiments](08-experiments.md) · [Index](README.md)
