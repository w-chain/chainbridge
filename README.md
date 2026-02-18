# ChainBridge-core Example — Internal Guide
<a href="https://discord.gg/ykXsJKfhgq"><img alt="discord" src="https://img.shields.io/discord/593655374469660673?label=Discord&logo=discord&style=flat" /></a>

- Audience: internal devs/maintainers
- Scope: thin example app + CLI that wires ChainSafe chainbridge-core into a runnable relayer
- Upstream: chainbridge-core framework https://github.com/ChainSafe/chainbridge-core

## TL;DR
- Build: `go build ./` → binary `chainbridge`
- Run: `./chainbridge run --config ./config/relayer_1.json --keystore ./keys --blockstore ./lvldbdata`
- Docker: `docker-compose up` (set KEYSTORE_PASSWORD; provide config and keys)

## Architecture
- Entry: [main.go](file:///Users/evanstinger/Dev/w-chain/chainbridge/main.go#L10-L12) → [cmd/cmd.go](file:///Users/evanstinger/Dev/w-chain/chainbridge/cmd/cmd.go#L35-L40) (Cobra)
- Runtime: [example/app.go](file:///Users/evanstinger/Dev/w-chain/chainbridge/example/app.go#L23-L72) orchestrates config load, blockstore, chain setup, relayer loop
- CLI: root commands (run) + EVM sub-CLI from core (accounts, deploy, etc.)
- Persistence: LevelDB via core’s `lvldb` + `store.BlockStore`
- Logging: zerolog

```mermaid
flowchart TD
  A[Start CLI] --> B[Load Flags (viper)]
  B --> C[Load Config (core.config)]
  C --> D[Open LevelDB (lvldb)]
  D --> E[New BlockStore]
  E --> F[Build Chains]
  F --> G[New Relayer]
  G --> H[Start Loop]
  H -->|errChn| I[Log error + stop]
  H -->|signals| J[Graceful exit]
```

## Repo Layout
- [main.go](file:///Users/evanstinger/Dev/w-chain/chainbridge/main.go) — program entry
- [cmd/](file:///Users/evanstinger/Dev/w-chain/chainbridge/cmd) — Cobra root, `run` command, mounts EVM CLI
- [example/app.go](file:///Users/evanstinger/Dev/w-chain/chainbridge/example/app.go) — relayer wiring
- [config/](file:///Users/evanstinger/Dev/w-chain/chainbridge/config) — example relayer configs
- [docker-compose.yml](file:///Users/evanstinger/Dev/w-chain/chainbridge/docker-compose.yml) — multi-relayer example
- [Makefile](file:///Users/evanstinger/Dev/w-chain/chainbridge/Makefile), [Dockerfile](file:///Users/evanstinger/Dev/w-chain/chainbridge/Dockerfile), [.goreleaser.yml](file:///Users/evanstinger/Dev/w-chain/chainbridge/.goreleaser.yml)

## Build & Run
- Local
  - `go build ./` or `make build`
  - `./chainbridge run --config ./config/relayer_1.json --keystore ./keys --blockstore ./lvldbdata`
- Docker
  - `docker-compose up` (uses image `chainsafe/chainbridge-core-example:main`)
  - Config mounts to `/config`; keys to `/keys`; env `KEYSTORE_PASSWORD` required

Run command surface (from Cobra):
```text
Run example app
Usage:
  run [flags]
Flags:
      --blockstore string   Specify path for blockstore (default "./lvldbdata")
      --config string       Path to JSON configuration file
      --fresh               Disables loading from blockstore at start (default: false)
  -h, --help                help for run
      --keystore string     Path to keystore directory (default "./keys")
      --latest              Overrides blockstore and start block (default: false)
      --testkey string      Applies a predetermined test keystore to the chains
```

### Global Flags
Available on all subcommands (exposed by core):
```text
--gasLimit uint               gasLimit used in transactions (default 6721975)
--gasPrice uint               gasPrice used for transactions (default 20000000000)
--jsonWallet string           Encrypted JSON wallet
--jsonWalletPassword string   Password for encrypted JSON wallet
--networkid uint              networkid
--privateKey string           Private key to use
--url string                  node url (default "ws://localhost:8545")
```

## Configuration
- File provided via `--config`; loaded with viper → core `config.GetConfig`
- Schema (simplified):
```json
{
  "relayer": {},
  "chains": [
    {
      "name": "chain-name",
      "type": "evm",
      "id": 1,
      "endpoint": "wss://…",
      "from": "0x…",
      "bridge": "0x…",
      "erc20Handler": "0x…",
      "gasLimit": 200000,
      "maxGasPrice": 20000000000,
      "blockConfirmations": 10
    }
  ]
}
```
- Notes
  - `type` must be `"evm"` in this example
  - Handler addresses in config drive which assets/handlers are active
  - Avoid committing secrets or API keys; keep endpoints public/ephemeral

### Chain Setup
```mermaid
flowchart TD
  Cfg[ChainConfig (type=evm)] --> S[SetupDefaultEVMChain]
  S --> H[Register handlers (erc20/erc721/generic)]
  S --> T[Tx factory (evmtransaction.NewTransaction)]
  S --> B[Attach BlockStore]
  S --> RC[RelayedChain]
```

Key wiring:
- Iterate `configuration.ChainConfigs`; for `type == "evm"` call
  [SetupDefaultEVMChain](file:///Users/evanstinger/Dev/w-chain/chainbridge/example/app.go#L39-L45)
- Append resulting `relayer.RelayedChain` to `chains`
- Construct relayer with chains + console telemetry; start in a goroutine

## Event Flow (High Level)
```mermaid
flowchart LR
  RPC[EVM RPC] --> F[Filter/Subscribe]
  F --> D[Decode events]
  D --> R[Route to handler]
  R --> TX[Build tx]
  TX --> S[Sign (keystore)]
  S --> SUB[Submit tx]
  SUB --> UPD[Update BlockStore]
```

## Keystore, Blockstore, Telemetry
- Keystore
  - Path via `--keystore`
  - Password via env (see compose `KEYSTORE_PASSWORD`)
  - `--testkey` applies a predetermined test key (for dev only)
- Blockstore
  - LevelDB path via `--blockstore`; created/opened in
    [example/app.go](file:///Users/evanstinger/Dev/w-chain/chainbridge/example/app.go#L27-L33)
  - Tracks last processed block per chain
- Telemetry
  - Console exporter used by example: `opentelemetry.ConsoleTelemetry`

## Error Handling & Signals
- Startup: DB/chain setup errors cause panic in example wiring
- Runtime: relayer errors propagate on `errChn`; logged then stop signal sent
- Signals: SIGTERM/SIGINT/SIGHUP/SIGQUIT trigger graceful exit

## CLI Surface
```mermaid
flowchart TB
  root[root] --> run[run]
  root --> evm[EVM CLI (from core)]
```
- Root + `run` defined here:
  - [cmd/cmd.go](file:///Users/evanstinger/Dev/w-chain/chainbridge/cmd/cmd.go#L15-L33)
  - EVM subtree from core mounted at root add: [cmd/cmd.go](file:///Users/evanstinger/Dev/w-chain/chainbridge/cmd/cmd.go#L35-L37)
- List EVM subcommands: `./chainbridge evm --help` (delegated to core)

## Dependencies
- Declared in [go.mod](file:///Users/evanstinger/Dev/w-chain/chainbridge/go.mod#L5-L11)
  - `github.com/ChainSafe/chainbridge-core v1.4.2`
  - `github.com/spf13/cobra`, `github.com/spf13/viper`
  - `github.com/rs/zerolog`
  - indirect: `github.com/ethereum/go-ethereum`

## Development
- Go ≥ 1.15; modules enabled
- Tests: none local; E2E harness referenced in CI (external)
- Lint/format: follow standard Go fmt; keep binaries small; avoid logging secrets
- Release: Docker via [.goreleaser.yml](file:///Users/evanstinger/Dev/w-chain/chainbridge/.goreleaser.yml)

## Security
- Report vulnerabilities to [security@chainsafe.io](mailto:security@chainsafe.io)

## License
GNU Lesser General Public License v3.0
