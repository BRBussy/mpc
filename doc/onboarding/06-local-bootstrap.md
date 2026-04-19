# §6 — Local bootstrap + direct MPC calls (deep)

[← §5 Indexers](05-indexers.md) · [Index](README.md) · Next: [§7 Extending →](07-extending-chains.md)

---

This is the hands-on spine. Everything else in the guide is background for this section.

## 6.1 Prereqs

From [integration-tests/README.md:5-11](../../integration-tests/README.md#L5-L11):

```bash
docker pull redis:7.4.2
```

On macOS you may need to symlink the docker socket:

```bash
sudo ln -s $HOME/.docker/run/docker.sock /var/run/docker.sock
```

Solana integration tests require `solana-test-validator` installed locally (not a docker container) — [containers.rs ~lines 595-700](../../integration-tests/src/containers.rs).

`[UNVERIFIED]` Ethereum tests require Foundry's `anvil` available either as the docker image `ghcr.io/foundry-rs/foundry:nightly` or locally.

Rust toolchain: as pinned by `rust-toolchain.toml` / the workspace. Run `./setup.sh` if unsure.

## 6.2 Option A — full cluster with one command

```bash
cd integration-tests
cargo run -- setup-env --nodes 3 --threshold 2
```

What you get ([integration-tests/src/main.rs:54-116](../../integration-tests/src/main.rs#L54-L116)):

- Docker bridge network named `mpc_it_network` (or per-instance `[UNVERIFIED]`).
- `redis:7.4.2` container.
- NEAR sandbox (`near-workspaces` spawns the `ghcr.io/near/sandbox:latest` image).
- 3 MPC nodes on docker, each listening on port 3000 in-container (mapped to a random host port).
- Deployed MPC contract on the sandbox with pre-generated key material baked in via [integration-tests/src/mpc_fixture/3_nodes_2_threshold.json](../../integration-tests/src/mpc_fixture/3_nodes_2_threshold.json). This shortcut shaves ~20s of key-gen off every boot ([cluster/spawner.rs:26-103](../../integration-tests/src/cluster/spawner.rs#L26-L103) `[UNVERIFIED]`).

The process prints node URLs, NEAR account IDs, secret keys, public keys, and then blocks on Ctrl-C. Leave it running.

**To add Ethereum**: [integration-tests/src/main.rs:26-42](../../integration-tests/src/main.rs#L26-L42) already accepts eth flags; pass `--eth-execution-rpc-http-url http://localhost:8545` etc. and run your own anvil. Same idea with Solana — pass `--sol-*` flags `[UNVERIFIED]` (flags exist in node cli; confirm they're threaded through `setup-env` before relying on them).

## 6.3 Option B — dep services only (BYO node)

```bash
cd integration-tests
cargo run -- dep-services
```

Starts redis + sandbox without MPC nodes ([main.rs:117-126](../../integration-tests/src/main.rs#L117-L126)). Then start a node manually — gives you a debuggable process with full logs. The README suggests using this pattern for fast iteration ([integration-tests/README.md:118-126](../../integration-tests/README.md#L118-L126)).

## 6.4 Option C — run an existing test and freeze the containers

```bash
TESTCONTAINERS=keep cargo test -p integration-tests test_solana_signature_basic -- --nocapture
```

After the test ends, containers stick around. Use `docker ps` to find them, then:

```bash
docker ps | grep mpc
# copy the host port mapped to container port 3000 for a node
curl http://127.0.0.1:<mapped-port>/state | jq .
curl http://127.0.0.1:<mapped-port>/status | jq .
curl http://127.0.0.1:<mapped-port>/metrics | head -40
docker logs <container-id>
```

Cleanup when done:

```bash
docker ps -aq --filter "name=mpc" | xargs docker rm -f
```

Full playbook: [integration-tests/README.md:89-111](../../integration-tests/README.md#L89-L111).

## 6.5 Option D — in-process MPC fixture (fastest, for SDK unit tests)

[integration-tests/src/mpc_fixture/builder.rs](../../integration-tests/src/mpc_fixture/builder.rs) constructs an MPC network inside one process (no docker at all). The recent commit `c59ba07` moved the fixture logic into `build()` and made `threshold` required. Imports needed are at [builder.rs:4-38](../../integration-tests/src/mpc_fixture/builder.rs#L4-L38). Build pattern: `MpcFixtureBuilder::new(...).threshold(...).use_preshared_key(true).build()` `[UNVERIFIED]` — read [mpc_fixture/mod.rs](../../integration-tests/src/mpc_fixture/mod.rs) for the public API.

## 6.6 What the node actually exposes — the HTTP surface

From [chain-signatures/node/src/web/mod.rs:77-96](../../chain-signatures/node/src/web/mod.rs#L77-L96):

| Method + path | Purpose |
|---|---|
| `GET /` | Healthcheck — 200 OK only. |
| `GET /state` | Node's view of the protocol state + triple/presignature counts. Useful for integrators to confirm the node is `Running` before sending test load. |
| `GET /status` | Node status + protocol version. |
| `GET /metrics` | Prometheus scrape endpoint. |
| `GET /checkpoint` | Backlog checkpoint — for ops/debugging. |
| `GET /debug` | Debug HTML page (requires `debug-page` feature). |
| `POST /msg` | **Node-to-node** encrypted protocol messages (HPKE + CBOR). Not for integrators. |
| `POST /sync` | **Node-to-node** state sync (up to 20 MB CBOR payload). Not for integrators. |

**There is no `POST /sign` or similar.** Signature requests always go through a chain contract. This is intentional: signing is gated by chain-level fees / finality / replay-protection, none of which could be meaningfully enforced by a bare HTTP endpoint.

## 6.7 Getting a signature "directly" — the minimum viable path

The closest thing to "call the MPC directly" is: stand up the cluster, then call the NEAR sandbox's MPC contract from a NEAR CLI script. That contract has **no indexer in the loop** — the node reads pending requests from the contract's state directly ([indexer.rs:85-95](../../chain-signatures/node/src/indexer.rs#L85-L95)). Steps:

1. Bring up the cluster: `cargo run -- setup-env --nodes 3 --threshold 2` — note the NEAR sandbox RPC address in the output.
2. Use near-cli (or near-workspaces, or the raw RPC) to call `sign` on the MPC contract account. Example script body: see generated [chain-signatures/contract/EXAMPLE.md](../../chain-signatures/contract/EXAMPLE.md), regenerate via `cargo run -- contract-commands` ([integration-tests/src/main.rs:127-132](../../integration-tests/src/main.rs#L127-L132)).
3. The call will **hang** until the network responds — that's the promise yield. You receive the signature as the result of the call. No event listening required on NEAR.

Calling contracts on other chains requires also standing up that chain (anvil for Ethereum, solana-test-validator for Solana) and deploying the signing contract there. The cleanest reference is the test harness itself — `Cluster::sign().solana()` in [integration-tests/src/actions/sign.rs](../../integration-tests/src/actions/sign.rs) is exactly the shape an SDK would take.

## 6.8 Node CLI flags (when running by hand)

From [chain-signatures/node/src/cli.rs:33-98](../../chain-signatures/node/src/cli.rs#L33-L98):

Required for any startup:
- `--near-rpc` / `MPC_NEAR_RPC`
- `--mpc-contract-id` / `MPC_CONTRACT_ID`
- `--account-id` / `MPC_ACCOUNT_ID` (this node's NEAR account)
- `--account-sk` / `MPC_ACCOUNT_SK` (ed25519 secret key)
- `--cipher-sk` / `MPC_CIPHER_SK` (hex-encoded 32-byte HPKE key)
- Redis URL via `storage::Options` flags
- GCP project id for secret-share storage (integration tests use a local path override)

Optional:
- `--web-port` / `MPC_WEB_PORT` (default 3000) — [cli.rs:52-56](../../chain-signatures/node/src/cli.rs#L52-L56)
- `--sign-sk` (derived from account_sk if absent)
- `--my-address` / `MPC_LOCAL_ADDRESS` — URL peers use to reach this node ([cli.rs:75-81](../../chain-signatures/node/src/cli.rs#L75-L81))
- `--override-config` / `MPC_OVERRIDE_CONFIG` — local overrides of contract config
- All `--eth-*` / `--sol-*` / `--hydration-*` groups (see [§5](05-indexers.md)).

External dependencies that must be reachable before the node starts:

1. Redis — or the node loops on storage init.
2. NEAR RPC — always required (the NEAR contract is authoritative for network state/membership).
3. Whatever chain's RPC if its indexer is enabled.
4. GCP Application Default Credentials — only if using Secret Manager for the key share (prod).

## 6.9 Logging & tracing

Three modes ([integration-tests/README.md:22-48](../../integration-tests/README.md#L22-L48)):

- **fmt** — default, human-readable console.
- **OTLP** — point at a collector (e.g. Jaeger) via `MPC_OTLP_ENDPOINT` (default `http://localhost:4318`) and `MPC_OPENTELEMETRY_LEVEL` (`debug` / `info`). Useful to see the per-request trace flow.
- **Stackdriver** — GCP-only.

For a running test with verbose logs: `RUST_BACKTRACE=full RUST_LOG=debug cargo test test_name -- --nocapture`.

---

[← §5 Indexers](05-indexers.md) · [Index](README.md) · Next: [§7 Extending →](07-extending-chains.md)
