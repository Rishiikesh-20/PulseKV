# PulseKV

PulseKV is a distributed key-value store written in Go. It uses a from-scratch Raft implementation to elect a leader, replicate mutations across nodes, persist Raft state, compact logs with snapshots, and expose a simple HTTP API plus a live browser dashboard.

The project is intentionally small enough to study, but it covers the important pieces of an etcd-like system: consensus, leader-only writes, linearizable reads, snapshot install, crash recovery, and a watch stream for live updates.

## Features

- Raft consensus with follower, candidate, and leader states.
- Leader election with randomized election timeouts.
- Log replication through Go `net/rpc`.
- Persistent Raft state stored in local `raft_state_<node_id>.json` files.
- KV snapshots and Raft log compaction after a configurable number of applied entries.
- `InstallSnapshot` support for followers that fall behind compacted logs.
- In-memory key-value state machine with client sequence deduplication.
- HTTP REST API for status, reads, writes, deletes, and watch events.
- Server-Sent Events (SSE) watcher for real-time PUT/DELETE updates.
- Static web dashboard served by each node.
- Small CLI client in `cmd/pulsectl`.

## Project Layout

```text
.
|-- api/
|   `-- server.go              # HTTP API, static UI server, SSE watcher
|-- cmd/
|   `-- pulsectl/main.go       # CLI client for put/get/delete/watch
|-- kvstore/
|   |-- kvstore.go             # In-memory KV state machine and snapshots
|   `-- kvstore_test.go
|-- raft/
|   |-- raft.go                # Raft node, election, replication, reads, snapshots
|   |-- rpc.go                 # Raft RPC request/response types and handlers
|   |-- transport.go           # TCP net/rpc transport and client cache
|   |-- persister.go           # File and memory persisters
|   `-- *_test.go
|-- ui/
|   |-- index.html             # Browser dashboard
|   |-- styles.css
|   `-- app.js
|-- main.go                    # Node entrypoint and wiring
|-- go.mod
`-- README.md
```

## Requirements

- Go 1.22 or newer for method-aware `http.ServeMux` routes such as `GET /v1/status`.
- Three free HTTP ports and three free RPC ports when running a local 3-node cluster.

The current `go.mod` declares `go 1.25.7`.

## Quick Start

Start three nodes in separate terminals from the repository root.

Terminal 1:

```bash
go run main.go -id 1 -http-port 8080 -rpc-port 9090 -peers "2=localhost:9091,3=localhost:9092"
```

Terminal 2:

```bash
go run main.go -id 2 -http-port 8081 -rpc-port 9091 -peers "1=localhost:9090,3=localhost:9092"
```

Terminal 3:

```bash
go run main.go -id 3 -http-port 8082 -rpc-port 9092 -peers "1=localhost:9090,2=localhost:9091"
```

After a few seconds one node becomes leader. Open any node in a browser:

- http://localhost:8080
- http://localhost:8081
- http://localhost:8082

The dashboard shows node ID, role, term, known leader, the live key-value table, and a real-time event stream.

## Running a Single Node

For a simpler local demo, run one node without peers:

```bash
go run main.go -id 1 -http-port 8080 -rpc-port 9090
```

A single-node cluster elects itself leader and commits writes immediately.

## CLI Flags

`main.go` supports:

| Flag | Default | Description |
| --- | --- | --- |
| `-id` | `1` | Unique numeric node ID. |
| `-http-port` | `8080` | Client API and UI port. |
| `-rpc-port` | `9090` | Internal Raft RPC port. |
| `-peers` | empty | Comma-separated peer list: `id=host:rpcPort,...`. |
| `-snapshot-interval` | `50` | Snapshot every N applied log entries. Use `0` to disable automatic snapshots. |

Example with faster snapshotting:

```bash
go run main.go -id 1 -http-port 8080 -rpc-port 9090 -snapshot-interval 10
```

## HTTP API

All API routes are under `/v1`.

| Method | Path | Description |
| --- | --- | --- |
| `GET` | `/v1/status` | Return this node's Raft status. |
| `GET` | `/v1/keys` | Return all keys from the leader using `ReadIndex`. |
| `GET` | `/v1/{key}` | Return one key from the leader using `ReadIndex`. |
| `PUT` | `/v1/{key}` | Propose a PUT command to the Raft log. Request body is the value. |
| `DELETE` | `/v1/{key}` | Propose a DELETE command to the Raft log. |
| `GET` | `/v1/watch` | Stream committed mutation events over SSE. |

Writes and linearizable reads must be sent to the current leader. If a follower receives one of these requests, it returns HTTP `307` with a JSON body containing the known `leader_id`.

### API Examples

Check node status:

```bash
curl http://localhost:8080/v1/status
```

Put a key on the leader:

```bash
curl -X PUT http://localhost:8080/v1/config/timeout \
  -H "Content-Type: text/plain" \
  --data "5000"
```

Get a key:

```bash
curl http://localhost:8080/v1/config/timeout
```

List all keys:

```bash
curl http://localhost:8080/v1/keys
```

Delete a key:

```bash
curl -X DELETE http://localhost:8080/v1/config/timeout
```

Watch live events:

```bash
curl -N http://localhost:8080/v1/watch
```

Use idempotency headers when retrying client mutations:

```bash
curl -X PUT http://localhost:8080/v1/session/token \
  -H "Content-Type: text/plain" \
  -H "X-Client-ID: demo-client" \
  -H "X-Client-Seq: 1" \
  --data "abc123"
```

The state machine records the highest sequence number seen per client and ignores duplicate or older commands from that client.

## CLI Client

Build the node binary and CLI:

```bash
go build -o pulsekv .
go build -o pulsectl ./cmd/pulsectl
```

Use `pulsectl` against the leader address:

```bash
./pulsectl --addr=http://localhost:8080 put greeting hello
./pulsectl --addr=http://localhost:8080 get greeting
./pulsectl --addr=http://localhost:8080 delete greeting
./pulsectl --addr=http://localhost:8080 watch
```

If `--addr` is omitted, the CLI uses `http://localhost:8080`.

## How Writes Flow Through the System

1. A client sends `PUT /v1/{key}` or `DELETE /v1/{key}` to a node.
2. The API checks whether the node is the Raft leader.
3. The leader appends the command to its local Raft log.
4. The leader sends `AppendEntries` RPCs to followers.
5. Once a majority replicate the entry, the leader advances `commitIndex`.
6. The node applies committed entries to the `KVStore`.
7. The watcher broadcasts a PUT/DELETE event to dashboard and SSE clients.
8. Periodically, the KV store serializes a snapshot and Raft compacts old log entries.

The HTTP write response confirms that the leader accepted the proposal. The live table and watch stream update after the entry is applied.

## Consistency Model

- Mutations are leader-only and replicated through Raft.
- Reads use the leader's `ReadIndex` path. The implementation verifies leadership with an immediate heartbeat round and waits until the state machine has applied the target commit index.
- Followers reject client reads and writes with a JSON "not the leader" response.
- Client retries can be made idempotent with `X-Client-ID` and `X-Client-Seq`.

This is an educational implementation, not a production replacement for etcd. The `ReadIndex` flow is simplified compared with production Raft systems, and the command format is space-delimited, so keys and values should be simple strings without whitespace.

## Persistence and Snapshots

Each node writes local state to:

```text
raft_state_<node_id>.json
```

The file contains encoded Raft state and snapshot bytes. On restart, the node restores Raft state from disk, then reloads the KV store from the latest snapshot before applying newer committed log entries.

Snapshots are controlled with:

```bash
-snapshot-interval <N>
```

Set `N` to `0` to disable automatic snapshots.

To reset a local demo cluster, stop all nodes and remove the generated state files:

```bash
rm raft_state_*.json
```

## Web Dashboard

The UI in `ui/` is served by every node at `/`.

It:

- polls `/v1/status` every 2 seconds,
- loads initial data from `/v1/keys`,
- streams committed changes from `/v1/watch`,
- sends writes with client idempotency headers,
- warns when the current node is not leader.

## Testing

Run all tests with:

```bash
go test ./...
```

In restricted environments where the default Go build cache is read-only, use a writable cache:

```bash
CCACHE_DISABLE=1 GOCACHE=/tmp/pulsekv-go-build go test ./...
```

At the time this README was updated, the Raft tests passed, but `kvstore` had two failing tests: `TestApplyInvalidCommand` and `TestApplyMalformedPut`. Those tests expect validation errors that `KVStore.Apply` currently does not return.

The Raft and KV test files cover node defaults, role transitions, election timing, append entries behavior, persistence, snapshots, and concurrent KV access.

## Development Notes

- Internal Raft traffic uses TCP `net/rpc`; client traffic uses HTTP.
- `api.Server` owns HTTP routes, static file serving, and SSE fan-out.
- `main.go` wires together the Raft node, transport, KV store, watcher, snapshot restore, apply loop, and graceful shutdown.
- `raft.MemoryPersister` is available for tests; `raft.FilePersister` is used by running nodes.
- Checked-in binaries such as `pulsekv`, `pulsectl`, and `PulseKV` are build artifacts. Rebuild them locally from source when needed.
