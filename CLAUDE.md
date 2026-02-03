# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

mc-router is a Minecraft Java Edition reverse proxy that routes client connections to backend servers based on the requested server address (hostname). It multiplexes multiple Minecraft servers onto a single public IP/port and supports auto-discovery via Kubernetes and Docker, auto-scaling (scale to zero/one), webhooks, rate limiting, IP filtering, PROXY protocol, and ngrok tunneling.

## Build & Test Commands

```bash
go build ./cmd/mc-router/       # Build the binary
make test                        # Run all tests (go test ./...)
go test ./server/...             # Run only server package tests
go test ./mcproto/...            # Run only protocol package tests
go test -run TestRouteLookup ./server/  # Run a single test
docker build -t mc-router .      # Build Docker image
```

Go version: 1.25. Testing uses `testify` (assert/require). Tests are table-driven.

## Architecture

### Request Flow

1. **Connector** (`server/connector.go`) accepts TCP connections on port 25565
2. **mcproto** package (`mcproto/`) reads the Minecraft handshake packet to extract the target server address
3. **Routes** (`server/routes.go`) looks up the backend address for that hostname
4. If auto-scale is enabled and the backend is sleeping, a **waker** function starts it (Kubernetes StatefulSet replica 0→1 or Docker container start/unpause)
5. Traffic is proxied bidirectionally between client and backend
6. On disconnect, metrics are updated, webhooks fired, and the **DownScaler** (`server/down_scaler.go`) may schedule scale-down after idle timeout

### Key Packages

- **`cmd/mc-router/`** — Entry point. Parses CLI flags via `go-flagsfiller`, sets up signal handling (SIGINT for shutdown, SIGHUP for config reload).
- **`server/`** — Core router logic:
  - `server.go` — Initializes all subsystems (metrics, routes, connector, API, service discovery)
  - `connector.go` — Connection handler: accepts clients, reads handshake, proxies traffic, manages rate limiting and client filtering
  - `routes.go` — In-memory route table mapping server addresses to backends; supports default route fallback
  - `routes_config_loader.go` — Loads/watches JSON routes config file (with fsnotify)
  - `k8s.go` — Kubernetes service discovery via annotation `mc-router.itzg.me/externalServerName`
  - `docker.go` / `docker_swarm.go` — Docker/Swarm container discovery via label `mc-router.host`
  - `down_scaler.go` — Auto-scale down after idle period
  - `api_server.go` — REST API (`GET/POST /routes`, `POST /defaultRoute`, `DELETE /routes/{serverAddress}`)
  - `metrics.go` — Pluggable metrics backends (Prometheus, InfluxDB, expvar, discard)
  - `webhook_notifier.go` — POST notifications on connect/disconnect events
  - `client_filter.go` / `allow_deny_list.go` — IP and player allow/deny lists
  - `configs.go` — All configuration structs with CLI flag annotations
- **`mcproto/`** — Minecraft Java protocol implementation:
  - `types.go` — Frame, Packet, Handshake, LoginStart types
  - `read.go` / `write.go` — Network I/O for Minecraft protocol frames
  - `decode.go` — Packet decoding (handshake, login, status)

### Configuration

CLI flags are the primary config mechanism, with environment variable support via `go-flagsfiller` (flag `--some-flag` maps to env `SOME_FLAG`). Routes can also be loaded from a JSON config file (`--routes-config`). The `Config` struct in `server/configs.go` defines all options.

### Service Discovery

Routes are populated from three sources that can be combined:
1. Static `--mapping` flags or JSON config file
2. Kubernetes: watches Services with `mc-router.itzg.me/externalServerName` annotation
3. Docker/Swarm: watches containers with `mc-router.host` label

### Global Singletons

Several package-level singletons must be initialized before use:
- `server.Routes` — Initialized at package load via `NewRoutes()`
- `server.DownScaler` — Initialized in `NewServer()`; **must exist before route operations that involve auto-scaling**
- `server.RoutesConfigLoader` — Zero-value struct, configured via `Load()`

In tests, you must initialize `DownScaler` manually if testing route/scaling behavior:
```go
DownScaler = NewDownScaler(context.Background(), false, 1*time.Second)
```

### Server Address Normalization

`FindBackendForServerAddress()` in `routes.go` normalizes incoming hostnames before route lookup. The normalization chain strips, in order: Forge mod suffixes (`\x00`), infinity-filter suffixes (`\\`), DNS trailing dots, TCPShield patterns, and optionally simplifies SRV records. This is critical to understand when debugging route matching issues.

### Code Conventions

- Logging: `sirupsen/logrus` with structured fields (`logrus.WithField(...).Info(...)`)
- Error wrapping: older code uses `pkg/errors` (`errors.Wrap`), newer code uses `fmt.Errorf` with `%w`
- Interfaces: some prefixed with `I` (`IRoutes`, `IDownScaler`) and some not (`RouteFinder`, `ConnectionNotifier`)
- Tests are in the same package (not `_test` suffix), enabling access to unexported types
- Test fixtures for protocol packets are `.hex` files in `mcproto/`
- Testify `mock.Mock` is used for interface mocking (see `MockedRoutesHandler` in `k8s_test.go`)

### Protocol Notes

The `mcproto` package handles Minecraft Java protocol quirks: Forge mod identifiers appended to server addresses (separated by `\x00`), DNS root zone trailing dots, legacy server list ping format, and VarInt encoding. Server address matching in routes strips these suffixes before lookup. Login packet decoding varies by protocol version (1.18.2 has no UUID, 1.19–1.19.2 has optional UUID with signature data, 1.20.2+ always includes UUID).
