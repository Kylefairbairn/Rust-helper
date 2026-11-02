# Debugging NATS JetStream “No Leader” / 503 Error 10008

## Symptom this guide covers

A NATS client reaches a server, but a JetStream request fails with an error like:

```text
JetStream system temporarily unavailable
API error 503
error code 10008
```

The server log also reports that it cannot elect a JetStream leader. Four of five boards are powered on, so the expectation is that the cluster still has quorum.

## What the error means

The NATS client has received a JetStream API response. This is different from a TCP connection failure or an authentication rejection. The JetStream metadata Raft group has no elected leader, so the server cannot service JetStream API requests.

Core NATS and JetStream are separate layers. Core publish/subscribe may still work while JetStream is unavailable. A successful Core NATS connection does not prove JetStream is healthy.

A five-member Raft group needs three members for a majority. Therefore, four of five *actual members of the relevant Raft group*, able to communicate, should be enough. The number of boards powered on is not by itself proof that four members of the JetStream metadata group are participating.

## First: avoid making recovery harder

Until you understand the peer state:

- Do not delete JetStream storage directories.
- Do not copy one board’s JetStream store onto another board.
- Do not change server names, JetStream cluster names, or replica settings as a trial fix.
- Do not start two servers against the same JetStream storage directory.
- Do not repeatedly restart all nodes at once. Preserve logs and state first.

Each JetStream-enabled server needs its own persistent local storage directory. JetStream is not designed to use a shared filesystem as a standby mechanism.

## Step 1: separate Core NATS connectivity from JetStream readiness

If anonymous access is enabled, use the NATS CLI for a Core NATS test. Run the subscriber in one terminal:

```bash
nats --server nats://<node-ip>:4222 sub diag.core
```

In another terminal, publish:

```bash
nats --server nats://<node-ip>:4222 pub diag.core hello
```

If your server requires authentication, supply the appropriate client credentials. These commands test ordinary pub/sub; they do not test JetStream health.

Interpretation:

- **Core publish/subscribe works, JetStream returns 503/10008:** client connectivity is working; focus on JetStream metadata leadership and cluster state.
- **The CLI cannot connect at all:** first investigate the client endpoint, port, network path, TLS, and authentication. That is a separate failure from error 10008.

## Step 2: use the monitoring HTTP port without a NATS user

The monitoring port is separate from the NATS client port. If the server has monitoring enabled (commonly port `8222`) and it is reachable from your workstation, it can be queried with `curl`; it does not require a NATS system user unless your monitoring endpoint has been secured.

Check JetStream state on each running board:

```bash
curl -s http://<node-ip>:8222/jsz | jq '.meta_cluster'
```

Check which NATS cluster routes each board sees:

```bash
curl -s http://<node-ip>:8222/routez \
  | jq -r '[.routes[].remote_name] | unique | .[]'
```

Check health status:

```bash
curl -s -i 'http://<node-ip>:8222/healthz?js-enabled-only=true'
```

Also capture the full responses if `jq` is unavailable:

```bash
curl -s http://<node-ip>:8222/jsz
curl -s http://<node-ip>:8222/routez
```

Run the checks against all four online boards and compare results. `/routez` may list multiple route connections for one peer in newer server versions, so count unique `remote_name` values, not raw route entries. For a full mesh among four online nodes, each should see the other three online nodes.

On `/jsz`, inspect the `meta_cluster` object for the leader and peer information that your NATS version exposes. If there is no leader, or peer membership includes more members than expected, that is strong evidence that the problem is in metadata-group membership or peer communication.

## Step 3: inspect logs on every online board

Collect a window of logs from all four active boards around startup and the failed client request. Search for phrases such as:

```text
cannot elect JetStream leader
JetStream has not established contact with a meta leader
no metadata leader
No cluster meta leader found
```

Also note any messages about:

- expected member counts versus responding members
- a peer being offline or not reachable
- route connection failures or authorization failures on routes
- duplicate server names or changed server identity
- JetStream storage initialization, permissions, or disk errors

A log saying, for example, that the metadata cluster expects six nodes is important even if your physical deployment has five boards: it means JetStream's stored membership and your current inventory do not match.

## Step 4: compare configuration across boards

Compare the effective configuration on all five boards (including the powered-off one if possible). Confirm:

1. **JetStream is enabled on every intended JetStream member.**
2. **The NATS cluster name is identical** on all members. The JetStream cluster name must also match wherever configured.
3. **Each server has a unique, stable server name.** Do not clone a disk image with an already-initialized JetStream store and leave the same identity on multiple boards.
4. **Each server uses its own persistent `store_dir`.** It must survive restarts and must never be shared concurrently with another board.
5. **Cluster routes are reachable in both directions.** Client access on `4222` does not prove the server-to-server cluster port (commonly `6222`) is reachable.
6. **Any replica or placement constraints are consistent.** A stream's replica count and placement are not simply the number of powered-on boards; inspect the actual peer set for the affected group.
7. **No board was renamed, reprovisioned, or given a fresh store while old peer membership remains.** Server identity changes can leave JetStream expecting old peers.

Keep the exact NATS server version and startup command/config from each board. Configuration differences are easy to miss when boards are provisioned separately.

## Step 5: understand which quorum is missing

JetStream has separate Raft groups:

- **Metadata group:** manages JetStream API state and stream placement. If it has no leader, JetStream API calls can return 503/10008 across the cluster.
- **Stream group:** replicates a particular stream's data. A stream can be unavailable even if the metadata group has a leader.
- **Consumer group:** replicates consumer state and acknowledgements.

The reported inability to elect a JetStream leader points first to the metadata group. Do not diagnose it by counting only the replicas of one stream, or by counting physical boards without checking the metadata peer group.

## Why four of five can still show no leader

If four boards are powered on but there is no metadata leader, investigate these possibilities:

- The metadata group’s stored membership is not five (for example, it still expects old or replaced server identities).
- Fewer than four boards are actually joined to the same NATS route mesh; the client port may work while the route port is blocked or misconfigured.
- The boards have different NATS/JetStream cluster names or different server identities.
- The boards are running but JetStream failed locally because its storage directory is missing, unwritable, full, or not persistent.
- A network partition means the four powered boards cannot all communicate with each other.
- The client is connecting to a different NATS cluster, domain, or endpoint than the one whose logs you are checking.

If the metadata group truly has five members and four of those members are healthy and mutually connected, a missing board alone should not prevent a majority. At that point, compare the exact metadata peer state and logs rather than changing the expected quorum.

## The CLI “user required” message

Commands under `nats server ...` are operator commands. They query server/system information and generally require credentials for a configured `$SYS` system account. If you did not configure a system user, those operator commands may be unavailable; that does not prove that ordinary NATS clients cannot connect.

Use the monitoring endpoints above for read-only diagnostics when enabled. For Core NATS, use `nats pub` and `nats sub` only if the server allows the credentials or anonymous access you supply. Do not add broad anonymous permissions just to make the operator CLI work.

## Minimal diagnostic bundle to capture

Before making changes, save:

- NATS server version and startup config for every board
- the complete leader-election log excerpt from each online board
- `/jsz` response from each online board
- `/routez` response from each online board
- whether Core `pub`/`sub` works on each node
- which JetStream data directory is used by each board, and whether it is persistent and unique
- the exact client error and which URL it connects to

Redact credentials, tokens, private keys, and sensitive IPs before sharing logs or config.

## References

- [NATS monitoring endpoints (`/jsz`, `/routez`, `/healthz`)](https://docs.nats.io/learn/monitoring/monitoring-endpoints)
- [NATS clustering and replication](https://docs.nats.io/learn/clustering/)
- [JetStream clustering configuration and quorum](https://docs.nats.io/running-a-nats-service/configuration/clustering/jetstream_clustering)
- [NATS server configuration reference](https://docs.nats.io/running-a-nats-service/configuration)
