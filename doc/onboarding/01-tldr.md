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
5. **The proposer calls `respond(request_id, signature)`** back on-chain. This either (a) emits `SignatureResponded` for the caller to observe, or (b) on NEAR, wakes the caller's yielded Promise so the signature is returned as a normal function result.

## Important consequences of this design

There is **no direct HTTP endpoint** to request a signature on the MPC node — you always go via a signing contract on some chain. The node's web server ([chain-signatures/node/src/web/mod.rs:77-96](../../chain-signatures/node/src/web/mod.rs#L77-L96)) only exposes health/state/metrics and inter-node `/msg` + `/sync` RPCs. The closest thing to a "direct call" is bringing up the NEAR sandbox and calling the MPC contract there, since NEAR uses Promise yield/resume and returns the signature as a normal function result — see [§6.7](06-local-bootstrap.md#67-getting-a-signature-directly--the-minimum-viable-path).

Membership, threshold, and config are owned by the **NEAR orchestrating contract**. The Ethereum and Solana signing contracts carry **no membership state** — they are pure event plumbing and anyone may call `respond()` on them ([ChainSignatures.sol:69-78](../../chain-signatures/contract-eth/contracts/ChainSignatures.sol#L69-L78): "Any address can emit this event. Clients should always verify the validity of the signature."). Caveat emptor for SDK work: on non-NEAR chains, your SDK must verify the returned signature itself.

## How to use this guide

Skim [§2 Architecture](02-architecture.md) and [§3 Key files](03-key-files.md). Do [Experiment 1](08-experiments.md#experiment-1--trace-one-signrespond-cycle-on-paper) with the guide open. Then read [§4 Event flow](04-event-flow.md) and [§5 Indexers](05-indexers.md) before attempting hands-on work. [§6 Bootstrap](06-local-bootstrap.md) is the spine. [§7 New chains](07-extending-chains.md) is later reading.

---

[← Index](README.md) · Next: [§2 Architecture →](02-architecture.md)
