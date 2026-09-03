---
title: Remote Database
---

Juno normally reads and writes its own database on local disk. It can instead
read from another Juno instance over gRPC, so that one node owns the data and
one or more others use it without keeping a copy.

<div className="topology">
<svg viewBox="0 0 680 190" role="img" aria-label="Node A syncs from Starknet and serves its database over gRPC on port 6064. Node B reads from Node A and serves JSON-RPC.">
  <defs>
    <marker id="rdb-arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="currentColor" />
    </marker>
  </defs>

  <text x="94" y="18" textAnchor="middle" className="rdb-cap">Starknet</text>
  <line x1="94" y1="30" x2="94" y2="62" stroke="currentColor" strokeWidth="1.25" strokeDasharray="3 3" markerEnd="url(#rdb-arrow)" opacity="0.55" />

  <rect x="14" y="66" width="220" height="86" rx="5" fill="none" stroke="currentColor" strokeWidth="1.25" />
  <text x="30" y="92" className="rdb-title">Node A</text>
  <text x="30" y="114" className="rdb-line">syncs, owns the database</text>
  <text x="30" y="134" className="rdb-flag">--grpc</text>

  <line x1="446" y1="109" x2="238" y2="109" stroke="currentColor" strokeWidth="1.25" markerEnd="url(#rdb-arrow)" />
  <text x="342" y="100" textAnchor="middle" className="rdb-cap">gRPC :6064</text>
  <text x="342" y="128" textAnchor="middle" className="rdb-cap rdb-dim">read only</text>

  <rect x="450" y="66" width="216" height="86" rx="5" fill="none" stroke="currentColor" strokeWidth="1.25" />
  <text x="466" y="92" className="rdb-title">Node B</text>
  <text x="466" y="114" className="rdb-line">no local database</text>
  <text x="466" y="134" className="rdb-flag">--remote-db</text>

  <text x="558" y="18" textAnchor="middle" className="rdb-cap">JSON-RPC clients</text>
  <line x1="558" y1="62" x2="558" y2="30" stroke="currentColor" strokeWidth="1.25" strokeDasharray="3 3" markerEnd="url(#rdb-arrow)" opacity="0.55" />
</svg>
</div>

Node A syncs from Starknet and serves its database. Node B keeps no database of
its own and answers RPC from A's data.

## When to use it

The reading node has nothing to sync and no second copy of the database on
disk, so it starts as soon as it can read from the serving node. That makes it
useful for adding RPC capacity beside a node that is already synced, or for a
short-lived instance you do not want to provision storage for. Note that a
serving node which is reachable but empty is enough for the reading node to
start, so check that A is synced rather than merely up.

The cost is that the reading node has two dependencies, not one. If A stops, B
has no data. B also polls the Starknet feeder gateway once a minute for the
latest block header, so it needs outbound access to the gateway as well as to
A. Every read from B is a network round trip rather than a local disk read, so
latency is higher and it grows with the distance between the two.

## Serving a database

On the node that owns the data, enable the gRPC server.

```bash
juno --grpc --grpc-host 0.0.0.0 --grpc-port 6064
```

| Flag | Default | Description |
| - | - | - |
| `grpc` | `false` | Enable the gRPC server |
| `grpc-host` | `localhost` | Interface the gRPC server listens on |
| `grpc-port` | `6064` | Port the gRPC server listens on |

The default host only accepts connections from the same machine. Set
`--grpc-host` to an interface the reading node can reach. Juno sets no limit on
how many readers may attach to one serving node.

:::warning
The gRPC connection is neither encrypted nor authenticated, and Juno offers no
option to change that. Anyone who can reach the port can read the entire
database, and anyone on the network path can read the traffic. Keep it on a
private network and never expose the gRPC port to the internet.
:::

## Reading from a remote database

On the reading node, point `--remote-db` at the serving node.

```bash
juno --remote-db <host>:6064 --http --http-port 6060
```

`--remote-db` replaces the local database entirely, so `--db-path` is unused and
the node keeps no local database. The connection is read only. Every write
through it is rejected, so a reading node cannot modify the data it is served.

The reading node still runs its synchronizer, but only to follow the chain
head, and it fetches and verifies no blocks of its own. Because it needs that
much, `--remote-db` cannot be combined with `--disable-sync`. Juno rejects the
pair at startup.

:::caution
Head tracking is what `/ready` and `/ready/sync` report on. If the reading node
cannot reach the feeder gateway, it never learns the latest block header and
both endpoints return 503 indefinitely, even though reads from the serving node
are working normally. A readiness probe pointed at either one will never pass.
Use `/ready/rpc` on a reading node, which reports on the database rather than on
sync progress, or allow outbound access to the gateway.
:::

## What this is not

The gRPC server exposes a single service with two methods, `Version` and `Tx`.
It exists so one Juno instance can serve its database to another, and Juno is
its only consumer. It is not a general purpose indexing or state query
interface, and it is not intended for third party clients.
