# §8 — Experiments

[← §7 Extending](07-extending-chains.md) · [Index](README.md) · Next: [§9 Glossary →](09-glossary.md)

---

Progression: 1-2 reading, 3-4 hybrid, 5-8 hands-on.

## Experiment 1 — Trace one sign→respond cycle on paper

**Goal.** Build a mental map of how a Solana sign request becomes a Solana signature response, without running anything.

**Prerequisites.** Repo cloned. This guide open.

**Steps.**
1. Open [integration-tests/tests/cases/solana.rs:7-42](../../integration-tests/tests/cases/solana.rs#L7-L42). Read end-to-end.
2. Follow `cluster.sign().solana()` into [integration-tests/src/actions/sign.rs](../../integration-tests/src/actions/sign.rs). Find `SolSignAction::execute` (~line 414 per research).
3. Follow the instruction construction into [integration-tests/src/containers.rs](../../integration-tests/src/containers.rs) `Solana::sign` (~line 940). Note the discriminator `[5, 221, 155, 46, 237, 91, 28, 236]` — first 8 bytes of SHA256("global:sign").
4. Open [contract-sol/src/lib.rs:63-87](../../chain-signatures/contract-sol/src/lib.rs#L63-L87). Read the `sign` function and the emitted event struct at [lib.rs:206-219](../../chain-signatures/contract-sol/src/lib.rs#L206-L219).
5. Open [chain-signatures/node/src/indexer_sol.rs:146-217](../../chain-signatures/node/src/indexer_sol.rs#L146-L217). Read `generate_request_id` and `generate_sign_request`.
6. Open [chain-signatures/node/src/rpc.rs](../../chain-signatures/node/src/rpc.rs). Search for `try_publish_sol`. Read the function.
7. Come back to the test: [integration-tests/tests/cases/solana.rs:22-35](../../integration-tests/tests/cases/solana.rs#L22-L35) — note how verification uses the **derived** public key, not the root.

**What to observe.** The request id computation is identical in the indexer and what a JS/TS SDK would have to reproduce off-chain. The derivation `derive_key(root_pk, derive_epsilon_sol(key_version, sender, path))` is the formula an SDK user must re-implement to know their Sig.Network address.

**Verification.** You can state in one sentence, without looking, what every field of `SignatureRequestedEvent` is for and who fills it in.

**What this teaches.** The chain-side surface. Nothing past the indexer is integrator-facing.

## Experiment 2 — Diff the three contract surfaces

**Goal.** Internalize what's common across NEAR/Ethereum/Solana and what's chain-specific. Will pay off when you're writing cross-chain SDK docs.

**Prerequisites.** Experiment 1 done.

**Steps.**
1. Read [contract/src/lib.rs:128-189](../../chain-signatures/contract/src/lib.rs#L128-L189) (NEAR `sign`), then :248-292 (NEAR `respond`).
2. Read [ChainSignatures.sol:114-142](../../chain-signatures/contract-eth/contracts/ChainSignatures.sol#L114-L142) (Eth `sign` + `respond`).
3. Read [contract-sol/src/lib.rs:26-87](../../chain-signatures/contract-sol/src/lib.rs#L26-L87) (Solana `sign` + `respond`).

**What to observe.** NEAR validates substantially more than the others (deposit amount, gas, pending-request cap, duplicate `SignId`). Ethereum validates only the deposit. Solana's current minimal implementation validates even less. All three emit the **same semantic event** but the return mechanism differs: NEAR returns directly via Promise; Eth/Solana return via event that the caller must listen to.

**Verification.** You can answer: "which chain's `respond()` verifies the signature cryptographically before emitting?" (Answer: NEAR does — [contract/src/lib.rs:278-286](../../chain-signatures/contract/src/lib.rs#L278-L286). Ethereum and Solana emit blindly; it is the caller's job to verify.)

**What this teaches.** Why your SDK must verify signatures client-side on Ethereum/Solana, and why SDK error surfaces will look quite different across chains.

## Experiment 3 — Reproduce a request_id off-chain

**Goal.** Produce the same 32-byte id the indexer will compute for a given request, in a throwaway script.

**Prerequisites.** A JavaScript / Python / Rust scratchpad. `keccak256` and ABI-encoding primitives handy (e.g. `ethers.utils.solidityPack` + `keccak256`, or `alloy`, or `web3.utils`).

**Steps.**
1. Read [chain-signatures/node/src/indexer_sol.rs:146-162](../../chain-signatures/node/src/indexer_sol.rs#L146-L162). Note the exact encoding order: `Token::String(sender), Token::Bytes(payload), Token::String(path), Token::Uint(key_version), Token::String(chain_id), Token::String(algo), Token::String(dest), Token::String(params)`. `Token::String` + `Token::Bytes` → ethabi dynamic types.
2. Write a function `computeRequestId(sender, payload, path, keyVersion, chainId, algo, dest, params) -> bytes32` in your language of choice.
3. Test it against a request you submit in Experiment 6.

**What to observe.** The ordering of fields and types is load-bearing. Ethereum's integration tests already do this in [integration-tests/tests/cases/ethereum.rs](../../integration-tests/tests/cases/ethereum.rs) ~line 45-66 per research — good cross-reference.

**Verification.** Your off-chain id matches the `requestId` field on the `SignatureResponded` event for the request you submitted.

**What this teaches.** Your SDK will need this exact computation to correlate responses to requests — and to deduplicate.

## Experiment 4 — Start only the dependency services

**Goal.** See the redis+sandbox infra come up, without the MPC nodes on top. Lets you poke the NEAR sandbox directly.

**Prerequisites.** Docker running. `docker pull redis:7.4.2` done.

**Steps.**
```bash
cd integration-tests
cargo run -- dep-services
```

Leave it running. In another terminal:
```bash
docker ps
```

Find the sandbox and redis containers. Hit the sandbox RPC:
```bash
curl -X POST http://localhost:<sandbox-port> \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":"1","method":"status","params":[]}' | jq .
```

**What to observe.** A running near sandbox RPC and a redis port both reachable from the host. Redis is empty (no node has connected yet).

**Verification.** `status` returns a JSON-RPC response showing chain_id `localnet` (or similar).

**What this teaches.** The base stack required by any MPC node. If you ever see a node fail to start, the problem is almost always redis unreachable or NEAR RPC unreachable.

## Experiment 5 — Boot a full cluster and poke the nodes

**Goal.** See MPC nodes running, confirm they're in the `Running` protocol state, and call their HTTP endpoints.

**Prerequisites.** Experiment 4 works.

**Steps.**
```bash
cd integration-tests
cargo run -- setup-env --nodes 3 --threshold 2
```

In a second terminal:
```bash
docker ps | grep mpc
# identify the three node containers and the port mapped to 3000 on each
for port in <port-a> <port-b> <port-c>; do
  echo "=== node on $port ==="
  curl -s http://127.0.0.1:$port/state | jq '{ type: .type, triples: .triple_count, presignatures: .presignature_count, latest_block: .latest_block_height }'
done
curl -s http://127.0.0.1:<port-a>/metrics | grep -E '^(mpc|sign_|presignature_|triple_)' | head -20
```

**What to observe.** All three nodes return `type: "running"`. Triple and presignature counts should be non-zero (pre-seeded by the fixture). Metrics show protocol counters incrementing.

**Verification.** Healthy `/state` response on every node, and metrics not stuck at zero.

**What this teaches.** Operational visibility. This is the same HTTP surface prod nodes expose. Your SDK will never call it, but you will call it constantly when demoing or debugging.

## Experiment 6 — Run `test_solana_signature_basic` end-to-end with containers kept

**Goal.** Observe one complete sign→respond cycle with logs and see the on-chain events.

**Prerequisites.** Experiment 5 works; `solana-test-validator` installed locally.

**Steps.**
```bash
TESTCONTAINERS=keep cargo test -p integration-tests test_solana_signature_basic -- --nocapture
```

Inspect afterwards:
```bash
docker ps
docker logs <any-mpc-node-container> 2>&1 | grep -E 'solana|sign_id|respond'
# find the solana validator RPC port from the log noise above, then:
curl http://127.0.0.1:<solana-port> -X POST -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"getHealth"}'
```

**What to observe.**
- In the node logs: a "found solana event" line (from [indexer_sol.rs:165](../../chain-signatures/node/src/indexer_sol.rs#L165)), followed by protocol progress, then a "try_publish_sol" publish line.
- Test output says OK — signature verified against the derived public key.

**Verification.** Test passes; logs show the sign-request log line and a subsequent respond publish.

**What this teaches.** The real cycle in operation, not in your head. You've now seen what your SDK will need to orchestrate externally.

## Experiment 7 — Submit a sign request to the NEAR sandbox by hand

**Goal.** Submit a signature request without using the integration test harness.

**Prerequisites.** Cluster from Experiment 5 running. `near-cli-rs` or `near-workspaces` installed (or use curl against the sandbox RPC directly).

**Steps.**
1. From the `setup-env` output, note the MPC contract id (typically `v1.signer.test.near` or similar in the sandbox `[UNVERIFIED]`) and the sandbox RPC URL.
2. Generate example commands: `cd integration-tests && cargo run -- contract-commands` — this writes [chain-signatures/contract/EXAMPLE.md](../../chain-signatures/contract/EXAMPLE.md) with ready-to-run near-cli-rs invocations.
3. Open that file and run the `sign` example, substituting your sandbox RPC URL and an account you control on the sandbox.
4. The call will hang for a few seconds; the signature will come back as the return value.

**What to observe.** The call literally blocks — NEAR's yield/resume gives you a synchronous experience even though the node does the work asynchronously. Compare to Ethereum/Solana where you would have had to subscribe to an event.

**Verification.** `near view` the returned signature; run `check_ec_signature` in a scratchpad (see [integration-tests/tests/cases/solana.rs:33-35](../../integration-tests/tests/cases/solana.rs#L33-L35)) against the payload you sent and the derived public key.

**What this teaches.** The NEAR-specific UX is markedly friendlier than event-based flows. When you write SDK guides, NEAR is the easy-mode starting point; Ethereum and Solana need explicit listener state machines.

## Experiment 8 — Break something on purpose

**Goal.** Exercise the validation path. A silent failure is the most common pain your SDK users will hit.

**Prerequisites.** Cluster running.

**Steps.**
1. Submit a `sign` request with `key_version = 999` (intentionally too high). On NEAR, the contract rejects you immediately with `UnsupportedKeyVersion`. On Ethereum/Solana, **the contract will accept and emit the event**, then the indexer will drop it silently ([indexer_sol.rs:171-174](../../chain-signatures/node/src/indexer_sol.rs#L171-L174)).
2. Submit a `sign` with `deposit = 0` on Ethereum. The contract reverts with "Insufficient deposit" ([ChainSignatures.sol:115](../../chain-signatures/contract-eth/contracts/ChainSignatures.sol#L115)).
3. Submit a `sign` with a payload that is not a valid scalar (e.g. all `0xff`). Indexer rejects silently ([indexer_sol.rs:176-187](../../chain-signatures/node/src/indexer_sol.rs#L176-L187)).

For the silent-failure cases, check the node logs:
```bash
docker logs <node-container> 2>&1 | grep -iE 'warn|reject|unsupported'
```

**What to observe.** The indexer WARN-level logs are the only artifact for indexer-side rejection. On Ethereum/Solana, the user has no on-chain signal that anything went wrong — they just never get a response.

**Verification.** You can see the WARN logs; no `SignatureResponded` event is emitted for these requests.

**What this teaches.** Your SDK needs to validate client-side what the indexer validates: `key_version`, payload-as-scalar, payload-within-curve-order. Otherwise your users will wait forever for a response that isn't coming.

---

[← §7 Extending](07-extending-chains.md) · [Index](README.md) · Next: [§9 Glossary →](09-glossary.md)
