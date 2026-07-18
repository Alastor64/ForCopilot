# AGENTS.md

## Project

DHT (Distributed Hash Table) — PPCA 2026 academic assignment. Implements **Chord** and **Kademlia** protocols over Go net/rpc.

## Build & Test

```bash
go build -o dht .                         # build from project root
go test ./node/... -v -timeout 0          # run ALL tests (always use -timeout 0!)
go test ./node -run TestBasic -v -timeout 0
go test ./node -run TestForceQuit -v -timeout 0
go test ./node -run TestQuitAndStabilize -v -timeout 0
go test ./node -run TestDelete -v -timeout 0
```

Tests spawn many nodes on localhost; they need Linux/WSL2, **not WSL1 or Windows**.

## Architecture

Two implementations of the `DhtNode` interface ([node/interface.go](node/interface.go)):

| File | Protocol | Routing | Key features |
|---|---|---|---|
| [node/node.go](node/node.go) | Chord | Finger table + successor/predecessor | Stabilization loop, data backup to successor |
| [node/kdm.go](node/kdm.go) | Kademlia | k-buckets with XOR distance (k=10, α=3) | Iterative lookup, versioned KV with last-writer-wins |

Factory ([node/factory.go](node/factory.go)) currently returns `*Kdm` by default.

**Hash space:** 8-bit (`m=8`, 256 positions). Type alias `hint = uint8`. Hash function is a polynomial rolling hash (`base=37`). Because the space is so small, `Join` does linear probing (`id.Code++`) on collision.

## Key Conventions

- **Logging:** `logrus` (sirupsen/logrus) → `dht-test.log`. Both `node.go` and `kdm.go` call `logrus.SetOutput` in `init()`.
- **RPC:** `RemoteCall` wraps `client.Call` with connection pooling (`clients` map). After non-`GetCode` calls, it also sends an `UpdateBucket` to the remote peer.
- **Concurrency:** Per-struct `sync.RWMutex` (e.g., `dataLock`, `bucketLock`). The `period()` goroutine holds `periodLock` while running; `Quit()` acquires `periodLock` to wait for it to stop.
- **Data versioning:** Kademlia uses versioned key-value pairs. `Put`/`Delete` read current version from k-closest, then write version+1. Delete = tombstone with incremented version. Get returns highest-version value.

## Pitfalls

- **Chord `Join`** has a bare `for` loop on `FindSuc` failure — infinite hang if bootstrap is unreachable.
- **Kademlia `Quit`** does NOT transfer data to peers (unlike Chord). Relies on periodic `sendData()` for durability.
- **`fmt.Println`** calls scattered in code (`"Version fail!!!"` etc.) — use `logrus` instead.
- **Test port ranges** are hardcoded (20000–20100, 20200–20250, 20400–20450). Don't exceed them.
- **Global state:** `localAddress` in `addr.go` and `testutil.Wg` are package-level variables shared across tests.
- **Debugging:** Don't use a step-through debugger — DHT timing is sensitive. Use logrus + [klogg](https://klogg.filimonov.dev/) for log analysis.

## Docs

- [README.md](README.md) — build, run, CLI flags
- [doc/tutorial.md](doc/tutorial.md) — Go references, protocol papers, debugging tips
- [doc/Go.md](doc/Go.md) — Go syntax cheat sheet
