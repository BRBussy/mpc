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

## 10.6 What happens to the user's deposit on failure

| Chain | Standard flow | Bidirectional flow |
|---|---|---|
| NEAR | Yield times out (~200 blocks ≈ 4 min), **full deposit refunded** via `refund_on_fail` ([contract/src/lib.rs:812-817](../../chain-signatures/contract/src/lib.rs#L812-L817), [:833-856](../../chain-signatures/contract/src/lib.rs#L833-L856)). User's `sign()` call returns `SignError::Timeout`. | N/A — NEAR doesn't have bidirectional. |
| Ethereum | Deposit stays in contract balance, admin-withdrawable ([ChainSignatures.sol:181-195](../../chain-signatures/contract-eth/contracts/ChainSignatures.sol#L181-L195)). `respondError` event can be emitted by the network ([ChainSignatures.sol:148-156](../../chain-signatures/contract-eth/contracts/ChainSignatures.sol#L148-L156)) but is not auto-emitted on proposer failure. Listener hangs indefinitely. | N/A — Ethereum doesn't have bidirectional. |
| Solana | Deposit field exists but `[UNVERIFIED]` enforcement in current minimal contract. No explicit refund path. | Same as standard, plus destination-chain gas paid by the node's own account `[UNVERIFIED]`. |
| Hydration | `[UNVERIFIED]` — pallet is outside this repo. | Same. |

## 10.7 Is this overblown? An honest take

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

## 10.8 SDK-level mitigations (what integrators should do)

1. **Always apply a client-side timeout.** Don't rely on the network to signal failure.
2. **Retry with the same parameters** for idempotent requests — `request_id` is deterministic, and the backlog replaces on insert ([stream/ops.rs:263-282](../../chain-signatures/node/src/stream/ops.rs#L263-L282)). A retry will typically land with a different proposer.
3. **Do not retry bidirectional requests automatically** without checking whether the destination-chain tx already executed. Use the `tx_id` from `ExecutionConfirmed` (if observable from the user's side) or poll the destination chain manually to detect partial success before resubmitting.
4. **On Ethereum, subscribe to `SignatureError` as well as `SignatureResponded`.** Any `SignatureError` with your `request_id` is a permission slip to stop waiting; it's still not a guarantee you'll receive one on failure.
5. **Prefer NEAR for correctness-sensitive flows** where a user is willing to pay in NEAR. It's the only chain with (a) contract-level signature verification, (b) runtime-enforced yield timeout, and (c) deposit refund on failure.
6. **Monitor node /state endpoints** ([§6.6](06-local-bootstrap.md#66-what-the-node-actually-exposes--the-http-surface)) during integration testing to verify the network is actually `Running`. A network with insufficient live nodes will accept `sign()` calls silently and never respond.

## 10.9 Possible in-network fixes (what would resolve this)

These are observations, not proposals — but they frame the risk's severity by showing what *could* be done.

- **Multi-node publish fallback.** The cheapest improvement: if the proposer hasn't published within N seconds, let the next-ranked participant try. Every participant already has the signature in memory. This would convert "at-most-once" into something closer to "at-least-once with mild over-publishing" on Ethereum/Solana (both chains tolerate duplicate `respond()` calls since the event is permissionless and the caller verifies). A roughly one-file change to [signature.rs:1032-1039](../../chain-signatures/node/src/protocol/signature.rs#L1032-L1039).
- **Explicit failure events.** Auto-emit `SignatureError` on Ethereum when a proposer gives up. Equivalent on Solana/Hydration. Gives SDKs a positive timeout signal rather than an ambiguous absence-of-response.
- **Cross-chain two-phase commit for bidirectional.** Hard — classic distributed systems problem. Probably out of scope; the realistic alternative is stronger client-side reconciliation tooling.
- **Economic incentives.** Staking + slashing would align operator behaviour with user interests. Large design change; also introduces its own failure modes (griefing, nothing-at-stake).

## 10.10 What this means for your onboarding-guide work

When you write SDK docs:

- Be explicit that `sign()` is **best-effort** on Ethereum and Solana.
- Document retry idempotence: same inputs → same `request_id` → safe to resubmit.
- For bidirectional flows, publish an "execution reconciliation" helper that inspects the destination chain before allowing retry.
- Recommend timeouts and back off recommendations.
- Caveat: "delivery is not guaranteed" is a reasonable thing to put in red on the landing page of a partner SDK. It is not uncharitable to the network — it's accurate.

---

[← §9 Glossary](09-glossary.md) · [Index](README.md)
