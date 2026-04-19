# §6 — Local bootstrap + direct MPC calls (deep)

[← §5 Indexers](05-indexers.md) · [Index](README.md) · Next: [§7 Extending →](07-extending-chains.md)

---

This is the hands-on spine. Everything else in the guide is background for this section.

## 6.1 Prerequisites

| Prerequisite | Version | When needed | Install |
|---|---|---|---|
| Docker (Engine or Desktop) | any recent | Always — Redis, NEAR sandbox, optional anvil | [docker.com](https://www.docker.com/get-started/) |
| `redis:7.4.2` image | exactly 7.4.2 | Always | `docker pull redis:7.4.2` |
| `rustup` | latest | Always | [rustup.rs](https://rustup.rs) |
| Rust stable toolchain | per [rust-toolchain.toml](../../rust-toolchain.toml) | Always — node binary, test harness | `rustup show` (auto-installed on first cargo invocation in the repo) |
| Rust `1.81.0` toolchain | exactly 1.81.0 | Always — NEAR contract WASM build | `rustup install 1.81.0` |
| `wasm32-unknown-unknown` target | on 1.81.0 | Always — NEAR contract | `rustup target add wasm32-unknown-unknown --toolchain 1.81.0` |
| `solana-test-validator` | latest | Only when testing the Solana flow | [Solana CLI tools](https://solana.com/docs/intro/installation) — installed as a native binary on PATH |
| Foundry (`anvil`) | nightly | Only when testing the Ethereum flow | [getfoundry.sh](https://getfoundry.sh) — `[UNVERIFIED]` whether the harness uses the local install or the `ghcr.io/foundry-rs/foundry:nightly` container |
| `near/sandbox:latest` image | latest | Always | Pulled automatically by `near-workspaces` on first use — no manual step |

### Things that are *not* a prereq (despite popular belief)

- **Prebuilding the contract or node binary.** Both are built automatically by a cargo runner hook (`./setup.sh`) that fires on every `cargo run -p integration-tests` or `cargo test -p integration-tests`. See [§6.3](#63-setup-type-full-cluster) for details.
- **Symlinking the docker socket on macOS.** The harness has a built-in fallback for Docker Desktop's socket location. Only needed in edge cases — see [§6.11 Troubleshooting](#611-troubleshooting).
- **A real GCP account.** Integration tests use a local file for secret-share storage, not GCP Secret Manager. The `--gcp-project-id` value is just a label.

## 6.2 Pick your setup type

| Setup type | When to use it | Section |
|---|---|---|
| **Full cluster** | End-to-end experiments against real sandbox chains (NEAR mandatory; Ethereum/Solana optional). What the integration tests themselves use. | [§6.3](#63-setup-type-full-cluster) |
| **Dependencies only (BYO node)** | Debugging a node: run Redis + NEAR sandbox via the harness, run your own `mpc-node` binary manually so you can attach a debugger, stream logs, iterate fast. | [§6.4](#64-setup-type-dependencies-only-byo-node) |
| **Keep-containers from a test run** | Inspect the exact state a specific test produced. Hit endpoints, read logs, reproduce an assertion. | [§6.5](#65-setup-type-keep-containers-from-a-test-run) |
| **In-process MPC fixture** | SDK unit tests / fast iteration. No docker, no chains — an MPC network inside one Rust process. | [§6.6](#66-setup-type-in-process-mpc-fixture) |
| **Node by hand against a real network** | Connect to testnet / mainnet / a custom chain. Raw CLI, no harness. | [§6.7](#67-setup-type-node-by-hand-cli-flags) |

---

## 6.3 Setup type: full cluster

```bash
cd integration-tests
cargo run -- setup-env --nodes 3 --threshold 2
```

The process prints URLs, NEAR account IDs, secret keys, public keys for each node, the sandbox RPC, the Redis address, and then blocks on Ctrl-C. Leave it running.

### There is no `setup-env` config file

That threw me at first too. `setup-env` is a **Rust-defined subcommand**, not a config file. If you grep for `setup-env` or `setup_env` you'll find the actual source of truth is the `Cli` enum at [integration-tests/src/main.rs:17-47](../../integration-tests/src/main.rs#L17-L47):

```rust
#[derive(Parser, Debug)]
enum Cli {
    SetupEnv {
        #[arg(short, long, default_value_t = 3)]
        nodes: usize,
        #[arg(short, long, default_value_t = 2)]
        threshold: usize,
        #[arg(long, default_value = "http://localhost:8545")]
        eth_consensus_rpc_http_url: String,
        ...
    },
    DepServices,
    ContractCommands,
}
```

All flag defaults are hardcoded in the clap annotations. All cluster settings that aren't exposed as flags (network name, GCP project id, env label) are hardcoded in [cluster/spawner.rs:18-20](../../integration-tests/src/cluster/spawner.rs#L18-L20):

```rust
const DOCKER_NETWORK: &str = "mpc_it_network";
const GCP_PROJECT_ID: &str = "multichain-integration";
const ENV: &str = "integration-tests";
```

To change these, edit the source and recompile. There's no YAML / TOML / `.env` for the harness.

### What actually runs

Tracing end-to-end from `cargo run` to "cluster up":

**1. Cargo invokes `./setup.sh` as a cargo runner.** This is wired via [.cargo/config.toml](../../.cargo/config.toml):

```toml
[target.'cfg(not(target = "wasm32-unknown-unknown"))']
runner = "./setup.sh"
```

Every `cargo run -p integration-tests ...` or `cargo test -p integration-tests ...` gets rewritten to `./setup.sh <binary-path> <args...>`. The runner ([setup.sh:11-44](../../setup.sh)) then:

1. Builds the NEAR contract WASM via `./build-contract.sh` (`cargo +1.81.0 build -p mpc-contract --release --target wasm32-unknown-unknown`) → `target/wasm32-unknown-unknown/release/mpc_contract.wasm`.
2. Builds the node binary via `cargo build -p mpc-node --release --features test-feature,debug-page` → `target/release/mpc-node`.
3. Execs the integration-tests binary with the original args.

Env-var knobs for setup.sh:
- `MPC_SETUP_SKIP=1` — skip the prebuild entirely. Faster iteration when you know the artifacts are fresh.
- `MPC_SETUP_ALWAYS=1` — run the prebuild even for non-integration-tests cargo invocations.

**2. [main.rs:53-116](../../integration-tests/src/main.rs#L53-L116) parses args, builds a `NodeConfig`** with `nodes=3, threshold=2`, attaches an `EthConfig` (from the `--eth-*` flags, which have defaults so they're always present), and constructs a `ClusterSpawner::default()`.

**3. `ClusterSpawner::default()` at [spawner.rs:134-168](../../integration-tests/src/cluster/spawner.rs#L134-L168)** sets up:
- A `DockerClient` (bollard connection).
- `release: true` — use `target/release/mpc-node`.
- Pregenerated keys loaded from [mpc_fixture/3_nodes_2_threshold.json](../../integration-tests/src/mpc_fixture/3_nodes_2_threshold.json) if the (nodes, threshold) tuple matches a known fixture. For `(3, 2)` and `(5, 4)` this skips ~20s of live key-generation and starts nodes directly in the `Running` state. For any other tuple you fall through to full keygen.

**4. `.init_network()` creates the docker bridge network** `mpc_it_network`.

**5. `.run()` branches on cargo feature `docker-test`:**

| Feature flag | Mode | Nodes run as | Called function |
|---|---|---|---|
| *(default)* | `host` | **Native processes on your machine** | [lib.rs:573-664](../../integration-tests/src/lib.rs#L573-L664) |
| `--features docker-test` | `docker` | Docker containers | [lib.rs:450-522](../../integration-tests/src/lib.rs#L450-L522) |

`setup-env` uses `host`. So the MPC nodes are plain binaries running on your host — **not** docker containers, even though their dependencies (Redis, NEAR sandbox) are containerised. This is faster to iterate on.

**6. `setup(spawner)` at [lib.rs:329-448](../../integration-tests/src/lib.rs#L329-L448) builds the infra stack** in order:

1. **NEAR sandbox** — `near_workspaces::sandbox()` spawns the `near/sandbox:latest` container. Exposes JSON-RPC on a random host port.
2. **Creates N NEAR accounts** on sandbox (one per MPC node).
3. **Deploys the compiled MPC contract WASM** from `target/wasm32-unknown-unknown/release/mpc_contract.wasm` via `worker.dev_deploy()`. Built by `setup.sh` in step 1.
4. **Redis container** — `redis:7.4.2`. Port 6379 on a random host port.
5. **If `use_ethereum`** — spawns `EthereumSandbox` (anvil), deploys `ChainSignatures.sol` via ethers-rs.
6. **If `cfg.sol` is set** — spawns `solana-test-validator` as a native subprocess, deploys the Solana program from `chain-signatures/contract-sol/artifacts/chain_signatures.so`.
7. **Creates `target/tmp/secrets/`** — where each node's secret key share is written as plain files (not GCP Secret Manager).
8. **If pregenerated keys are enabled** ([lib.rs:407-433](../../integration-tests/src/lib.rs#L407-L433)) — writes each participant's key share into their local `secret_storage` so they come up in `Running` state immediately.

**7. `host(spawner)` spawns the MPC node processes** in parallel. Each is `mpc-node start ...` with env vars pointing at the sandbox RPC, Redis URL, local secret path, web port.

**8. Registers all nodes as participants** on the MPC contract via `init_running` (skips keygen, uses the pregenerated public key) or `init` (triggers keygen).

**9. Prints a summary** ([main.rs:93-111](../../integration-tests/src/main.rs#L93-L111)) and **blocks on `signal::ctrl_c()`**.

### Net result

- Docker: `mpc_it_network` bridge, `redis:7.4.2` container, `near/sandbox:latest` container, plus anvil/solana-test-validator if enabled.
- Host processes: 3× `mpc-node` binaries.
- NEAR contract deployed on sandbox with 3 participants registered and threshold 2.
- ~0 key-generation wait time.

Warm boot: 15–30 seconds. First-time cold boot (pulling images, building release profile): 5+ minutes.

### Changing behaviour

| To do this | Do this |
|---|---|
| Different node count / threshold | `--nodes N --threshold T`. Non-(3/2) or (5/4) tuples fall through to live keygen (~+20s boot). |
| Enable the Ethereum indexer end-to-end | The `--eth-*` flags are always attached, but anvil doesn't boot automatically from `setup-env`. Run anvil separately and set `--eth-execution-rpc-http-url` to it, or fall back to `cargo run --features docker-test -- setup-env ...` `[UNVERIFIED]` for anvil-in-container. |
| Enable Solana end-to-end | The `setup-env` subcommand does **not** expose `--sol-*` flags today `[UNVERIFIED]`. You'd add flags or use the `ClusterSpawner` builder API from Rust. |
| Run nodes as docker containers | `cargo run --features docker-test -- setup-env ...` |
| Skip the MPC nodes, keep only dependencies | `cargo run -- dep-services` — see [§6.4](#64-setup-type-dependencies-only-byo-node). |

---

## 6.4 Setup type: dependencies only (BYO node)

```bash
cd integration-tests
cargo run -- dep-services
```

Starts Redis + NEAR sandbox without MPC nodes ([main.rs:117-126](../../integration-tests/src/main.rs#L117-L126)). Then start a node manually — full logs, attachable debugger, fast iteration ([integration-tests/README.md:118-126](../../integration-tests/README.md#L118-L126)).

Starting a node against this setup is a [§6.7](#67-setup-type-node-by-hand-cli-flags) exercise.

---

## 6.5 Setup type: keep containers from a test run

Run an integration test, freeze the containers after it completes:

```bash
TESTCONTAINERS=keep cargo test -p integration-tests test_solana_signature_basic -- --nocapture
```

Inspect after:

```bash
docker ps | grep mpc
# copy the host port mapped to container port 3000 for a node
curl http://127.0.0.1:<mapped-port>/state | jq .
curl http://127.0.0.1:<mapped-port>/status | jq .
curl http://127.0.0.1:<mapped-port>/metrics | head -40
docker logs <container-id>
```

Cleanup:

```bash
docker ps -aq --filter "name=mpc" | xargs docker rm -f
```

Full playbook: [integration-tests/README.md:89-111](../../integration-tests/README.md#L89-L111).

---

## 6.6 Setup type: in-process MPC fixture

[integration-tests/src/mpc_fixture/builder.rs](../../integration-tests/src/mpc_fixture/builder.rs) constructs an MPC network **inside one Rust process** — no docker, no chains. Useful as an SDK unit-test harness where you want signing semantics but don't need to exercise the indexer or chain path.

The recent commit `c59ba07` moved the fixture logic into `build()` and made `threshold` required. Imports needed are at [builder.rs:4-38](../../integration-tests/src/mpc_fixture/builder.rs#L4-L38). Build pattern: `MpcFixtureBuilder::new(...).threshold(...).use_preshared_key(true).build()` `[UNVERIFIED]` — read [mpc_fixture/mod.rs](../../integration-tests/src/mpc_fixture/mod.rs) for the public API.

---

## 6.7 Setup type: node by hand (CLI flags)

For running a node against anything other than the test harness — real testnet, partner mainnet, custom config. Source of truth: [chain-signatures/node/src/cli.rs:33-98](../../chain-signatures/node/src/cli.rs#L33-L98).

### Required flags

| Flag | Env var | Purpose |
|---|---|---|
| `--near-rpc` | `MPC_NEAR_RPC` | NEAR RPC endpoint |
| `--mpc-contract-id` | `MPC_CONTRACT_ID` | MPC contract NEAR account |
| `--account-id` | `MPC_ACCOUNT_ID` | This node's NEAR account |
| `--account-sk` | `MPC_ACCOUNT_SK` | This node's ed25519 secret key |
| `--cipher-sk` | `MPC_CIPHER_SK` | Hex-encoded 32-byte HPKE cipher key for inter-node messages |
| Redis URL | via `storage::Options` | Local Redis URL |
| GCP project id | via `storage::Options` | Used as label (no real GCP unless you also set `--sk-share-secret-id`) |

### Optional flags

| Flag | Env var | Default | Notes |
|---|---|---|---|
| `--web-port` | `MPC_WEB_PORT` | 3000 | [cli.rs:52-56](../../chain-signatures/node/src/cli.rs#L52-L56) |
| `--sign-sk` | — | derived from `account_sk` | Inter-node message signing key |
| `--my-address` | `MPC_LOCAL_ADDRESS` | — | URL peers use to reach this node — [cli.rs:75-81](../../chain-signatures/node/src/cli.rs#L75-L81) |
| `--override-config` | `MPC_OVERRIDE_CONFIG` | — | Local overrides of contract config |
| `--eth-*` | `MPC_ETH_*` | (see [§5.2](05-indexers.md#52-ethereum)) | Ethereum indexer config |
| `--sol-*` | `MPC_SOL_*` | (see [§5.3](05-indexers.md#53-solana)) | Solana indexer config |
| `--hydration-*` | `MPC_HYDRATION_*` | (see [§5.4](05-indexers.md#54-hydration-substrate)) | Hydration indexer config |

### External dependencies to have running first

1. **Redis** — or the node loops on storage init.
2. **NEAR RPC** — always required (the NEAR contract is authoritative for network state/membership).
3. **Whatever chain RPC** for each indexer you've enabled.
4. **GCP Application Default Credentials** — only if using Secret Manager for the key share (prod).

---

## 6.8 What the node exposes — the HTTP surface

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

---

## 6.9 Getting a signature "directly" — the minimum viable path

The closest thing to "call the MPC directly" is: stand up the cluster, then call the NEAR sandbox's MPC contract from a NEAR CLI script. That contract has **no indexer in the loop** — the node reads pending requests from the contract's state directly ([indexer.rs:85-95](../../chain-signatures/node/src/indexer.rs#L85-L95)). Steps:

1. Bring up the cluster: `cargo run -- setup-env --nodes 3 --threshold 2` — note the NEAR sandbox RPC address in the output.
2. Use near-cli (or near-workspaces, or the raw RPC) to call `sign` on the MPC contract account. Example script body: see generated [chain-signatures/contract/EXAMPLE.md](../../chain-signatures/contract/EXAMPLE.md), regenerate via `cargo run -- contract-commands` ([integration-tests/src/main.rs:127-132](../../integration-tests/src/main.rs#L127-L132)).
3. The call will **hang** until the network responds — that's the promise yield. You receive the signature as the result of the call. No event listening required on NEAR.

Calling contracts on other chains requires also standing up that chain (anvil for Ethereum, solana-test-validator for Solana) and deploying the signing contract there. The cleanest reference is the test harness itself — `Cluster::sign().solana()` in [integration-tests/src/actions/sign.rs](../../integration-tests/src/actions/sign.rs) is exactly the shape an SDK would take.

---

## 6.10 Logging & tracing

Three modes ([integration-tests/README.md:22-48](../../integration-tests/README.md#L22-L48)):

- **fmt** — default, human-readable console.
- **OTLP** — point at a collector (e.g. Jaeger) via `MPC_OTLP_ENDPOINT` (default `http://localhost:4318`) and `MPC_OPENTELEMETRY_LEVEL` (`debug` / `info`). Useful to see the per-request trace flow.
- **Stackdriver** — GCP-only.

For a running test with verbose logs: `RUST_BACKTRACE=full RUST_LOG=debug cargo test test_name -- --nocapture`.

---

## 6.11 Troubleshooting

### "No such file or directory (os error 2)" when the harness tries to talk to Docker

The harness has a built-in fallback for macOS Docker Desktop's socket location. See [containers.rs:311-337](../../integration-tests/src/containers.rs#L311-L337):

```rust
let docker = match bollard::Docker::connect_with_defaults() {
    Ok(docker) => docker,
    Err(_) => {
        // fall back to Docker Desktop socket path
        let home_socket = format!("unix://{home}/.docker/run/docker.sock");
        bollard::Docker::connect_with_unix(&home_socket, timeout, api_version)
        ...
    }
};
```

`connect_with_defaults` tries `DOCKER_HOST` (if set), then `/var/run/docker.sock` on Unix. On failure, it retries at `~/.docker/run/docker.sock` — where Docker Desktop ≥4.13 actually puts its socket on macOS. So this error normally means:

- Docker Desktop isn't running.
- You have a stale `DOCKER_HOST` env var pointing somewhere wrong (`unset DOCKER_HOST`).
- You're on an old Docker Desktop predating the `~/.docker/run/` path.

**Fix options** (in order of preference):

1. Docker Desktop → Settings → Advanced → tick **"Allow the default Docker socket to be used"**. Creates `/var/run/docker.sock` via a helper. Cleanest.
2. `sudo ln -s $HOME/.docker/run/docker.sock /var/run/docker.sock` — the symlink from [integration-tests/README.md:128-133](../../integration-tests/README.md#L128-L133). Manual, and you have to clean it up when Docker Desktop upgrades or moves the socket.
3. Set `DOCKER_HOST=unix://$HOME/.docker/run/docker.sock` in your shell profile.

### "Missing `mpc_contract.wasm`" or similar

The cargo runner `./setup.sh` builds the contract WASM automatically on every `cargo run -p integration-tests` (see [§6.3](#63-setup-type-full-cluster)). You only see this error if:

- You set `MPC_SETUP_SKIP=1` but the WASM isn't built yet. Run `./build-contract.sh` once.
- You don't have the `1.81.0` rustup toolchain installed: `rustup install 1.81.0` and `rustup target add wasm32-unknown-unknown --toolchain 1.81.0`.

### Node stuck in `NotRunning` state indefinitely

Check `GET /state` on the node. If `triple_count` and `presignature_count` are both 0 and not climbing:

- If using pregenerated keys (nodes=3 threshold=2 or nodes=5 threshold=4): the fixture may not have loaded — check logs for `"stored key share for participant"`.
- If using live keygen: keygen takes ~20s the first time. Check logs for keygen progress.
- Verify Redis is reachable and not full. `docker logs` on the Redis container; `redis-cli` against it if you can reach the mapped port.

### "error trying to connect" to the NEAR sandbox RPC

- The sandbox picks a random host port. Get it from the `setup-env` output or from `docker ps`.
- Sandbox takes a few seconds to become ready after spawn — retry.

### Solana tests fail with "solana-test-validator: command not found"

Install the Solana CLI tools locally (they're expected on `PATH`, not in a container) — see [§6.1 prereqs](#61-prerequisites).

### `cargo +1.81.0` errors with "could not find `1.81.0` in `rustup`"

```bash
rustup install 1.81.0
rustup target add wasm32-unknown-unknown --toolchain 1.81.0
```

### Want to see the logs of a specific test in full colour

```bash
RUST_BACKTRACE=full RUST_LOG=debug cargo test -p integration-tests test_name -- --nocapture
```

Combine with `TESTCONTAINERS=keep` ([§6.5](#65-setup-type-keep-containers-from-a-test-run)) to inspect running state after the assertion fires.

---

[← §5 Indexers](05-indexers.md) · [Index](README.md) · Next: [§7 Extending →](07-extending-chains.md)
