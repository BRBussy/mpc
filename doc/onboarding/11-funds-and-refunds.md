# §11 — Funds & refunds

[← §10 Potential risks](10-potential-risks.md) · [Index](README.md)

---

> Who pays, who holds the money while a request is in flight, and who gets a refund (or doesn't) on each outcome. Consolidates material otherwise scattered across [§1](01-tldr.md), [§4.3](04-event-flow.md#43-request-validation--what-gates-a-request-before-nodes-participate), and [§10.10](10-potential-risks.md#1010-what-happens-to-the-users-deposit-on-failure).

## 11.1 Summary

| Chain | User pays on `sign()` | Who holds it in flight | Refunded on failure? |
|---|---|---|---|
| **NEAR** | Dynamic deposit (1 yocto → 1 NEAR based on system load) + gas ≥ 50 Tgas | Locked in NEAR contract state ([contract/src/lib.rs:178-180](../../chain-signatures/contract/src/lib.rs#L178-L180)) | **Yes — full refund** via `refund_on_fail` when the yield times out ([contract/src/lib.rs:812-817](../../chain-signatures/contract/src/lib.rs#L812-L817), [:833-856](../../chain-signatures/contract/src/lib.rs#L833-L856)). Overpayment is also refunded on success ([:819-828](../../chain-signatures/contract/src/lib.rs#L819-L828)). |
| **Ethereum** | `msg.value ≥ signatureDeposit` + gas (contract reverts if short, [ChainSignatures.sol:115](../../chain-signatures/contract-eth/contracts/ChainSignatures.sol#L115)) | Sits in contract balance | **No.** Deposit stays forever in the contract balance. Admin-withdrawable only ([ChainSignatures.sol:181-195](../../chain-signatures/contract-eth/contracts/ChainSignatures.sol#L181-L195)). |
| **Solana** | Just the base Solana tx fee (~5000 lamports). The `signature_deposit` field exists but is **not enforced** by the contract ([contract-sol/src/lib.rs:63-87](../../chain-signatures/contract-sol/src/lib.rs#L63-L87)). | Nothing transferred; the "deposit" in the emitted event is a config lookup. | **N/A — nothing to refund.** |
| **Hydration** | `[UNVERIFIED]` — pallet is outside this repo. The indexer rejects `deposit == 0` and requires `key_version ≤ LATEST`, so *some* deposit flow likely exists, but semantics aren't visible here. | `[UNVERIFIED]` | `[UNVERIFIED]` |

## 11.2 NEAR — the only chain with a working refund model

NEAR is the only chain in the network that:

1. Enforces a deposit at the contract ([contract/src/lib.rs:144-152](../../chain-signatures/contract/src/lib.rs#L144-L152)).
2. Scales the deposit with network load ([contract/src/lib.rs:231-242](../../chain-signatures/contract/src/lib.rs#L231-L242) — ranges from 1 yocto at quiet times to 1 NEAR under heavy load).
3. Refunds on failure via a runtime-enforced path ([contract/src/lib.rs:833-856](../../chain-signatures/contract/src/lib.rs#L833-L856)).
4. Refunds overpayment on success ([contract/src/lib.rs:819-828](../../chain-signatures/contract/src/lib.rs#L819-L828)).

Concretely, on NEAR:

- **Success** → Signature returned to caller as ordinary `sign()` return value. `refund_on_success` runs the overpayment-vs-required delta back to the requester.
- **Network timeout** (yield expires, ~200 blocks / ~4 min) → `clear_state_on_finish` fires with `Err(_)`, `refund_on_fail` returns the **full deposit** to the requester, caller's tx returns `SignError::Timeout`.
- **Contract-side validation reject** (gas too low, key_version unsupported, deposit too low, request collision, pending-request cap) → tx reverts, deposit never leaves the requester.

Gas is a separate concern — unused gas is refunded by NEAR's runtime normally, which is standard NEAR semantics and not specific to this contract.

## 11.3 Ethereum — deposit permanently in contract balance

On Ethereum, the `sign(SignRequest)` function is `payable`. `msg.value >= signatureDeposit` or the call reverts ([ChainSignatures.sol:115](../../chain-signatures/contract-eth/contracts/ChainSignatures.sol#L115)). But:

- **There is no refund path for the deposit.** No `refund()` function, no auto-refund on `respondError`, no time-bounded escrow.
- Deposits accumulate in the contract balance.
- Only the `DEFAULT_ADMIN_ROLE` (granted to `_mpc_network` at construction — [ChainSignatures.sol:105-108](../../chain-signatures/contract-eth/contracts/ChainSignatures.sol#L105-L108)) can withdraw via `withdraw(_amount, _receiver)` ([ChainSignatures.sol:181-195](../../chain-signatures/contract-eth/contracts/ChainSignatures.sol#L181-L195)).

From a user's perspective on Ethereum:

- **Success** → They paid the deposit, got the signature via event, no refund. Deposit is permanently gone.
- **Failure** (any kind — indexer dropped, proposer crashed, `respondError` emitted) → They paid the deposit, never got the signature, no refund. Deposit is permanently gone.

This makes the Ethereum deposit effectively a **fee**, not a refundable bond — and the fee is paid regardless of whether the service completed. That's a very different economic model from NEAR's.

Also worth knowing: the deposit is a **fixed** value ([ChainSignatures.sol:41, :170-174](../../chain-signatures/contract-eth/contracts/ChainSignatures.sol#L170-L174)) adjustable only by the admin. No load-based scaling.

## 11.4 Solana — the deposit is decorative

This is worth stating bluntly because it's a surprising fact the SDK must communicate correctly:

**The Solana `sign()` handler does not take any payment from the caller beyond the base Solana transaction fee.**

The `signature_deposit: u64` field in `ProgramState` ([contract-sol/src/lib.rs:138](../../chain-signatures/contract-sol/src/lib.rs#L138)) is set by the admin at `initialize()` and never read again — except to be copied into the emitted event ([contract-sol/src/lib.rs:77](../../chain-signatures/contract-sol/src/lib.rs#L77)). The `sign` handler:

- Does not call `system_program::transfer`.
- Does not adjust lamport balances (`**requester.try_borrow_mut_lamports()?` etc.).
- Does not read the instruction sysvar to verify a sibling transfer.
- Does not assert `requester.lamports() >= expected`.

Net effect: the `deposit` field in the emitted `SignatureRequestedEvent` is a **static lie** — it reflects config, not caller behaviour. The MPC indexer's downstream `deposit > 0` check ([indexer_sol.rs:166-169](../../chain-signatures/node/src/indexer_sol.rs#L166-L169)) lets every event through as long as the admin-configured value is non-zero.

**Implication for spam resistance on Solana:** effectively none. An attacker can invoke `sign()` in a tight loop paying only base tx fees. Rate limiting, if needed, has to happen somewhere else.

**Implication for refunds on Solana:** zero — there's nothing to refund because nothing was paid.

**How this could be fixed:** either (a) make the Solana contract enforce the transfer in the handler, or (b) have the MPC indexer cross-check the tx for a `system_program::transfer` sibling instruction with amount ≥ `deposit` to a designated recipient, before enqueueing the sign request. Neither is implemented today.

## 11.5 Hydration — `[UNVERIFIED]`

The Hydration pallet lives outside this repo. Indexer behaviour ([indexer_hydration.rs](../../chain-signatures/node/src/indexer_hydration.rs)) rejects events with `deposit == 0`, which implies the pallet carries a non-zero deposit in its events. But whether that deposit is actually transferred, and whether there's a refund path on the Hydration side, can't be determined from this repo alone. Treat as an open question when writing Hydration SDK docs.

## 11.6 Cross-chain gas in bidirectional flows

Flow C (Solana and Hydration's `sign_bidirectional`) is the trickiest case because **two chains' fees are in play**:

- **Origin-chain fees** — the user pays these directly when calling `sign_bidirectional()`. For Solana: base tx fee + (unenforced) deposit. For Hydration: `[UNVERIFIED]`.
- **Destination-chain fees** — the MPC network must pay these when broadcasting the signed tx on the destination chain (step 5 in the Flow C diagram in [§2.2](02-architecture.md#flow-c--bi-directional-cross-chain-sign--broadcast--respond)). `[UNVERIFIED]` exactly which account pays: the code uses per-chain client wallets configured via `--eth-account-sk`, `--sol-account-sk` etc. ([§6.8](06-local-bootstrap.md#68-node-cli-flags-when-running-by-hand)), which strongly suggests **the proposer node's own operational funds pay destination-chain gas** (I did not trace this to a specific broadcast call site).

If that inference is correct, the economic picture of a bidirectional flow is:

- User pays origin-chain fee (small).
- User's deposit is not transferred (Solana) or lives in an origin-chain contract (Hydration `[UNVERIFIED]`).
- MPC node operators absorb all destination-chain gas.
- On failure, MPC node operators still pay whatever destination-chain gas was consumed even if the destination tx reverted.

This is a significant liability for node operators and not sustainable at scale. An SDK that makes heavy use of bidirectional today is effectively being subsidised by the node operators. **Confirm this with the operator team before building any high-volume bidirectional integration** — it may be the reason bidirectional is currently flagged as experimental. Note also that this is a separate concern from the "responses don't invoke the caller contract" gap in [§10.9](10-potential-risks.md#109-bidirectional-responses-dont-actually-invoke-the-caller); they stack on the same flow but are independent design issues.

## 11.7 What a coherent fee/refund model might look like

These are observations, not proposals. Noting them because they frame how incomplete the current picture is:

- **Per-chain escrow.** Treat the deposit as a refundable bond on every chain. Require the contract itself to hold funds in escrow and release them to the requester on failure or to a network treasury on success. NEAR has this; Ethereum and Solana do not.
- **MPC-side transfer verification for Solana.** If the Solana contract can't easily be changed to enforce the transfer (compatibility, deployment friction), the indexer can gate enqueueing on observing a sibling `transfer` instruction of the required amount to a designated recipient. One-file node change.
- **Destination-chain gas reimbursement.** Either the user escrows an over-estimate of destination-chain gas at origin (refunded on success/failure), or a network treasury bills the caller out-of-band. Without this, bidirectional's economics don't close.
- **Fee disclosure in the contract view.** Every chain should expose a view function returning the current required deposit so SDKs don't guess. NEAR has `experimental_signature_deposit()` already ([contract/src/lib.rs:231-242](../../chain-signatures/contract/src/lib.rs#L231-L242)); Ethereum has `getSignatureDeposit()` ([ChainSignatures.sol:162-164](../../chain-signatures/contract-eth/contracts/ChainSignatures.sol#L162-L164)). Solana does not.

## 11.8 SDK implications

1. **Don't tell users "attach a deposit" on Solana.** The deposit parameter doesn't exist in the Solana `sign` instruction args — it's a contract-internal config. Users attach nothing beyond tx fees. Documenting a "deposit" on Solana as a user-facing parameter would be misleading.
2. **On NEAR, query `experimental_signature_deposit()` before each sign.** The deposit is dynamic; quoting a stale value will cause reverts under load.
3. **On Ethereum, query `getSignatureDeposit()` at init and on 4xx-like errors** to pick up admin changes. And make clear in docs that **the deposit is not refundable** — users should treat it as a fee.
4. **On bidirectional, warn users that destination-chain execution is best-effort** and that there's currently no formal reimbursement path if the operation fails after the destination tx has been mined. Consider requiring a pre-flight simulation on the destination chain to reduce the rate of partial failures.
5. **Do not build financial dApps on Ethereum non-refundable deposits.** A user who submits and doesn't receive a signature has no recourse. If your dApp has high-value requests, consider routing through NEAR (refundable) or adding an application-layer escrow on top of Ethereum.

---

[← §10 Potential risks](10-potential-risks.md) · [Index](README.md)
