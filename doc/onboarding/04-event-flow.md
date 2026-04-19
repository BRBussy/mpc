# §4 — Event flow & request validation (deep)

[← §3 Key files](03-key-files.md) · [Index](README.md) · Next: [§5 Indexers →](05-indexers.md)

---

## 4.1 What a `SignatureRequested` event actually contains

All chains carry the **same semantic payload**, though wire shapes differ.

**Ethereum** — [ChainSignatures.sol:55-65](../../chain-signatures/contract-eth/contracts/ChainSignatures.sol#L55-L65):

```solidity
event SignatureRequested(
    address sender,        // msg.sender at time of sign()
    bytes32 payload,       // 32-byte hash to sign
    uint32 keyVersion,     // 0 today
    uint256 deposit,       // msg.value
    uint256 chainId,       // block.chainid — NOT user-supplied
    string path,           // derivation path, caller's choice
    string algo,           // free-form, e.g. "ecdsa", for SDK dispatch
    string dest,           // free-form destination chain hint
    string params          // free-form extra params
);
```

**Solana** — [contract-sol/src/lib.rs:206-219](../../chain-signatures/contract-sol/src/lib.rs#L206-L219):

```rust
#[event]
pub struct SignatureRequestedEvent {
    pub sender: Pubkey,
    pub payload: [u8; 32],
    pub key_version: u32,
    pub deposit: u64,
    pub chain_id: String,        // currently hardcoded to "solana" in lib.rs:78
    pub path: String,
    pub algo: String,
    pub dest: String,
    pub params: String,
    pub fee_payer: Option<Pubkey>,
}
```

**NEAR**: no discrete "event" type — the NEAR contract appends a JSON log line ([contract/src/lib.rs:171-175](../../chain-signatures/contract/src/lib.rs#L171-L175)), **and** records the request in an on-contract pending-requests list that the indexer queries directly ([chain-signatures/node/src/indexer.rs:85-95](../../chain-signatures/node/src/indexer.rs#L85-L95)). The request hash is `SignId::from_parts(&predecessor, &payload_bytes, &path, key_version)` — see [contract/src/lib.rs:166](../../chain-signatures/contract/src/lib.rs#L166).

**Hydration**: Substrate event `Signet::SignatureRequested` with SCALE-encoded fields; verified via Merkle proof — [indexer_hydration.rs:460-478](../../chain-signatures/node/src/indexer_hydration.rs) `[UNVERIFIED]` (exact line numbers — directory was line-counted at 833 but specific pattern lines not individually verified).

The **`request_id`** an integrator uses to correlate response to request is a `keccak256` over the abi-encoded tuple `(sender, payload, path, key_version, chain_id, algo, dest, params)` — implemented identically in every indexer ([indexer_sol.rs:146-162](../../chain-signatures/node/src/indexer_sol.rs#L146-L162)) and easy to reproduce in any SDK. **Critically**: NEAR uses its own `SignId` scheme, not this keccak id; the two do not interchange.

Full derivation-path spec: [doc/ACCOUNT_DERIVATION.md](../ACCOUNT_DERIVATION.md).

### Error response events

There is a third event you should listen for on Ethereum: `SignatureError(bytes32 indexed requestId, address responder, string error)` ([ChainSignatures.sol:87-91](../../chain-signatures/contract-eth/contracts/ChainSignatures.sol#L87-L91)), emitted by `respondError(...)` ([ChainSignatures.sol:148-156](../../chain-signatures/contract-eth/contracts/ChainSignatures.sol#L148-L156)). An SDK that subscribes only to `SignatureResponded` will silently ignore network-signalled failures. Like `respond`, `respondError` is permissionless — anyone may emit the event, so clients must not use error events for business logic (the contract's own notice at [ChainSignatures.sol:82](../../chain-signatures/contract-eth/contracts/ChainSignatures.sol#L82)). NEAR returns errors via the Promise resolution directly; Solana's minimal contract has no error event today; Hydration's error path is `[UNVERIFIED]`.

## 4.2 The "bi-directional" pattern — what it actually is

Two distinct things share the word; don't conflate them:

**(a) Chain-neutral request/response** — the normal flow. Two transactions on the same chain: user's `sign()` → event observed → node's `respond()` → event observed → client resolves. This is what Ethereum and (standard) Solana do. It is technically request/response, not "bi-directional" in any novel sense.

**(b) `sign_bidirectional` / `respond_bidirectional`** — a cross-chain execution flow. Present on Solana and Hydration only. Sketch:

- Caller invokes `sign_bidirectional(serialized_transaction, caip2_id, ..., program_id, ...)` — [contract-sol/src/lib.rs:90-119](../../chain-signatures/contract-sol/src/lib.rs#L90-L119).
- The MPC network both **signs the serialized transaction** *and* **relays the signed transaction to the destination chain** (identified by `caip2_id`). The node logic for relaying lives in [chain-signatures/node/src/sign_bidirectional.rs](../../chain-signatures/node/src/sign_bidirectional.rs) + [respond_bidirectional.rs](../../chain-signatures/node/src/respond_bidirectional.rs).
- After the destination-chain tx returns a result, the node calls `respond_bidirectional(request_id, serialized_output, signature)` back on the origin chain ([contract-sol/src/lib.rs:43-61](../../chain-signatures/contract-sol/src/lib.rs#L43-L61)). **This does not invoke the caller's `program_id` via CPI** — the handler just emits `RespondBidirectionalEvent` and returns. The `program_id` parameter passed at `sign_bidirectional` time is unused by the current implementation. The origin-side dApp has to run its own off-chain listener to pick up the event and act on the output. This is a significant gap from real cross-chain invocation primitives (LayerZero, Wormhole, IBC, Hyperlane all deliver into destination contracts synchronously). See [§10.9](10-potential-risks.md#109-bidirectional-responses-dont-actually-invoke-the-caller) for the full analysis.

**For your SDK work**: treat `sign_bidirectional` as an experimental/optional feature. The mainline integration surface is plain `sign()` + event listening.

### NEAR's on-chain yield/resume primitive (NEP-519)

**NEAR's "callback" is different again** — and this is a detail worth internalizing, because it is *not* SDK magic. NEAR's runtime itself exposes host functions that let a contract **park a Promise on-chain** and have some later transaction resume it. It's specified in [NEP-519 "Yield Execution"](https://github.com/near/NEPs/blob/master/neps/nep-0519.md) and the calls are part of the NEAR VM ABI, not the `near-sdk` crate.

Mechanically, from [contract/src/lib.rs:762-794](../../chain-signatures/contract/src/lib.rs#L762-L794):

1. User's tx calls `sign(...)`. The contract invokes **`env::promise_yield_create(...)`** — a host function. This returns a Promise pointing at callback `clear_state_on_finish` and writes a 32-byte `data_id` into a register.
2. The contract persists `data_id` keyed by `sign_id` ([lib.rs:780](../../chain-signatures/contract/src/lib.rs#L780) `self.set_request_yield(...)`).
3. The contract chains a second callback `return_signature_on_finish` via `promise_then` and calls `env::promise_return(final_yield_promise)` — telling the runtime "my return value is this pending Promise."
4. NEAR's runtime parks the Promise in the state tree. The caller's tx waits on it, subject to a runtime-enforced yield timeout `[UNVERIFIED]` for this contract; NEP-519's default is 200 blocks.
5. Later, an MPC node calls `respond(sign_id, signature)`. That function at [contract/src/lib.rs:290](../../chain-signatures/contract/src/lib.rs#L290) invokes **`env::promise_yield_resume(&data_id, &serialized_signature)`**. The parked Promise wakes; `return_signature_on_finish` ([lib.rs:799-810](../../chain-signatures/contract/src/lib.rs#L799-L810)) receives the signature as a callback arg and returns it.
6. The original caller's `sign(...)` tx resolves with the signature as its ordinary return value.

This is **not** a callback to a third contract, and there is no SDK-side polling. From any caller — JS client, Rust client, another NEAR contract cross-contract-calling — `sign(...)` looks like an ordinary function that "takes a few seconds to return."

## 4.3 Request validation — what gates a request before nodes participate

Two checkpoints. Integrators need to know both, because a rejected request will just silently not produce a response.

### Checkpoint 1 — contract-side

The contract itself does minimal checks.

| Chain | Checks |
|---|---|
| NEAR | payload must be valid scalar ([lib.rs:136-139](../../chain-signatures/contract/src/lib.rs#L136-L139)); `key_version ≤ latest_key_version()` ([lib.rs:140-142](../../chain-signatures/contract/src/lib.rs#L140-L142)); `attached_deposit ≥ experimental_signature_deposit()` ([lib.rs:144-152](../../chain-signatures/contract/src/lib.rs#L144-L152)); `prepaid_gas ≥ GAS_FOR_SIGN_CALL` (50 Tgas, [lib.rs:154-160](../../chain-signatures/contract/src/lib.rs#L154-L160)); pending requests < 128 ([lib.rs:162-164](../../chain-signatures/contract/src/lib.rs#L162-L164)); no duplicate `SignId` ([lib.rs:167-169](../../chain-signatures/contract/src/lib.rs#L167-L169)). |
| Ethereum | Only `msg.value >= signatureDeposit` ([ChainSignatures.sol:115](../../chain-signatures/contract-eth/contracts/ChainSignatures.sol#L115)). That's it. Replay/dedup is the node's problem. |
| Solana | Requester must be signer (Anchor-enforced, [lib.rs:170-176](../../chain-signatures/contract-sol/src/lib.rs#L170-L176)). The `program_state` account carries a `signature_deposit` field ([lib.rs:138](../../chain-signatures/contract-sol/src/lib.rs#L138)) which is **echoed into the emitted event verbatim** ([lib.rs:77](../../chain-signatures/contract-sol/src/lib.rs#L77)) but is **not enforced**: the `sign` handler performs no `system_program::transfer` CPI, no lamport balance check, and no inspection of the instruction sysvar for a sibling transfer. The caller pays only the base Solana tx fee. The indexer's downstream `deposit > 0` check ([indexer_sol.rs:166-169](../../chain-signatures/node/src/indexer_sol.rs#L166-L169)) is therefore a function of the **Solana contract admin's `initialize()` config**, not caller behavior. An MPC-side cross-check (inspect the tx for a transfer of the expected amount to a designated address) would be the natural fix and is **not implemented**. See [§11](11-funds-and-refunds.md). |
| Hydration | Pallet logic — inspected indirectly through the indexer's validation (below). |

### Checkpoint 2 — indexer-side (every MPC node)

This is where most "invisible rejections" happen. Same rules across chains:

- `deposit == 0` → reject ([indexer_sol.rs:166-169](../../chain-signatures/node/src/indexer_sol.rs#L166-L169); analogous in eth/hydration).
- `key_version > LATEST_MPC_KEY_VERSION` → reject ([indexer_sol.rs:171-174](../../chain-signatures/node/src/indexer_sol.rs#L171-L174)).
- `payload` must decode to a valid secp256k1 scalar ([indexer_sol.rs:176-182](../../chain-signatures/node/src/indexer_sol.rs#L176-L182)).
- `payload > MAX_SECP256K1_SCALAR` (i.e. not reducible into the field) → reject ([indexer_sol.rs:184-187](../../chain-signatures/node/src/indexer_sol.rs#L184-L187)).

The indexer then derives `epsilon` via `derive_epsilon_{near,eth,sol,hydration}(key_version, sender_string, path)` ([indexer_sol.rs:191](../../chain-signatures/node/src/indexer_sol.rs#L191)), packages an `IndexedSignRequest` with an `entropy` field (source varies per chain — transaction hash on Ethereum, first 32 bytes of tx signature on Solana, `blake2_256(event_bytes)` on Hydration), and forwards it through `process_sign_request` ([stream/ops.rs:263-282](../../chain-signatures/node/src/stream/ops.rs#L263-L282)).

### Replay protection

`request_id` is deterministic from event fields; duplicate events mapping to the same id collide in the backlog (last write wins — [backlog/mod.rs:222-236](../../chain-signatures/node/src/backlog/mod.rs) `[UNVERIFIED]` exact lines). After the proposer publishes a successful response, the backlog entry is dropped, and further duplicate events are ignored because `backlog.get()` returns nothing ([stream/ops.rs:338-345](../../chain-signatures/node/src/stream/ops.rs#L338-L345) `[UNVERIFIED]`).

### Signature-side validation (before publishing)

The proposer verifies the produced `FullSignature { big_r, s }` against the expected derived public key and the payload before sending the tx ([chain-signatures/node/src/rpc.rs:135-165](../../chain-signatures/node/src/rpc.rs#L135-L165) `[UNVERIFIED]` exact line numbers). This guarantees the on-chain `respond()` call never publishes a bogus signature.

## 4.4 Finality & reorg behavior

| Chain | Finality rule | Reorg handling |
|---|---|---|
| Ethereum | Waits for `finalized` block tag unless `MPC_ETH_OPTIMISTIC_REQUESTS=true` ([indexer_eth/mod.rs](../../chain-signatures/node/src/indexer_eth/mod.rs)). Finalized-block refresh interval tuneable via `MPC_ETH_REFRESH_FINALIZED_INTERVAL` (default 10000 ms). | Post-finality hash mismatch → block skipped `[UNVERIFIED]`. |
| Solana | `CommitmentConfig::confirmed()` ([indexer_sol.rs:381-383](../../chain-signatures/node/src/indexer_sol.rs#L381-L383) `[UNVERIFIED]` exact line). | None explicit — relies on commitment. |
| Hydration | Subxt `subscribe_finalized()`; every event must carry a valid Merkle proof against the block `state_root`. | Reorg-proof by construction. |
| NEAR | Implicit — the indexer reads from a view call on the latest block; finalization handled by NEAR itself. | None — NEAR finalizes fast. |

## 4.5 What `path` means to the contract and to you

`path` is a **free-form string** the contract does not interpret. It is hashed together with the sender's identity to derive `epsilon`, which in turn bends the root public key into the user's derived public key (`root_pk + epsilon * G`). See [doc/ACCOUNT_DERIVATION.md](../ACCOUNT_DERIVATION.md). Choices your SDK users make:

- Use the same `path` across chains for one user → they effectively get one consistent "Sig.Network address" per CAIP-2 chain id.
- Use different `path`s for different dApps to produce distinct sub-accounts.
- Derive on the client: compute `derived_pk = derive_key(root_pk, derive_epsilon_<chain>(key_version, sender, path))`. The NEAR contract exposes `derived_public_key(...)` as a view helper ([contract/src/lib.rs:204-219](../../chain-signatures/contract/src/lib.rs#L204-L219)) if you want the network's canonical answer.

---

[← §3 Key files](03-key-files.md) · [Index](README.md) · Next: [§5 Indexers →](05-indexers.md)
