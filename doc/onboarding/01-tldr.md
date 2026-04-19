# §1 — TL;DR

[← Index](README.md) · Next: [§2 Architecture →](02-architecture.md)

---

Sig.Network MPC is a **distributed threshold-ECDSA signing service**. A fixed cohort of nodes (currently 8, threshold 5 — [doc/SCALING_AND_SECURITY.md](../SCALING_AND_SECURITY.md)) jointly holds shares of one secp256k1 root key. None of them can sign alone; a quorum must cooperate. Users never see the key.

A user "account" on any secp256k1 chain is a **derived public key** of `root_pk + caller_identity + path`. Because derivation is deterministic and public, an integrator can compute the same address client-side and on-chain, without talking to any MPC node. See [doc/ACCOUNT_DERIVATION.md](../ACCOUNT_DERIVATION.md).

## Core data flow (all chains, same shape)

1. **User calls `sign(...)`** on a signing contract (NEAR, Ethereum, Solana, or a Hydration-runtime pallet).
2. **Contract emits `SignatureRequested`** (as an event/log/CPI/substrate event — chain-specific shape, same semantic fields).
3. **MPC nodes' per-chain indexers observe the event**, normalize it into an internal `IndexedSignRequest`, validate it, and enqueue it for the signature pipeline.
4. **Threshold ECDSA runs.** One node (the "proposer") drives the protocol; `threshold` nodes must participate. This consumes one pre-generated "pre-signature" (itself built from pre-generated "triples").
5. **The proposer calls `respond(request_id, signature)`** back on-chain. This either (a) emits `SignatureResponded` for the caller to observe, or (b) on NEAR, resumes an on-chain **yielded Promise** (NEP-519, a NEAR-runtime primitive — not an SDK abstraction) so the signature is returned as the normal return value of the original `sign()` call. See [§4.2](04-event-flow.md#nears-on-chain-yieldresume-primitive-nep-519) for how that actually works.

## What it lets you do, per chain

Inverse view of the components table — capabilities, not components:

| Capability | What an integrator gets | Chains |
|---|---|---|
| **Async sign via event** | Call `sign()`, observe `SignatureRequested`, wait for `SignatureResponded` matching your `request_id`. Requires an off-chain listener state machine. | Ethereum, Solana, Hydration |
| **Sync-feeling sign via on-chain yield/resume** | Call `sign()`, just `.await` the tx. The NEAR runtime parks the Promise (NEP-519, [§4.2](04-event-flow.md#nears-on-chain-yieldresume-primitive-nep-519)) and resumes it when `respond()` arrives. No listener. No correlation id. | NEAR only |
| **Bi-directional sign + broadcast + respond** | Call `sign_bidirectional(serialized_tx, caip2_id, program_id, ...)`. The MPC signs, **submits the signed tx on the destination chain**, captures the execution result, and calls `respond_bidirectional(request_id, serialized_output, signature)` back on the origin. True "one-call cross-chain execute." | Solana (see [contract-sol/src/lib.rs:90-119](../../chain-signatures/contract-sol/src/lib.rs#L90-L119)); Hydration `[UNVERIFIED]` pallet-side, but indexer + node handlers support `SignBidirectionalRequested`/`RespondBidirectionalEvent` |
| **Off-chain address derivation** | Compute a user's Sig.Network address client-side: `derive_key(root_pk, epsilon(sender, path, key_version))`. Pure math, no network call. | All chains (deterministic, see [doc/ACCOUNT_DERIVATION.md](../ACCOUNT_DERIVATION.md)) |
| **On-chain view helper for derived public key** | Contract view fn that returns the derived pk for a given `(path, predecessor)` so smart contracts can look up addresses on-chain. | NEAR only — [contract/src/lib.rs:204-219](../../chain-signatures/contract/src/lib.rs#L204-L219) |
| **Explicit error-response event** | `respondError(...)` emits `SignatureError(request_id, responder, error)` so listeners can surface failures instead of hanging forever. | Ethereum only — [ChainSignatures.sol:148-156](../../chain-signatures/contract-eth/contracts/ChainSignatures.sol#L148-L156). NEAR returns errors via the Promise result; Solana/Hydration have no error event today `[UNVERIFIED]` for Hydration |
| **Deposit-based anti-spam** | Attach a fee to `sign()`. Gates abuse. Dynamic on NEAR (scales with load, [contract/src/lib.rs:231-242](../../chain-signatures/contract/src/lib.rs#L231-L242)), fixed on Ethereum, field exists on Solana but `[UNVERIFIED]` enforcement | NEAR (dynamic), Ethereum (fixed); Solana partial |

Two things worth noticing from this table:

1. **Bi-directional and yield/resume are orthogonal.** They solve different problems. Yield/resume on NEAR makes *single-chain* signing feel synchronous. Bi-directional on Solana/Hydration makes *cross-chain* execute feel atomic. A future chain could have both or neither.
2. **"Async sign via event" is the lowest common denominator.** Your SDK's core flow should target this; chain-specific features layer on top.

## Important consequences of this design

There is **no direct HTTP endpoint** to request a signature on the MPC node — you always go via a signing contract on some chain. The node's web server ([chain-signatures/node/src/web/mod.rs:77-96](../../chain-signatures/node/src/web/mod.rs#L77-L96)) only exposes health/state/metrics and inter-node `/msg` + `/sync` RPCs. The closest thing to a "direct call" is bringing up the NEAR sandbox and calling the MPC contract there, since NEAR uses Promise yield/resume and returns the signature as a normal function result — see [§6.7](06-local-bootstrap.md#67-getting-a-signature-directly--the-minimum-viable-path).

Membership, threshold, and config are owned by the **NEAR orchestrating contract**. The Ethereum and Solana signing contracts carry **no membership state** — they are pure event plumbing and anyone may call `respond()` on them ([ChainSignatures.sol:69-78](../../chain-signatures/contract-eth/contracts/ChainSignatures.sol#L69-L78): "Any address can emit this event. Clients should always verify the validity of the signature."). Caveat emptor for SDK work: on non-NEAR chains, your SDK must verify the returned signature itself.

## How to use this guide

Skim [§2 Architecture](02-architecture.md) and [§3 Key files](03-key-files.md). Do [Experiment 1](08-experiments.md#experiment-1--trace-one-signrespond-cycle-on-paper) with the guide open. Then read [§4 Event flow](04-event-flow.md) and [§5 Indexers](05-indexers.md) before attempting hands-on work. [§6 Bootstrap](06-local-bootstrap.md) is the spine. [§7 New chains](07-extending-chains.md) is later reading.

---

[← Index](README.md) · Next: [§2 Architecture →](02-architecture.md)
