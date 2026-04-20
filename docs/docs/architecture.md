# Architecture

Hyperledger Cacti is a pluggable interoperability framework for linking networks built on heterogeneous distributed ledgers and running transactions that span multiple networks. This page explains how the framework is organized after the 2022 merger of **Hyperledger Cactus** and the **Weaver Lab** project, how its modules fit together, and where new contributors should look first.

The integrated architecture is illustrated below.

<img src="../images/cacti-architecture-v2-integration.png">

The architecture is a curated set of modules, services (offering standard APIs), and libraries that can be composed into cross-network transaction pipelines. The integrated view was produced by fusing the pre-existing Cactus and Weaver architectures — shared components were re-labelled and unique components were called out separately.

For reference, here is the original Cactus architecture:

<img src="../images/cactus-architecture.png">

And here is the original Weaver architecture:

<img src="../images-weaver/weaver-architecture.png">

## Why two architectures inside one repo

Cactus and Weaver were created independently, with different — and complementary — trust assumptions:

- **Cactus** orchestrates cross-network transactions through a **Node Server** that applications call directly. It is connector-centric: each supported ledger gets a connector module that exposes a uniform REST/gRPC API, and the Node Server composes those APIs.
- **Weaver** orchestrates the same kinds of transactions through **Relays** deployed alongside each network, with ledger-side **drivers** and on-chain **interop modules** enforcing policy. It is protocol-centric: the relay network and its RFCs define how two chains talk to each other without a trusted intermediary.

Post-merger, Cacti keeps both mechanisms available so that integrators can choose — or combine — whichever trust model suits their use case.

The current repository contains the legacy Cactus and Weaver source trees in aggregated form with their original folder structures intact. Packages built from both sides are published under the Hyperledger org on npm — legacy ones under `@hyperledger/cactus-*` and newer ones under `@hyperledger-cacti/cacti-*` — and CI/CD is unified under a common set of GitHub Actions. A deeper merge of source code is on the roadmap (see [ROADMAP.md](https://github.com/hyperledger-cacti/cacti/blob/main/ROADMAP.md)); this page describes the current state.

## Repository layout

```
cacti/
├── packages/              # Cactus-legacy source: connectors, plugins, core, SATP-Hermes
├── weaver/                # Weaver-legacy source: drivers, relays, SDKs, protos, samples
├── examples/              # End-to-end example applications
├── tools/                 # Shared tooling (docker helpers, codegen, etc.)
├── docs/                  # This documentation site (MkDocs)
├── extensions/            # Experimental and community extensions
├── images/                # Diagrams and logos referenced from README/docs
├── whitepaper/            # Historical Cactus whitepaper sources
└── .github/               # Workflows, issue templates, PR template
```

The two most important directories are `packages/` (Cactus-legacy) and `weaver/` (Weaver-legacy). The sections below explain each.

## The `packages/` tree — Cactus-legacy subsystem

Every subdirectory in `packages/` is an independent npm workspace. They fall into the categories below.

### Core framework

These are the load-bearing modules every deployment consumes:

| Package                      | Role                                                                          |
|------------------------------|-------------------------------------------------------------------------------|
| `cactus-common`              | Shared types, logger, error helpers used across all modules.                  |
| `cactus-core`                | In-process plugin registry and consortium primitives.                         |
| `cactus-core-api`            | The plugin interfaces every ledger connector and feature plugin implements.   |
| `cactus-cmd-api-server`      | The Cactus Node Server binary — loads plugins from config and serves REST/gRPC/SocketIO APIs. |

### Ledger connectors

Each connector wraps a specific ledger's native SDK behind the unified `cactus-core-api` plugin interface. Adding support for a new DLT means adding a new connector here.

| Package                                  | Ledger                      |
|------------------------------------------|-----------------------------|
| `cactus-plugin-ledger-connector-besu`    | Hyperledger Besu            |
| `cactus-plugin-ledger-connector-ethereum`| Go-Ethereum-compatible      |
| `cactus-plugin-ledger-connector-fabric`  | Hyperledger Fabric          |
| `cactus-plugin-ledger-connector-corda`   | R3 Corda                    |
| `cactus-plugin-ledger-connector-sawtooth`| Hyperledger Sawtooth        |
| `cactus-plugin-ledger-connector-iroha2`  | Hyperledger Iroha 2         |
| `cactus-plugin-ledger-connector-polkadot`| Polkadot / Substrate        |
| `cactus-plugin-ledger-connector-xdai`    | Gnosis Chain (xDai)         |
| `cactus-plugin-ledger-connector-cdl`     | Fujitsu ConnectionChain     |
| `cactus-plugin-ledger-connector-aries`   | Hyperledger Aries (SSI)     |
| `cacti-plugin-ledger-connector-stellar`  | Stellar                     |

### Keychain plugins

Keychains hold the credentials connectors need (private keys, API tokens, wallet files). The backing store is pluggable.

| Package                                  | Backing store                          |
|------------------------------------------|----------------------------------------|
| `cactus-plugin-keychain-memory`          | In-memory (dev/test only).             |
| `cactus-plugin-keychain-memory-wasm`     | In-memory, WASM-compatible.            |
| `cactus-plugin-keychain-vault`           | HashiCorp Vault.                       |
| `cactus-plugin-keychain-aws-sm`          | AWS Secrets Manager.                   |
| `cactus-plugin-keychain-azure-kv`        | Azure Key Vault.                       |
| `cactus-plugin-keychain-google-sm`       | Google Secret Manager.                 |

### Feature plugins

These compose connectors to deliver specific cross-network capabilities:

| Package                                  | Capability                             |
|------------------------------------------|----------------------------------------|
| `cactus-plugin-htlc-eth-besu`            | Hash-Time-Locked Contracts on Besu.    |
| `cactus-plugin-htlc-eth-besu-erc20`      | HTLC for ERC-20 tokens on Besu.        |
| `cactus-plugin-persistence-ethereum`     | Index and persist Ethereum ledger data to PostgreSQL. |
| `cactus-plugin-persistence-fabric`       | Same, for Fabric.                      |
| `cactus-plugin-consortium-manual`        | Static consortium membership list.     |
| `cacti-plugin-consortium-static`         | Newer static consortium implementation.|

### SATP-Hermes subsystem

The **Secure Asset Transfer Protocol (SATP)** is a standards-track (IETF) cross-chain asset transfer protocol. Its reference implementation lives under `packages/` but has its own release cadence and build pipeline:

| Package                          | Role                                                                  |
|----------------------------------|-----------------------------------------------------------------------|
| `cactus-plugin-satp-hermes`      | SATP gateway implementation and Solidity contracts (requires Foundry).|
| `cactus-plugin-bungee-hermes`    | Companion plugin for SATP proof/view generation.                      |

SATP-Hermes is intentionally self-contained and documented separately in its [package docs folder](https://github.com/hyperledger-cacti/cacti/tree/main/packages/cactus-plugin-satp-hermes/docs).

### COPM — Cross-network Operations for Private Markets

A newer family of packages (`cacti-*` namespace) implementing cross-network operations for permissioned DLTs:

| Package                          | Role                                         |
|----------------------------------|----------------------------------------------|
| `cacti-copm-core`                | Shared COPM types and orchestration logic.   |
| `cacti-copm-test`                | Shared test harness for COPM implementations.|
| `cacti-plugin-copm-fabric`       | COPM for Hyperledger Fabric.                 |
| `cacti-plugin-copm-corda`        | COPM for R3 Corda.                           |

### Clients and tooling

| Package                          | Role                                         |
|----------------------------------|----------------------------------------------|
| `cactus-api-client`              | Typed REST/gRPC/SocketIO client the JS SDK re-exports. |
| `cactus-verifier-client`         | Client for network verification workflows.   |
| `cacti-ledger-browser`           | Web UI for exploring indexed ledger data.    |
| `cacti-plugin-weaver-driver-fabric` | Bridge that exposes a Weaver driver as a Cactus plugin — one of the first points of structural integration between the two halves of the repo. |

### Test packages

Any package whose name starts with `cactus-test-*` contains integration tests and harnesses for its non-test counterpart. These are separated from the shipped packages so the shipped artifacts stay lean.

## The `weaver/` tree — Weaver-legacy subsystem

Weaver is multi-language by design — chaincode and drivers are written in the language native to each ledger. The top-level layout:

```
weaver/
├── common/       # Protocol buffers (Go, Java/Kotlin, JS, Rust, Solidity) and policy DSL
├── core/         # Relay service and per-ledger drivers
│   ├── drivers/  # Corda and Fabric drivers (JVM + TypeScript)
│   ├── network/  # Test network helpers
│   └── relay/    # The Weaver relay — the heart of Weaver's cross-chain protocol
├── sdks/         # Client SDKs for Besu, Corda, and Fabric
├── samples/      # End-to-end sample applications
├── tests/        # Cross-driver integration tests
├── rfcs/         # Weaver protocol RFCs (authoritative design docs)
└── resources/    # Shared artifacts (certificates, chaincode bundles, etc.)
```

The three load-bearing concepts to understand here are:

- **Relay** (`weaver/core/relay`) — the long-running service deployed alongside a network that brokers cross-network requests. Two relays talking to each other is how two networks exchange data or assets in Weaver.
- **Driver** (`weaver/core/drivers/*-driver`) — a ledger-specific adapter the relay uses to submit queries and transactions into its own network. Fabric and Corda drivers live here; the Fabric driver is additionally exposed to the Cactus side through `packages/cacti-plugin-weaver-driver-fabric`.
- **Interop module** — on-chain code (Solidity / chaincode / CorDapp) deployed into each participating network to enforce access-control policies for incoming cross-network requests. These live under `weaver/common` (for Solidity) and under the relevant ledger sample/driver folders.

Authoritative design documents for each Weaver protocol are in `weaver/rfcs/`.

## Supporting subsystems

- **`cmd-api-server`** is the runtime that hosts Cactus-side connectors and plugins. It reads a JSON config that lists plugin package names plus per-plugin options, loads each one, and exposes unified REST, gRPC, and SocketIO endpoints. Starting the server with `npm run start:api-server` is the fastest way to see the composed framework running.
- **`ROADMAP.md`** at the repo root tracks the incremental deeper merge (unified build, consolidated SDK surface, shared namespace migration).
- **GitHub Actions** in `.github/workflows/` run unit tests, integration tests, container builds, SATP-Hermes-specific pipelines, security scans, and documentation publishing. The `code-quality-checks` and `ci` workflows are the ones every PR triggers.

## Where to start contributing

| If you want to…                                             | Start here                                                     |
|-------------------------------------------------------------|----------------------------------------------------------------|
| Add support for a new ledger                                | Copy an existing `packages/cactus-plugin-ledger-connector-*` and implement the `cactus-core-api` plugin interface. |
| Add a cross-network capability (e.g., a new HTLC variant)   | Create a feature plugin under `packages/` that composes existing connectors. |
| Work on SATP (IETF asset-transfer protocol)                 | `packages/cactus-plugin-satp-hermes` (install [Foundry](https://book.getfoundry.sh/) for Solidity work — see [BUILD.md](https://github.com/hyperledger-cacti/cacti/blob/main/BUILD.md)). |
| Work on Weaver relay protocol or drivers                    | `weaver/core/relay` and `weaver/core/drivers`; read the relevant RFC under `weaver/rfcs/` first. |
| Improve docs                                                | This site's source lives under `docs/docs/`; the config is `docs/mkdocs.yml`. |
| Reduce CI runtime or flakiness                              | `.github/workflows/` — start with the workflows that run on every PR (`ci.yaml`, `code-quality-checks.yaml`). |

For the Cleanup Initiative's current priorities (reducing complexity, removing deprecated modules, improving security posture, optimizing CI/CD), see the [public project board](https://github.com/orgs/hyperledger-cacti/projects/2).

## Further reading

- [ROADMAP.md](https://github.com/hyperledger-cacti/cacti/blob/main/ROADMAP.md) — planned deeper merge and integration milestones.
- [BUILD.md](https://github.com/hyperledger-cacti/cacti/blob/main/BUILD.md) — building from source and configuring a local dev environment.
- [CONTRIBUTING.md](https://github.com/hyperledger-cacti/cacti/blob/main/CONTRIBUTING.md) — contribution workflow and commit conventions.
- [Weaver architecture and design](weaver/architecture-and-design/overview.md) — deeper dives into the relay, drivers, and decentralized-identity subsystems.
- [SATP-Hermes architecture](https://github.com/hyperledger-cacti/cacti/blob/main/packages/cactus-plugin-satp-hermes/docs/architecture/satp-hermes.md) — the SATP gateway reference implementation.
