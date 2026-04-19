# §10 — Potential risks

[← §9 Glossary](09-glossary.md) · [Index](README.md)

---

> This section flags **liveness and delivery-guarantee risks** in the current design — things SDK authors and dApp integrators should know about, with concrete code citations. It is not a security audit of the cryptographic protocol.

## 10.1 Summary

Sig.Network MPC is a **non-Byzantine, non-crypto-economic** threshold-signing network. It has:

- ✓ Cryptographic safety (≥threshold honest operators can't be forged around).
- ✗ **No liveness guarantee** for any individual request.
- ✗ **No slashing or economic penalties.**
- ✗ **No at-least-once delivery** of signatures back to the requester.
- ✗ **No audit trail** of why a request was dropped (on Ethereum/Solana/Hydration).
- ✗ **No indexer catchup** on Solana — silent data loss during any node downtime ([§10.6](#106-indexer-delivery-guarantees--what-survives-a-crash)).
- ✗ **Computed signatures live only in proposer RAM** — never persisted ([§10.7](#107-the-computed-signature-in-ram-gap)).
- ✗ **Redis is a per-node single point of failure** and `[UNVERIFIED]` whether persistence (AOF/RDB) is configured in production ([§10.8](#108-redis-as-single-point-of-failure)).

The security model is "trust the whitelisted operator set to behave reasonably" — similar in spirit to federated / permissioned validator sets. It's explicitly stated in [doc/mpc_node_specification.md:40-42](../mpc_node_specification.md):

> It is not byzantine fault-tolerant, however, which this specification explicitly not is trying to achieve.

## 10.2 The honesty model

Membership of the operator set is governed by `vote_*` methods on the NEAR contract ([chain-signatures/contract/src/lib.rs](../../chain-signatures/contract/src/lib.rs) search for `vote_join`, `vote_leave`). A misbehaving operator is kicked out by a social/governance process, not by an automatic mechanism. Until that happens, a malicious operator can:

- **Decline to participate** in protocol invocations where they're asked.
- **Fail to publish** a response when they're the proposer (see [§10.4](#104-the-proposer-as-single-point-of-failure)).
- Not do anything worse — they **cannot forge signatures**, because cait-sith requires ≥threshold cooperators.

They gain nothing economically by misbehaving (no fees flow to them). They lose nothing economically either (no stake). Misbehaviour is a pure reputation/governance issue.

## 10.3 Delivery guarantees — what the code actually does

**Within a single proposer node**, there is bounded retry:

- `MAX_PUBLISH_RETRY = 6` ([rpc.rs:57](../../chain-signatures/node/src/rpc.rs#L57))
- 5-second fixed backoff, 120-second per-attempt timeout ([rpc.rs:946-950](../../chain-signatures/node/src/rpc.rs#L946-L950))
- After 6 attempts: `"exceeded max retries, trashing publish request"` ([rpc.rs:1026-1031](../../chain-signatures/node/src/rpc.rs#L1026-L1031)) — the signature is dropped.

**Across nodes**, there is nothing:

- Every participating node has the full `FullSignature { big_r, s }` in memory after the protocol completes.
- But only the proposer tries to publish ([signature.rs:1032-1039](../../chain-signatures/node/src/protocol/signature.rs#L1032-L1039)):

  ```rust
  if self.proposer == me {
      ctx.rpc.publish(...);
  }
  ```

- If the proposer crashes between "signature computed" and "tx mined", no other node picks it up. The request is permanently orphaned; the computed signature is discarded.

This is effectively **at-most-once** delivery, with a short retry window to cover transient RPC flakiness.

## 10.4 The proposer as single point of failure

Proposer selection rotates per-request via the `posit` round mechanism ([posit.rs](../../chain-signatures/node/src/protocol/posit.rs)), so a malicious operator can't monopolize the role — but they *can* DOS any request where they happen to be selected. What they can do:

| Scenario | Effect on the user |
|---|---|
| Proposer crashes after computing signature, before broadcasting | Signature lost. User's deposit behavior depends on chain (below). |
| Proposer deliberately drops the publish call | Same as above. Indistinguishable from a crash. |
| Proposer broadcasts but chain rejects (wrong nonce, gas, revert) | 6 retries, then dropped. |
| Proposer forges a bogus signature | **Not possible** — threshold crypto prevents it. On NEAR the contract also verifies ([contract/src/lib.rs:278-286](../../chain-signatures/contract/src/lib.rs#L278-L286)) so even a bogus `respond()` call is rejected. On Ethereum/Solana the SDK must verify client-side. |

What keeps this acceptable in practice:

- Small whitelisted operator set, operationally monitored.
- Retry from the client side is safe (request_id is deterministic from event fields; see [§4.1](04-event-flow.md#41-what-a-signaturerequested-event-actually-contains)).
- A new `sign()` call after a suspected failure will likely be picked up by a different proposer on the next posit round.

## 10.5 Flow-specific risk compounding

| Flow | Async hops where a failure drops the request |
|---|---|
| **Flow A** (event-based) | 2 — origin-indexer observe, proposer publish |
| **Flow B** (NEAR yield/resume) | 2 — near-indexer poll, proposer publish — but **yield timeout + full deposit refund** bounds the loss |
| **Flow C** (bi-directional) | **5** — origin-indexer observe, proposer broadcast to destination, **destination-indexer observe + extract output**, second-round signing, proposer publish respond_bidirectional |

Flow C is the most fragile by a wide margin. It requires:
1. An origin-chain indexer to observe `SignBidirectionalRequested`.
2. The threshold network to complete round-1 signing.
3. The proposer to broadcast to the destination chain successfully.
4. A **destination-chain indexer** on some MPC node to observe the executed tx and extract the output ([respond_bidirectional.rs:130-166](../../chain-signatures/node/src/respond_bidirectional.rs#L130-L166), [indexer_eth/mod.rs:1102](../../chain-signatures/node/src/indexer_eth/mod.rs#L1102)).
5. The threshold network to complete round-2 signing over `(request_id, serialized_output)`.
6. The proposer to publish `respond_bidirectional` on origin.

A failure at any of these stages silently strands the request. **In the worst case, the user paid origin-chain gas AND deposit AND destination-chain gas AND the destination tx executed successfully — but no response comes back to origin.** That's a non-idempotent failure: retrying the `sign_bidirectional` call would re-execute the destination-chain tx.

## 10.6 Indexer delivery guarantees — what survives a crash

Each MPC node runs its own per-chain indexer and its own Redis. There's no shared queue, no consensus on "which block have we processed." Each node independently observes the chain and pushes events into its own local pipeline. This makes per-chain indexer persistence **load-bearing for liveness**, and it varies dramatically between chains.

### Per-chain catchup matrix

| Indexer | Cursor persisted? | Recovery on restart | Silent loss during downtime? |
|---|---|---|---|
| **Ethereum** | YES — Redis checkpoint keyed `{account_id}:checkpoint:latest:v3:{chain}` ([checkpoint_storage.rs:35](../../chain-signatures/node/src/storage/checkpoint_storage.rs#L35), written on each processed block at [stream/mod.rs:160](../../chain-signatures/node/src/stream/mod.rs#L160)) | Loads checkpoint, resumes from `last_processed_block + 1` ([indexer_eth/mod.rs:790-813](../../chain-signatures/node/src/indexer_eth/mod.rs#L790-L813)) | **Partial** — only if node is down longer than RPC archive window (e.g. Infura ~128 blocks). Within the window: recovered. |
| **Hydration** | YES — same Redis checkpoint mechanism | Same as Ethereum | **Partial** — relies on `subscribe_finalized()` ([indexer_hydration.rs:390](../../chain-signatures/node/src/indexer_hydration.rs#L390)); if Subxt subscription drops *without* triggering a reconnect that does a catchup, missed finalized blocks are lost `[UNVERIFIED]` on whether Subxt reconnect replays |
| **Solana** | **NO** — in-memory dedup cache only ([indexer_sol.rs:637-742](../../chain-signatures/node/src/indexer_sol.rs#L637-L742)) | WebSocket resubscribes to current slot. No `getSignaturesForAddress` catchup, no slot replay. | **YES — total loss of any sign event that arrived while the node was disconnected.** |
| **NEAR** | **NO** — no cursor; polls `pending_requests_data()` view call every 750ms ([indexer.rs:85-95, 249-250](../../chain-signatures/node/src/indexer.rs#L85-L95)) | Reads current on-contract pending list; in-memory dedup cache rebuilt from scratch | **Conditional** — depends entirely on the NEAR contract's state. If a request was marked complete on-chain (respond() happened) while the node was down, the indexer will never see it. On the other hand, still-pending requests ARE recovered because they remain in contract state. |

So: Ethereum and Hydration have durable-ish recovery (modulo the RPC archive window and Redis persistence — see §10.8). **Solana silently drops everything missed during any outage.** NEAR is partially self-healing because the contract holds state, but completed-while-down requests vanish.

### The extract → checkpoint non-atomic window

Even on Ethereum/Hydration, there's a crash window. The indexer:

1. Fetches a block
2. Extracts logs / events
3. Pushes events into the in-memory mpsc channel
4. Later — when `ChainEvent::Block(n)` is processed downstream — writes `last_processed_block = n` to Redis

Steps 3 and 4 are not atomic. If the node crashes between "event pushed to channel" and "checkpoint persisted", then on restart the indexer re-reads that block ([stream/mod.rs:160](../../chain-signatures/node/src/stream/mod.rs#L160) writes the checkpoint after the block's events have been emitted). Re-reading is usually fine — events are re-emitted and either hit `process_sign_request` (which overwrites the backlog entry) or get rejected because the `request_id` already maps to a completed response. But the net effect is **at-least-once event delivery within the indexer only**, and only if the indexer actually survives the crash.

A more insidious gap: `process_sign_request` at [stream/ops.rs:263-282](../../chain-signatures/node/src/stream/ops.rs#L263-L282) does:

```rust
backlog.insert(sign_request.clone()).await;            // durable write
if let Err(err) = sign_tx.send(Sign::Request(sign_request)).await {  // in-memory channel
    tracing::error!(...);
}
```

If the node crashes *between* `backlog.insert` and `sign_tx.send`, the backlog has the entry but the mpsc queue doesn't. On restart, `recover_backlog()` replays ([backlog/mod.rs:620-704](../../chain-signatures/node/src/backlog/mod.rs#L620-L704), invoked from [stream/mod.rs:102-122](../../chain-signatures/node/src/stream/mod.rs#L102-L122)), so this is actually recovered. That's good — this specific window is safe.

But if the indexer crashes **before** `backlog.insert` completes (e.g. mid-validation or before it got to `process_sign_request`), the request is lost unless the indexer re-reads the block (which happens only if the checkpoint hadn't advanced past it). For Solana and NEAR where there's no block-level checkpoint, this "before-first-write" window is much wider.

### Destination-chain indexing for bidirectional (Flow C, step 7)

Same per-chain recovery properties apply to the destination-side observation. The destination-chain indexer:

1. Observes the executed tx in a block
2. Matches its hash against the execution-watcher list in Redis ([backlog/mod.rs:369-407](../../chain-signatures/node/src/backlog/mod.rs#L369-L407) — watchers are hydrated from Redis on startup)
3. Extracts `serialized_output` via `extract_success_tx_output` ([respond_bidirectional.rs:130-166](../../chain-signatures/node/src/respond_bidirectional.rs#L130-L166))
4. Emits `ChainEvent::ExecutionConfirmed` ([indexer_eth/mod.rs:1102](../../chain-signatures/node/src/indexer_eth/mod.rs#L1102))
5. Downstream, the confirmation is packaged as a new `IndexedSignRequest { kind: RespondBidirectional }` and inserted into the backlog ([stream/ops.rs:527-541](../../chain-signatures/node/src/stream/ops.rs#L527-L541))

**Every** MPC node that has the destination chain's indexer enabled does all of the above independently — there's redundancy. So a single-node crash is survivable as long as another node saw the tx and extracted the output. But:

- **If the destination chain is Solana**, redundancy is useless for this flow because Solana's indexer has no persistence. A brief network partition could mean **no node** sees the tx, and the execution confirmation is lost — even though the destination-chain tx executed successfully.
- **If all nodes share an RPC provider** and that provider rate-limits / outages, every watcher fails simultaneously. The redundancy is illusory.
- **Extract → emit is not atomic** either (same pattern as the sign-request side), so the same narrow crash window applies.

## 10.7 The computed-signature-in-RAM gap

A detail that sharpens §10.3: the `FullSignature { big_r, s }` produced by the threshold protocol is **never persisted** anywhere.

- Every participant has the signature in memory after cait-sith returns ([signature.rs:1020-1040](../../chain-signatures/node/src/protocol/signature.rs#L1020-L1040)).
- The proposer calls `ctx.rpc.publish(...)` which queues the signature to an in-process task ([rpc.rs:135-156](../../chain-signatures/node/src/rpc.rs#L135-L156)). Still no Redis write.
- The publish task retries up to 6 times ([rpc.rs:946-1031](../../chain-signatures/node/src/rpc.rs#L946-L1031)), entirely in-memory. If the proposer process dies mid-retry, the signature is gone.
- Non-proposer nodes hold the signature in RAM but have no code path that could publish it on the proposer's behalf.

**The consequence:** if the proposer crashes at *any* point between "protocol finished" and "chain accepted the respond tx", the signature is lost. The network must re-run the sign protocol — which requires a fresh presignature (presignatures are single-use by security construction, so the one consumed is gone too). The re-run is gated by the `posit` round-advance mechanism and can take a non-trivial amount of time.

For the user, this is indistinguishable from "my request was dropped" because there's no external signal that a sign protocol completed successfully before the crash.

## 10.8 Redis as single point of failure

Every node has its own Redis. Redis stores:

- Backlog (pending sign requests) — [checkpoint_storage.rs:41-59](../../chain-signatures/node/src/storage/checkpoint_storage.rs#L41-L59)
- Last-processed-block checkpoints (Ethereum, Hydration)
- Triples — [storage/triple_storage.rs](../../chain-signatures/node/src/storage/triple_storage.rs) `[UNVERIFIED]` exact lines
- Presignatures — [storage/presignature_storage.rs](../../chain-signatures/node/src/storage/presignature_storage.rs) `[UNVERIFIED]` exact lines
- Execution watchers (for bidirectional)

If Redis restarts with an empty dataset (no AOF/RDB persistence), **all of the above is lost from that node**. The node doesn't know anything was lost — it comes back up, reads empty state, and silently starts from scratch.

### Is Redis persistent in production?

**`[UNVERIFIED]` — likely not, based on tests.**

Integration tests spin up Redis via docker with no persistence flags ([integration-tests/src/containers.rs:339-382](../../integration-tests/src/containers.rs)) — purely in-memory. The Helm/Terraform modules under `infra/modules/` do not appear to set AOF/RDB flags either `[UNVERIFIED]` (didn't exhaustively grep every file). This is worth checking directly with ops before building production SDKs on assumptions.

### Per-node vs. network-wide consequences

- A **single node's** Redis loss: that node becomes partially blind until other nodes' indexers pick up the slack — but the "slack" only covers network-level consensus, not this specific node's pending backlog. Sign requests where this node was the proposer get re-proposed (eventually) by a different node.
- **Multiple nodes'** Redis loss simultaneously (e.g. shared infra outage): potentially catastrophic — triples/presignatures are consumed by threshold protocols and must be regenerated network-wide before signatures can resume. Triple generation takes ~30s in the best case ([doc/ARCHITECTURE.md:46](../ARCHITECTURE.md)), so throughput collapses until the stockpile refills.

## 10.9 What happens to the user's deposit on failure

| Chain | Standard flow | Bidirectional flow |
|---|---|---|
| NEAR | Yield times out (~200 blocks ≈ 4 min), **full deposit refunded** via `refund_on_fail` ([contract/src/lib.rs:812-817](../../chain-signatures/contract/src/lib.rs#L812-L817), [:833-856](../../chain-signatures/contract/src/lib.rs#L833-L856)). User's `sign()` call returns `SignError::Timeout`. | N/A — NEAR doesn't have bidirectional. |
| Ethereum | Deposit stays in contract balance, admin-withdrawable ([ChainSignatures.sol:181-195](../../chain-signatures/contract-eth/contracts/ChainSignatures.sol#L181-L195)). `respondError` event can be emitted by the network ([ChainSignatures.sol:148-156](../../chain-signatures/contract-eth/contracts/ChainSignatures.sol#L148-L156)) but is not auto-emitted on proposer failure. Listener hangs indefinitely. | N/A — Ethereum doesn't have bidirectional. |
| Solana | Deposit field exists but `[UNVERIFIED]` enforcement in current minimal contract. No explicit refund path. | Same as standard, plus destination-chain gas paid by the node's own account `[UNVERIFIED]`. |
| Hydration | `[UNVERIFIED]` — pallet is outside this repo. | Same. |

## 10.10 Is this overblown? An honest take

**You are not overblowing it.** Two separate concerns deserve separate weights:

### Concern A — Maliciousness

*Is malice possible?* Yes.
*Is it plausible at scale?* Probably not, given the small whitelisted operator set and public reputation stakes. This concern is roughly equivalent to the risk any federated validator set carries (Cosmos validators, etc.), and it's the kind of thing the industry has broadly accepted as livable for permissioned infrastructure.

### Concern B — Silent drops with paid-for operations

*Is unintentional drop possible?* Yes, trivially — any operator crash between signing and publishing.
*Does it leave the client out of pocket?* Depends on chain. NEAR: no (deposit refunds). Ethereum: yes, deposit is lost. Solana: currently trivial because deposit isn't enforced, but once it is: yes. Bidirectional: **yes, and the destination-chain gas is also lost, and the destination state change already happened.**
*Is there any record?* No. On Ethereum/Solana, the user's only signal is timeout. There's no `"request dropped"` event.
*Is this a known property of at-least-once messaging systems that most chains handle?* It is the classic liveness-vs-safety trade-off. Most chains (including Bitcoin and Ethereum at the L1 level) guarantee liveness because miners/validators are economically motivated to include transactions. This network has **no such incentive layer**, so it can't make the same guarantee.

In short: **this is a real risk, not a theoretical one**, and the bidirectional flow makes it concretely worse. The severity depends on:

- **Value per request** — if signatures gate high-value operations, a dropped request is material.
- **Idempotence of the user's operation** — if retrying is safe, the SDK can paper over drops. Bidirectional flows are often *not* idempotent.
- **User expectations** — a cross-chain "execute operation" primitive that silently fails 0.1% of the time is very different from one that guarantees delivery.

## 10.11 SDK-level mitigations (what integrators should do)

1. **Always apply a client-side timeout.** Don't rely on the network to signal failure.
2. **Retry with the same parameters** for idempotent requests — `request_id` is deterministic, and the backlog replaces on insert ([stream/ops.rs:263-282](../../chain-signatures/node/src/stream/ops.rs#L263-L282)). A retry will typically land with a different proposer.
3. **Do not retry bidirectional requests automatically** without checking whether the destination-chain tx already executed. Use the `tx_id` from `ExecutionConfirmed` (if observable from the user's side) or poll the destination chain manually to detect partial success before resubmitting.
4. **On Ethereum, subscribe to `SignatureError` as well as `SignatureResponded`.** Any `SignatureError` with your `request_id` is a permission slip to stop waiting; it's still not a guarantee you'll receive one on failure.
5. **Prefer NEAR for correctness-sensitive flows** where a user is willing to pay in NEAR. It's the only chain with (a) contract-level signature verification, (b) runtime-enforced yield timeout, (c) deposit refund on failure, and (d) indexer recovery backed by authoritative on-chain state (so indexer downtime doesn't lose requests that are still pending on-chain).
6. **Monitor node `/state` endpoints** ([§6.6](06-local-bootstrap.md#66-what-the-node-actually-exposes--the-http-surface)) during integration testing to verify the network is actually `Running`. A network with insufficient live nodes will accept `sign()` calls silently and never respond.
7. **Treat Solana sign requests as the highest-risk flow for delivery loss.** The Solana indexer has no persistence and no catchup ([§10.6](#106-indexer-delivery-guarantees--what-survives-a-crash)). A brief node outage can drop requests outright. Build in aggressive client-side timeouts and retries for Solana-originated traffic.
8. **Verify Redis persistence before production.** Confirm with the operator / infra team that each node's Redis is configured with AOF or RDB persistence — otherwise a Redis restart wipes triples, presignatures, backlog, and indexer checkpoints for that node ([§10.8](#108-redis-as-single-point-of-failure)). Unpersisted Redis in prod would materially increase drop rates.

## 10.12 Possible in-network fixes (what would resolve this)

These are observations, not proposals — but they frame the risk's severity by showing what *could* be done.

- **Multi-node publish fallback.** The cheapest improvement: if the proposer hasn't published within N seconds, let the next-ranked participant try. Every participant already has the signature in memory. This would convert "at-most-once" into something closer to "at-least-once with mild over-publishing" on Ethereum/Solana (both chains tolerate duplicate `respond()` calls since the event is permissionless and the caller verifies). A roughly one-file change to [signature.rs:1032-1039](../../chain-signatures/node/src/protocol/signature.rs#L1032-L1039).
- **Persist the computed signature before publishing.** Write `FullSignature { big_r, s }` to Redis keyed by `sign_id` before attempting RPC publish, and have a background reaper retry on startup. Closes the RAM-only gap identified in [§10.7](#107-the-computed-signature-in-ram-gap).
- **Solana indexer catchup.** Add `getSignaturesForAddress`-based replay on WebSocket reconnect so that slots missed during downtime are re-scanned. Closes the single largest silent-drop vector on the Solana path ([§10.6](#106-indexer-delivery-guarantees--what-survives-a-crash)).
- **Atomic extract-and-checkpoint.** Persist block events and the checkpoint advance as a single transaction (LPUSH + SET in one `MULTI` block, or use Redis streams). Removes the "extracted-but-not-checkpointed" crash window that exists on Ethereum/Hydration today.
- **Redis persistence by default.** Document AOF or RDB as a required configuration for all production nodes, and add a boot-time check that warns if persistence is disabled.
- **Explicit failure events.** Auto-emit `SignatureError` on Ethereum when a proposer gives up. Equivalent on Solana/Hydration. Gives SDKs a positive timeout signal rather than an ambiguous absence-of-response.
- **Cross-chain two-phase commit for bidirectional.** Hard — classic distributed systems problem. Probably out of scope; the realistic alternative is stronger client-side reconciliation tooling.
- **Economic incentives.** Staking + slashing would align operator behaviour with user interests. Large design change; also introduces its own failure modes (griefing, nothing-at-stake).

## 10.13 What this means for your onboarding-guide work

When you write SDK docs:

- Be explicit that `sign()` is **best-effort** on Ethereum and Solana.
- Document retry idempotence: same inputs → same `request_id` → safe to resubmit.
- For bidirectional flows, publish an "execution reconciliation" helper that inspects the destination chain before allowing retry.
- Recommend timeouts and back off recommendations.
- Caveat: "delivery is not guaranteed" is a reasonable thing to put in red on the landing page of a partner SDK. It is not uncharitable to the network — it's accurate.

---

[← §9 Glossary](09-glossary.md) · [Index](README.md)
