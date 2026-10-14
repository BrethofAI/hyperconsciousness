<!-- screenpipe — AI that knows everything you've seen, said, or heard -->
<!-- https://screenpipe.com -->

# Query a trusted node without replicating its brain

A small device can use the existing HC search and MCP interfaces while a trusted
node holds the brain and search index. This is an optional thin transport, not a
required hosted service. The client needs no brain identity, decryption keys,
full history, local index or replica. The trusted node decrypts and searches;
it is not a keyless storage provider.

## Node

Reuse an existing grant-scoped `hc serve-http` deployment or the authenticated
shared MCP endpoint:

```sh
hc mcp --as <read-grant-id> --dir /private/brain \
  --bind 127.0.0.1:7782 --token-file /private/query-node.token
```

Provision a random private token file as described in [hosted access](HOSTED-ACCESS.md).
For access across machines, put this loopback listener behind TLS terminating on
that trusted node. The client requires HTTPS except for numeric loopback HTTP,
which also permits an explicitly configured local SSH tunnel. No public listener,
DNS route, reverse proxy, service or firewall rule is created by client setup.

Keep personal and company endpoints and grants separate. Issue a narrow,
expiring read grant; do not use an owner key as a transport credential. The node
rechecks grants and revocation on reads and retains its existing audit behavior.
A read-only grant rejects writes even though MCP exposes the usual tool names.
Node freshness still depends on ingestion/replication to that node: this API
cannot retrieve updates that never left an offline source device.

## Client

Provision a credential file over an already trusted administrative channel. It
contains the node's existing bearer credential and its **exact endpoint**:

```json
{
  "endpoint": "https://knowledge.example.com",
  "access_token": "<provisioned-node-credential>"
}
```

Keep this file out of Git, command arguments and shared logs. On Unix it must be
owned by the client user with no group/other permissions; use mode 0600. Symlinks
and nonregular files are refused. On Windows protect this operator-provisioned
file with the user's ACL: unlike hosted sessions it is not imported into
Credential Manager. This adapter supports bearer credentials, not device-bound
DPoP. Use a separate grant endpoint/credential per independently revocable
consumer rather than distributing a single broad pairing credential.

A separate, nonsecret connection file selects the node:

```json
{
  "version": 1,
  "name": "hc",
  "origin": "https://knowledge.example.com",
  "routePrefix": "",
  "credentialFile": "/private/node-credential.json",
  "defaultReader": true
}
```

The credential path must be absolute. Its endpoint must match the normalized
origin plus route prefix exactly. Use `routePrefix: ""` for a root `/mcp`
endpoint; the existing hosted login profile still defaults to `/access`.
Each credential is reread per request so atomic file replacement takes effect
without restarting the client. Redirects are refused rather than forwarding
credentials. No certificate-validation bypass is available.

```sh
hc setup node.json
hc status --remote
hc find 'workout plan' --scope hc
hc mcp
```

The opt-in `defaultReader` means unqualified `hc mcp` and `hc find` use this
connection. MCP retains the server's normal tools, arguments, ranking, references,
clipping, cursors and error results; it does not add a `remote_search` tool.
`hc find` displays its familiar `hc  <reference> — <excerpt>` rows using the
server's grant-scoped literal search. MCP offers the existing ranked mode.

Explicit `--as`, `--space`, `--dir`, `HC_DIR` or `BRAINMESH_DIR` retains local
selection. `hc find --local ...` also bypasses the default. `--remote` explicitly
selects the configured connection, including when a store environment variable
exists, but refuses simultaneous local CLI selectors. Existing `hc mcp --remote`
registrations work unchanged. Setup can print registration JSON or use the
existing explicit `--codex` integration. Setup refuses to replace a different
connection; use a separate `HC_CONFIG_DIR` for another configuration.

`find --scope skills` and `--scope pkm` still search installed client files.
`--scope all` combines those with remote HC results, but does not discover secret
capability names remotely. Use an explicit local store for `--scope secrets`.
`ask <grant>`, storage, sync, local status and administrative commands retain their
local contracts; this is a query route, not remote administrative shell access.

For node credentials `hc login`/`logout` do not enter the hosted authentication
flow or pretend to revoke credentials. Revocation and rotation happen at the
issuer. Existing hosted login/refresh/logout behavior is unchanged.

## Failures, privacy and performance

The client never silently falls back to a personal brain, replays a failed
write, persists query/result plaintext, or substitutes stale cached results.
Unauthorized responses, unavailable endpoints and rate limits remain distinct
MCP errors. Invalid JSON, oversized responses and mismatched JSON-RPC IDs are
refused. Upstream HTTP error bodies and token values are not included in transport
diagnostics. Requests cap at 64 KiB, responses at 2 MiB; connect timeout is five
seconds, metadata timeout fifteen seconds, and tool timeout sixty-five seconds.
A TCP accept that stalls during response reading is covered by the overall
request timeout. The process reuses its HTTP agent; keepalive requires upstream
support (the current built-in server closes each response).

Search indexing stays on the large node and its existing encrypted, rebuildable
index is reused across queries. This avoids full pulls before searching; it does
not promise local latency over a WAN. The network RTT and node query time still
apply. Measure cold/warm reads and p50/p95 on the actual route before claiming
latency improvements. The development tests use synthetic records over loopback,
not Scaleway, cellular networks or a production-size corpus.

Offline recent/pinned result caching is **not implemented by this adapter**.
A disconnected client reports unavailability. Existing full replicas still work
locally with an explicit local route. A future offline cache must define expiry,
revocation, retractions, origin binding and stale-result presentation before it
can be treated as authoritative. A query API does not replace independent backups
or synchronization between archive nodes. TLS protects transport to its terminator;
the trusted node and any model receiving retrieved text can see that text.
