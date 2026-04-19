# §2 — Architecture overview

[← §1 TL;DR](01-tldr.md) · [Index](README.md) · Next: [§3 Key files →](03-key-files.md)

---

## 2.1 Components

| Component | What it is | Where in repo |
|---|---|---|
| **MPC node** | Long-running Rust process that holds a key share, runs the threshold protocol, runs indexers, and calls `respond()` back on-chain. | [chain-signatures/node/](../../chain-signatures/node/) |
| **NEAR orchestrating contract** | Source of truth for the network itself (membership, config, key versions) **and** the NEAR signing contract. Yield/resume-based. | [chain-signatures/contract/](../../chain-signatures/contract/) |
| **Ethereum signing contract** | Solidity. `sign()` emits `SignatureRequested`; `respond()` emits `SignatureResponded`. Pure event plumbing. | [chain-signatures/contract-eth/contracts/ChainSignatures.sol](../../chain-signatures/contract-eth/contracts/ChainSignatures.sol) |
| **Solana signing program** | Anchor. Same `sign`/`respond` shape, plus `sign_bidirectional`/`respond_bidirectional` for the callback pattern. | [chain-signatures/contract-sol/src/lib.rs](../../chain-signatures/contract-sol/src/lib.rs) |
| **Hydration pallet** | Substrate pallet on Hydration emits `Signet::SignatureRequested`/`Responded`. | (runtime external; indexer at [chain-signatures/node/src/indexer_hydration.rs](../../chain-signatures/node/src/indexer_hydration.rs)) |
| **Per-chain indexers** | Embedded in the node. One per chain. Detect events, validate, normalize to `IndexedSignRequest`. | [indexer.rs](../../chain-signatures/node/src/indexer.rs) (NEAR), [indexer_eth/](../../chain-signatures/node/src/indexer_eth/), [indexer_sol.rs](../../chain-signatures/node/src/indexer_sol.rs), [indexer_hydration.rs](../../chain-signatures/node/src/indexer_hydration.rs) |
| **cait-sith library** | External crate — threshold ECDSA engine. The node calls it; you shouldn't need to read it. | external dep |
| **Redis** | Local node-scoped storage for triples, presignatures, sync checkpoints. | required at runtime |
| **GCP Secret Manager** | Stores each node's secret key share in production. Local runs use a file. | required at runtime for prod |

Threshold and cohort are determined by the NEAR contract's `ProtocolContractState` ([chain-signatures/contract/src/lib.rs](../../chain-signatures/contract/src/lib.rs)); the non-NEAR contracts carry **no membership state** — they are pure plumbing that anyone may call `respond()` on ([ChainSignatures.sol:69-78](../../chain-signatures/contract-eth/contracts/ChainSignatures.sol#L69-L78) — "Any address can emit this event. Clients should always verify the validity of the signature").

## 2.2 Data-flow diagram

```mermaid
sequenceDiagram
    autonumber
    participant U as User / Caller contract
    participant C as Signing contract<br/>(NEAR / ETH / Solana / Hydration)
    participant IDX as Per-chain indexer<br/>(in MPC node)
    participant NODE as MPC node core<br/>(protocol loop)
    participant PEERS as Other MPC nodes<br/>(≥threshold-1)
    participant RPC as Node rpc.rs

    U->>C: sign(payload, path, key_version, ...)<br/>+ deposit
    C-->>C: validate deposit / key_version / etc.
    C->>C: emit SignatureRequested(...)<br/>(event / log / CPI)

    Note over IDX: poll / subscribe / proof
    C-->>IDX: event observed
    IDX->>IDX: validate (deposit, scalar, key_version)<br/>derive epsilon, build SignId
    IDX->>NODE: ChainEvent::SignRequest(IndexedSignRequest)

    NODE->>NODE: insert into backlog<br/>(persistent, keyed by (chain, sign_id))
    NODE->>NODE: proposer selection
    NODE->>PEERS: cait-sith protocol messages<br/>(over /msg HPKE-encrypted CBOR)
    PEERS-->>NODE: protocol messages
    NODE->>NODE: FullSignature { big_r, s }
    NODE->>NODE: verify signature matches derived pk

    alt Proposer only
        NODE->>RPC: rpc.publish(...)
        RPC->>C: respond(request_id, signature)
        C->>C: emit SignatureResponded / resume yield Promise
    end

    C-->>U: signature delivered<br/>(event listener OR Promise return)
```

The MPC network's internal node-to-node message protocol is **out of scope** for integrators — see [doc/mpc_node_specification.md](../mpc_node_specification.md) if you need to reason about node-level behaviour. For integrators, only steps ①–③ and step ⑧ are the surface area you touch.

## 2.3 How the pieces communicate

- **User ↔ chain**: normal chain transaction + event stream. Off-chain you compute a `request_id` hash and listen for `SignatureResponded` with that id (except on NEAR, where you just `.await` the Promise). Request-id encoding is `keccak256(abi_encode(sender, payload, path, key_version, chain_id, algo, dest, params))` across chains — see [indexer_sol.rs:146-162](../../chain-signatures/node/src/indexer_sol.rs#L146-L162). More in [§4.1](04-event-flow.md#41-what-a-signaturerequested-event-actually-contains).
- **Indexer ↔ MPC core**: all chains produce the same `ChainEvent` enum and drain through the same stream. See [chain-signatures/node/src/stream/mod.rs](../../chain-signatures/node/src/stream/mod.rs) + [stream/ops.rs](../../chain-signatures/node/src/stream/ops.rs). More in [§5.1](05-indexers.md#51-shared-shape).
- **Node ↔ node**: HPKE-encrypted CBOR over HTTP `POST /msg` and `POST /sync` ([web/mod.rs:86-92](../../chain-signatures/node/src/web/mod.rs#L86-L92)). You will not integrate against these.
- **Node ↔ Redis**: triple, presignature, backlog, and checkpoint storage. Nothing external to worry about — but without Redis the node will not start ([cli.rs](../../chain-signatures/node/src/cli.rs)).

Canonical architecture doc: [doc/ARCHITECTURE.md](../ARCHITECTURE.md) for the official view of the same picture.

---

[← §1 TL;DR](01-tldr.md) · [Index](README.md) · Next: [§3 Key files →](03-key-files.md)
