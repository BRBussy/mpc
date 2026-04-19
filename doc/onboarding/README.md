# Sig.Network MPC — Onboarding Guide (Integrator View)

> Audience: a partner engineer who will build SDKs, demos, and guides against this network.
> Scope: **integration surface** — events, indexers, contracts, local bootstrap, direct calls.
> Out of scope: threshold-crypto internals (cait-sith math, share derivation).
>
> Source: this guide reads from code in this repo. Every non-trivial claim cites `path:line-range`. Inferred claims are marked `[UNVERIFIED]`.

## How to read this guide

Skim [§1 TL;DR](01-tldr.md) and [§2 Architecture](02-architecture.md) first. Walk [§3 Key files](03-key-files.md) with the repo open. Do [Experiment 1](08-experiments.md#experiment-1--trace-one-signrespond-cycle-on-paper) with this guide beside you. Then read [§4 Event flow](04-event-flow.md) and [§5 Indexers](05-indexers.md) before attempting the hands-on experiments. [§6 Local bootstrap](06-local-bootstrap.md) is the hands-on spine — do not skip. [§7 New chains](07-extending-chains.md) is optional.

## Table of contents

1. [TL;DR](01-tldr.md) — what this repo is, the core data flow, how to use the guide.
2. [Architecture overview](02-architecture.md) — components, responsibilities, and the mermaid data-flow diagram.
3. [Key files tour](03-key-files.md) — ~20 files ranked by integrator relevance.
4. [Event flow & request validation](04-event-flow.md) — event shapes, request_id formula, the "bi-directional" pattern clarified, validation checkpoints, finality.
5. [Indexing across chains](05-indexers.md) — Ethereum / Solana / Hydration / NEAR in detail, with config flags.
6. [Local bootstrap + direct MPC calls](06-local-bootstrap.md) — four ways to stand it up locally, the HTTP surface, how to call the MPC "directly."
7. [Extending to a new chain](07-extending-chains.md) — pointers only, for later.
8. [Experiments](08-experiments.md) — 8 structured experiments (3 reading, 5 hands-on).
9. [Glossary](09-glossary.md) — terms whose meaning in this codebase differs from public usage.

## Further reading in this repo

- [README.md](../../README.md) — top-level starting point.
- [doc/ARCHITECTURE.md](../ARCHITECTURE.md) — the official architecture view.
- [doc/ACCOUNT_DERIVATION.md](../ACCOUNT_DERIVATION.md) — **essential** — full derivation spec.
- [doc/SCALING_AND_SECURITY.md](../SCALING_AND_SECURITY.md) — current operating numbers.
- [doc/mpc_node_specification.md](../mpc_node_specification.md) — internal distributed-algorithm spec. Skip unless you're debugging a node.
- [infra/README.md](../../infra/README.md) — production/partner deployment context.
- [integration-tests/README.md](../../integration-tests/README.md) — authoritative local-run reference.
- [chain-signatures/contract/EXAMPLE.md](../../chain-signatures/contract/EXAMPLE.md) — generated near-cli-rs examples, regenerate with `cd integration-tests && cargo run -- contract-commands`.

---

Next: [§1 TL;DR →](01-tldr.md)
