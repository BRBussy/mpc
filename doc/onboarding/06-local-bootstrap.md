# §6 — Local bootstrap + direct MPC calls (deep)

[← §5 Indexers](05-indexers.md) · [Index](README.md) · Next: [§7 Extending →](07-extending-chains.md)

---

This is the hands-on spine. Everything else in the guide is background for this section.

## 6.1 Prereqs

**Docker** running (Docker Desktop on macOS/Windows, Docker Engine on Linux). Pull the Redis image ahead of time:

```bash
docker pull redis:7.4.2
```

That's it for most setups. The integration-test harness talks to Docker via [bollard](https://docs.rs/bollard) and already handles the two common socket locations itself — see [containers.rs:311-337](../../integration-tests/src/containers.rs#L311-L337):

```rust
let docker = match bollard::Docker::connect_with_defaults() {
    Ok(docker) => docker,
    Err(default_err) => {
        // fall back to Docker Desktop socket path
        let home_socket = format!("unix://{home}/.docker/run/docker.sock");
        bollard::Docker::connect_with_unix(&home_socket, timeout, api_version)
        ...
    }
};
```

`connect_with_defaults` tries `DOCKER_HOST` (if set), then `/var/run/docker.sock` on Unix. If that fails, it retries at `~/.docker/run/docker.sock` — which is where Docker Desktop ≥4.13 actually puts its socket on macOS. So **on macOS with modern Docker Desktop, no socket setup is needed**; on Linux, the socket is already at `/var/run/docker.sock` natively. The `sudo ln -s ... /var/run/docker.sock` workaround mentioned in [integration-tests/README.md:128-133](../../integration-tests/README.md#L128-L133) only applies if you run into the "No such file or directory (os error 2)" error, which typically means:

- Docker Desktop isn't running, or
- You have an unusual `DOCKER_HOST` env var set that points somewhere else, or
- You're on an older Docker Desktop that didn't even provide `~/.docker/run/docker.sock`, or
- You're using a third-party tool (not bollard) that hardcodes `/var/run/docker.sock` and you can enable the "Allow the default Docker socket to be used" checkbox in Docker Desktop → Settings → Advanced as a cleaner alternative to the symlink.

Try running without the symlink first. If it works, skip that step.

**Chain-specific extras:**

- **Solana**: requires `solana-test-validator` installed **locally** (not a docker container) — [containers.rs:595-700](../../integration-tests/src/containers.rs) spawns it as a native subprocess. Install via the [Solana CLI tools](https://solana.com/docs/intro/installation).
- **Ethereum** (when `--eth-*` flags are used or `use_ethereum` is set): requires Foundry's `anvil`. The harness uses either the `ghcr.io/foundry-rs/foundry:nightly` docker image or a local install `[UNVERIFIED]` which it actually uses in what mode.
- **NEAR**: comes "free" — the harness pulls `near/sandbox:latest` via the `near-workspaces` crate. No manual install.

**Rust toolchain**: as pinned by `rust-toolchain.toml` / the workspace, **plus** a Rust `1.81.0` channel installed via rustup — [build-contract.sh](../../build-contract.sh) uses `cargo +1.81.0` to build the NEAR contract WASM. Install with `rustup install 1.81.0` if you don't have it.

### `./setup.sh` — the cargo runner that prebuilds everything

You don't run this directly. It's wired in as a cargo runner via [.cargo/config.toml](../../.cargo/config.toml):

```toml
[target.'cfg(not(target = "wasm32-unknown-unknown"))']
runner = "./setup.sh"
```

Every `cargo run -p integration-tests ...` or `cargo test -p integration-tests ...` invocation becomes `./setup.sh <binary-path> <args...>`. The runner ([setup.sh:11-44](../../setup.sh)) then:

1. Builds the NEAR contract WASM via `./build-contract.sh` (cargo 1.81.0 → `target/wasm32-unknown-unknown/release/mpc_contract.wasm`).
2. Builds the node binary via `cargo build -p mpc-node --release --features test-feature,debug-page` (→ `target/release/mpc-node`).
3. Execs the integration-tests binary.

**Env-var knobs:**
- `MPC_SETUP_SKIP=1` — skip the prebuild entirely. Useful when you know the binaries are already fresh and want faster iteration.
- `MPC_SETUP_ALWAYS=1` — run the prebuild even for non-integration-tests cargo invocations.

The prebuild only runs when `CARGO_PKG_NAME=integration-tests`, so other cargo invocations skip it by default.

## 6.2 Option A — full cluster with one command

```bash
cd integration-tests
cargo run -- setup-env --nodes 3 --threshold 2
```

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

All flag defaults are hardcoded in the clap annotations. All cluster settings that aren't exposed as flags (network name, GCP project id, env label, binary path) are hardcoded in [cluster/spawner.rs:18-20](../../integration-tests/src/cluster/spawner.rs#L18-L20):

```rust
const DOCKER_NETWORK: &str = "mpc_it_network";
const GCP_PROJECT_ID: &str = "multichain-integration";
const ENV: &str = "integration-tests";
```

To change these, edit the source and recompile. There's no YAML / TOML / `.env` for the harness.

### What actually runs when you execute the command

Tracing end-to-end from `cargo run` to "cluster up":

**1. Cargo compiles and runs the `integration-tests` binary** against the local workspace. The `-- setup-env --nodes 3 --threshold 2` after the `--` is passed to the binary as argv.

**2. [main.rs:53-116](../../integration-tests/src/main.rs#L53-L116) parses args, builds a `NodeConfig`** with `nodes=3, threshold=2`, attaches an `EthConfig` (from the `--eth-*` flags, which have defaults so they're always present), and constructs a `ClusterSpawner::default()`.

**3. `ClusterSpawner::default()` at [spawner.rs:134-168](../../integration-tests/src/cluster/spawner.rs#L134-L168)** sets up:
- A `DockerClient` (the bollard connection described in §6.1).
- `release: true` — use the release-mode binary at `target/release/mpc-node`.
- `wait_for_running: true`.
- Docker network name: `mpc_it_network`.
- GCP project id: `multichain-integration` (used only as a string label; no real GCP calls).
- Pre-stockpile: generate 4× the normal triple count (so tests don't stall on stockpile warm-up).
- **Pregenerated keys loaded from [mpc_fixture/3_nodes_2_threshold.json](../../integration-tests/src/mpc_fixture/3_nodes_2_threshold.json)** if the (nodes, threshold) tuple matches a known fixture. For `(3, 2)` and `(5, 4)` this skips ~20s of live key-generation protocol and starts nodes directly in the `Running` state. For any other tuple you fall through to full keygen.

**4. `.init_network()` creates the docker bridge network** `mpc_it_network` (via bollard).

**5. `.run()` branches on cargo feature `docker-test`:**

| Feature flag | Mode | Nodes run as | Called function |
|---|---|---|---|
| *(default)* | `host` | **native processes on your machine** | [lib.rs:573-664](../../integration-tests/src/lib.rs#L573-L664) |
| `--features docker-test` | `docker` | docker containers | [lib.rs:450-522](../../integration-tests/src/lib.rs#L450-L522) |

`setup-env` uses the default (`host`). That means the nodes are plain binaries running on your host, **not** docker containers, even though their dependencies (Redis, NEAR sandbox) are containerised. This is faster to iterate on.

**6. `setup(spawner)` at [lib.rs:329-448](../../integration-tests/src/lib.rs#L329-L448) builds the infra stack** in order:

1. **NEAR sandbox** — `near_workspaces::sandbox().await` spawns the `near/sandbox:latest` docker image. Exposes a JSON-RPC endpoint on a random host port.
2. **Creates N NEAR accounts on sandbox** (one per MPC node) — funded from the dev account.
3. **Deploys the compiled MPC contract WASM** from `target/wasm32-unknown-unknown/release/mpc_contract.wasm` via `worker.dev_deploy()`. That's the contract at [chain-signatures/contract/](../../chain-signatures/contract/). **The WASM is built automatically** by the cargo runner `./setup.sh` (see §6.1) before your binary starts — no manual prebuild step needed. If you're running with `MPC_SETUP_SKIP=1` you must have built it yourself via `./build-contract.sh` or this step fails with a missing-file error.
4. **Redis container** — `redis:7.4.2`. Exposes port 6379 on a random host port.
5. **If `use_ethereum`** — spawns `EthereumSandbox` (anvil) and deploys `ChainSignatures.sol` via ethers-rs.
6. **If `cfg.sol` is set** — spawns `solana-test-validator` as a native subprocess, deploys the Solana program from `chain-signatures/contract-sol/artifacts/chain_signatures.so`.
7. **Creates `target/tmp/secrets/`** — this is where each node's secret key share will be written (plain file, not GCP Secret Manager).
8. **If pregenerated keys are enabled** ([lib.rs:407-433](../../integration-tests/src/lib.rs#L407-L433)) — writes each participant's key share into their `secret_storage` before the node boots, so they come up in `Running` state immediately.

**7. `host(spawner)` at [lib.rs:573-664](../../integration-tests/src/lib.rs#L573-L664) spawns the MPC node processes**, one per account, in parallel. Each node is invoked as the `mpc-node start ...` binary (the same CLI from [cli.rs](../../chain-signatures/node/src/cli.rs)) with env vars set to point at the sandbox RPC, the Redis URL, the local secret path, the web port, etc.

**8. Back in `host()`: register all nodes as participants** on the MPC contract via a `init_running` call (skips keygen, uses the pregenerated public key) or `init` (triggers keygen).

**9. Back in `main()`: print a summary** of URLs, account IDs, secret keys, public keys for each node, and the sandbox/Redis addresses ([main.rs:93-111](../../integration-tests/src/main.rs#L93-L111)).

**10. Block on `signal::ctrl_c()`** ([main.rs:113](../../integration-tests/src/main.rs#L113)). The cluster stays up until you Ctrl-C.

### Net result

You get, all running on your host:

- Docker: `mpc_it_network` bridge, `redis:7.4.2` container, `near/sandbox:latest` container, plus anvil/solana-test-validator if enabled.
- Host processes: 3× `mpc-node` binaries.
- NEAR contract deployed on the sandbox with 3 participants registered and threshold 2.
- ~0 key-generation wait time (pregenerated fixture).

Approximate boot time on a warm Docker cache: 15–30 seconds. First-time boot on a cold machine (pulling images, building the binary release profile) can be 5+ minutes.

### Changing behaviour

- **Different node count / threshold** → pass `--nodes N --threshold T`. If N/T doesn't match a pregenerated fixture (today: 3/2 or 5/4), you fall through to live key generation which adds ~20s to boot.
- **Enable Ethereum indexer** → the `--eth-*` flags are already populated via defaults and always attached to the `EthConfig` in [main.rs:69-79](../../integration-tests/src/main.rs#L69-L79), so the Eth indexer is always *configured* — but whether it actually reaches an Ethereum node depends on whether you've got anvil at `http://localhost:8545`. Run anvil separately or set `--eth-execution-rpc-http-url` to a real endpoint.
- **Enable Solana indexer** → the `setup-env` subcommand at [main.rs:20-42](../../integration-tests/src/main.rs#L20-L42) currently does **not** expose `--sol-*` flags, so Solana is off by default from this entry point `[UNVERIFIED]`. To enable it you'd either add flags here or use the `ClusterSpawner` builder API from a Rust test. Confirm before relying on it.
- **Run nodes as docker containers** → `cargo run --features docker-test -- setup-env ...`.
- **Skip the MPC nodes, keep only dependencies** → use `cargo run -- dep-services` instead, see §6.3.

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
