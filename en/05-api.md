# 05 · API Specification

[← Index](00-README.md) · [Български](../bg/05-api.md) · Machine-readable contract: [openapi.yaml](../api/openapi.yaml)

## 1. General rules

| Topic | Rule |
|---|---|
| Base URL | `https://admin.example.com/api/v1` |
| Format | JSON (UTF-8), ISO-8601 UTC dates, UUID IDs |
| Authentication | `Authorization: Bearer <JWT>` (admin session) or `X-API-Key: tvr_xxx` (integrations) |
| Pagination | `?page=0&size=50&sort=name,asc` → `{items, page, size, total}` |
| Filters | Query parameters, e.g. `?status=ONLINE&nodeId=...&q=btv` |
| Errors | RFC 9457 Problem Details: `{type, title, status, detail, errors:[{field, message}]}` |
| Versioning | In the path (`/v1`). Breaking changes → `/v2` |
| Optimistic locking | `version` in the body; on conflict → `409 Conflict` |
| Rate limit | 20 requests/s per API key; `429` with `Retry-After` |
| Real time | `GET /api/v1/events` (Server-Sent Events) |

## 2. Endpoints

### 2.1 Admin authentication

| Method | Path | Description |
|---|---|---|
| POST | `/auth/login` | `{email, password, totp?}` → `{accessToken, refreshToken}` |
| POST | `/auth/refresh` | New access token |
| POST | `/auth/logout` | Revokes refresh token |
| GET | `/auth/me` | Current user and role |
| POST | `/auth/2fa/setup` · `/auth/2fa/verify` | TOTP setup |

### 2.2 Streams

| Method | Path | Description | Role |
|---|---|---|---|
| GET | `/streams` | List (filters: `status, categoryId, nodeId, packageId, q`) | Viewer+ |
| POST | `/streams` | Create (with inputs and outputs) | Operator+ |
| GET | `/streams/{id}` | Detail + current status | Viewer+ |
| PUT | `/streams/{id}` | Full update (`version` required) | Operator+ |
| DELETE | `/streams/{id}` | Soft delete | Operator+ |
| POST | `/streams/{id}/actions/start` · `stop` · `restart` | Control (TVR-021): `start`/`stop` set `enabled` (kept across restarts), `restart` restarts the processes on the agent. 202 + the channel; 409 — `restart` of a stopped channel, without a node or with a disconnected node; 404 — unknown action | Operator+ |
| POST | `/streams/{id}/actions/switch-input` | `{inputId, pin: bool}` | Operator+ |
| POST | `/streams/{id}/clone` | Clone | Operator+ |
| GET | `/streams/{id}/analyzer` | Bitrate, PIDs with codecs and languages, CC errors (live, every 5 s; TVR-026) | Viewer+ |
| GET | `/streams/{id}/metrics?from&to&step` | Time series (bitrate, CC, viewers) | Viewer+ |
| GET | `/streams/{id}/events` | Event history, newest first: `FAILOVER`, `STATUS`, … Filters `type` (several), `from`, `to` (ISO-8601), `page`, `size` | Viewer+ |
| GET | `/streams/{id}/logs?tail=500` | Last log lines from the agent (1–1000, TVR-023; SSE `/logs/stream` later) | Operator+ |
| GET | `/streams/{id}/thumbnail.jpg` | Latest thumbnail | Viewer+ |
| POST | `/streams/bulk` | `{ids[], action: start\|stop\|move\|addToPackage, params}` | Operator+ |
| POST | `/inputs/probe` | `{type, url, timeoutSec?, options?}` → ffprobe result within 10 s (FR-INP-07, TVR-022) | Operator+ |

**Example — create a stream:**

```json
POST /api/v1/streams
{
  "name": "BTV HD",
  "slug": "btv-hd",
  "lcn": 3,
  "categoryId": "8f0c…",
  "nodeId": "a12e…",
  "mode": "ALWAYS_ON",
  "enabled": true,
  "epgId": "btv.bg",
  "inputs": [
    {"priority": 1, "type": "HTTP_TS", "url": "http://provider-a/btv.ts", "timeoutSec": 5, "hotStandby": false},
    {"priority": 2, "type": "UDP", "url": "udp://239.10.0.3:1234", "options": {"iface": "eth1"}, "hotStandby": true}
  ],
  "outputs": [
    {"type": "HLS", "options": {"segmentSec": 4, "listSize": 6}, "public": true},
    {"type": "HTTP_TS", "public": true}
  ],
  "packageIds": ["c4a1…"]
}
```

```json
201 Created
{
  "id": "5b7d…", "slug": "btv-hd", "version": 0,
  "status": {"state": "STOPPED", "activeInputId": null, "bitrateIn": null, "since": null, "lastError": null},
  "urls": {"hls": "https://tv.example.com/live/btv-hd/index.m3u8", "ts": "https://tv.example.com/ts/btv-hd.ts"}
}
```

> The response contains the whole channel (inputs and outputs with `id`). `version` starts at 0 and grows by 1 with every change. `status` is the runtime state reported by the agent (TVR-020): `STARTING` → `ONLINE` (with `bitrateIn` in bits/s, `activeInputId`, `since`) or `OFFLINE`/`ERROR` with `lastError`; `STOPPED` when the channel is disabled or no report came in the last 30 s. `?status=` filters the list by it (an unknown value → 400). Errors: 400 with `errors:[{field, message}]` (e.g. `{"field": "inputs[0].url", "message": "Неподдържан протокол"}`), 409 for a taken slug or a stale `version`.

**Analyzer (TVR-026):**

```json
GET /api/v1/streams/5b7d…/analyzer

200 OK
{
  "streamId": "5b7d…", "nodeId": "a12e…", "activeInputId": "2fc3…", "at": "2026-10-26T12:57:45.469Z",
  "bitrate": 2225493, "ccErrorsPerMin": 0, "scrambled": false,
  "pids": [
    {"pid": 0,    "type": "PSI",   "codec": null,   "language": null, "bitrate": 12633,   "ccErrors": 0, "scrambled": false},
    {"pid": 256,  "type": "VIDEO", "codec": "h264", "language": null, "bitrate": 2056561, "ccErrors": 0, "scrambled": false},
    {"pid": 257,  "type": "AUDIO", "codec": "aac",  "language": "bul", "bitrate": 141021, "ccErrors": 0, "scrambled": false},
    {"pid": 4096, "type": "PSI",   "codec": null,   "language": null, "bitrate": 12633,   "ccErrors": 0, "scrambled": false}
  ],
  "video": {"pid": 256, "type": "VIDEO", "codec": "h264", …},
  "audio": [{"pid": 257, "type": "AUDIO", "codec": "aac", "language": "bul", …}]
}
```

> The agent's TsSwitcher analyses the active input: bitrate of the whole TS and of every PID (5 s average), codecs and languages from the PMT, CC errors, scrambling. It comes with the channel status (every 5 s) and is kept in Redis for 30 s. `at: null` (and empty `pids`) = no data: the channel is stopped, still starting, or the agent does not report. 404 for an unknown channel.

**Channel log (TVR-023):**

```json
GET /api/v1/streams/5b7d…/logs?tail=3

200 OK
{
  "streamId": "5b7d…", "nodeId": "a12e…", "tail": 3, "found": true,
  "lines": [
    {"at": "2026-10-26T09:00:00.250Z", "level": "INFO", "source": "agent", "text": "btv-hd/input-1 started (pid 4242): ffmpeg … -i http://user:****@provider/btv.ts …"},
    {"at": "2026-10-26T09:05:12.031Z", "level": "WARN", "source": "agent", "text": "btv-hd/input-1 exited with code 137 after 5m11s; restart in 1s"},
    {"at": "2026-10-26T09:05:13.118Z", "level": "INFO", "source": "agent", "text": "btv-hd/input-1 started (pid 4371): …"}
  ]
}
```

> Core asks the agent of the channel's node (gRPC `FetchLogs`) and waits up to 5 s. The agent keeps the last 1 000 lines per channel (FFmpeg output + start/exit/restart events), also after the channel stops, until the agent restarts. URLs in the lines have their passwords/tokens masked. `level` is a guess from the words in the line. Errors: 400 (`tail` outside 1–1000), 404, 409 (no node / node not connected), 504 (the agent did not answer within 5 s).

**Input probe (TVR-022):**

```json
POST /api/v1/inputs/probe
{"type": "HTTP_TS", "url": "http://localhost:9001/a.ts"}

200 OK
{
  "format": "mpegts", "formatName": "MPEG-TS (MPEG-2 Transport Stream)", "bitrate": 2631336,
  "programs": [{"pnr": 101, "serviceName": "Test A", "provider": "TvRestream", "pmtPid": 4096, "pcrPid": 256, "streams": [0, 1]}],
  "streams": [{"index": 0, "pid": 256, "type": "VIDEO", "codec": "h264", "profile": "Main", "width": 1280, "height": 720, "fps": 25.0, …},
              {"index": 1, "pid": 257, "type": "AUDIO", "codec": "aac", "lang": "bul", "channels": 2, "sampleRate": 48000, "bitrate": 130532, …}],
  "video": {"codec": "h264", "profile": "Main", "width": 1280, "height": 720, "fps": 25.0},
  "audio": [{"codec": "aac", "profile": "LC", "lang": "bul", "channels": 2, "sampleRate": 48000, "bitrate": 130532}],
  "probeMs": 412
}
```

> ffprobe runs on the tvr-core host with a hard limit of 9 s, so the response arrives within 10 s. Errors: 400 with `errors` (the rules of a channel input; DVB cannot be probed); **422** with `code` (`TIMEOUT`, `CONNECTION_REFUSED`, `HOST_NOT_FOUND`, `NETWORK_UNREACHABLE`, `NOT_FOUND`, `ACCESS_DENIED`, `HTTP_ERROR`, `INVALID_DATA`, `NO_STREAMS`, `PROTOCOL_NOT_ALLOWED`, `FAILED`), `reason` (ffprobe's last line) and `probeMs` — `detail` is Bulgarian text, `code` is for the panel's translation; 429 for more than 4 parallel probes; 503 with `code: FFPROBE_MISSING`.

### 2.3 Categories and packages

| Method | Path | Description |
|---|---|---|
| GET/POST | `/categories` | List (by `sortOrder`, then name) / create `{name, sortOrder?}` — without `sortOrder` it goes last |
| GET/PUT/DELETE | `/categories/{id}` | Detail / update / delete (channels stay, without a category) |
| PUT | `/categories/order` | `[id, id, …]` — full order of all categories; returns the new list |
| GET/POST | `/packages` | List / create |
| GET/PUT/DELETE | `/packages/{id}` | Detail / update / delete |
| PUT | `/packages/{id}/streams` | `[{streamId, sortOrder}]` — full ordering |

> Categories (TVR-013): read — any role, change — ADMIN/OPERATOR. The name is unique ignoring case (409 with `errors[].field = name`), up to 100 characters; `sortOrder` 0–100000. The response carries `streamCount` (channels not deleted). With `PUT /categories/order` every category must appear exactly once — otherwise 400.

### 2.4 Subscribers

| Method | Path | Description | Role |
|---|---|---|---|
| GET | `/subscribers` | Filters: `status, packageId, expiresBefore, q` | Support+ |
| POST | `/subscribers` | Create | Support+ |
| GET/PUT | `/subscribers/{id}` | Detail / update | Support+ |
| DELETE | `/subscribers/{id}` | Delete | Admin |
| POST | `/subscribers/{id}/actions/extend` | `{days}` or `{until}` | Support+ |
| POST | `/subscribers/{id}/actions/suspend` · `resume` · `ban` | Status | Support+ |
| POST | `/subscribers/{id}/actions/regenerate-token` | New token, kicks sessions | Support+ |
| GET | `/subscribers/{id}/devices` · DELETE `/devices/{deviceId}` | Devices | Support+ |
| GET | `/subscribers/{id}/sessions` | Active sessions and history | Support+ |
| POST | `/subscribers/import` (CSV) · GET `/subscribers/export` | Bulk operations | Admin |

### 2.5 Sessions

| Method | Path | Description |
|---|---|---|
| GET | `/sessions` | Active sessions (filters: `streamId, subscriberId, nodeId, country`) |
| DELETE | `/sessions/{sessionKey}` | Kick |
| DELETE | `/subscribers/{id}/sessions` | Kick all sessions of a subscriber |

### 2.6 Servers

| Method | Path | Description |
|---|---|---|
| GET | `/nodes` | List (by name): `status` (ONLINE/OFFLINE/DRAIN), `lastSeenAt`, `agentVersion`, `ffmpegVersion`, `connected` (an open gRPC stream), `metrics` — the last metrics of the agent (as in `node.metrics`; `null` when none came in the last 30 s, TVR-035) |
| POST | `/nodes/enrollment-tokens` | Key to enroll a new server (valid 1 h) |
| GET/PUT/DELETE | `/nodes/{id}` | Detail / update / remove |
| POST | `/nodes/{id}/actions/drain` · `undrain` | Maintenance mode |

### 2.7 Monitoring, alerts, audit, settings

| Method | Path | Description |
|---|---|---|
| GET | `/dashboard/summary` | Dashboard (TVR-034): channels per state, problem channels (up to 20), servers, the 10 newest events of 24 h, `egressBps` — network out of all servers with fresh metrics (TVR-035); viewers/subscribers — `null` until their tickets |
| GET | `/events?types=` | SSE (TVR-030): `stream.status`, `stream.failover`, `stream.event`, `node.status`, `node.metrics`; later `session.count`, `alert`. See below | Viewer+ |
| GET/POST/PUT/DELETE | `/alert-rules`, `/notification-channels` | Alert rules and channels |
| POST | `/notification-channels/{id}/test` | Test notification |
| GET | `/alerts?state=FIRING` · POST `/alerts/{id}/ack` | Incidents |
| GET | `/audit?actorId&entityType&from&to` | Audit log |
| GET/PUT | `/settings` | Global settings |
| POST | `/import/m3u` · `/import/astra` | Import (returns preview; `?commit=true` saves) |
| GET/POST | `/config/export` · `/config/import` | Full configuration backup |
| GET | `/reports/{type}?from&to` | `top-streams`, `peak-viewers`, `uptime`, `subscribers` |
| GET/POST/DELETE | `/admin-users`, `/api-keys` | Administrators and keys |

**Real time — `GET /api/v1/events` (TVR-030, FR-MON-02, NFR-10 ≤ 2 s).** Server-Sent Events (`text/event-stream`). The token goes in `Authorization: Bearer …` or — for the browser's `EventSource`, which cannot send headers — in `?access_token=` (on this path only). `?types=` selects the types (comma separated; `stream.*`, `node.*`); without it everything is sent, an unknown type → 400.

```text
event: ready
data: {"at":"2026-10-28T18:01:09Z","until":"2026-10-28T18:16:09Z","types":["node.metrics","node.status","stream.event","stream.failover","stream.status"]}
retry: 3000

event: stream.status
data: {"streamId":"…","state":"ONLINE","previousState":"STARTING","changed":true,"activeInputId":"…","bitrateIn":2236545,"since":"…","lastError":null,"nodeId":"…","at":"…"}

: ping

event: bye
data: {"reason":"token-expired"}
```

| Event | When | `data` |
|---|---|---|
| `ready` | right after connecting | `at`, `until` (when the stream will end), `types` |
| `stream.status` | every report of the agent (≈ 5 s and on change) and Stop through the API | `streamId, state, previousState, changed, activeInputId, bitrateIn, since, lastError, nodeId, at`; `changed` = new state or input |
| `stream.failover` | the agent switched the input | `streamId, type: FAILOVER, nodeId, inputId, message, data{from, to, from_index, to_index, reason}, at` |
| `stream.event` | any other agent event (`PROCESS_EXITED` …) | like `stream.failover` |
| `node.status` | a server went ONLINE / OFFLINE | `nodeId, name, status, lastSeenAt, at` |
| `node.metrics` | metrics of the agent every 5 s (TVR-035); `cpuPercent` −1 = not measured yet | `nodeId, at, cpuPercent, memUsedBytes, memTotalBytes, netRxBps, netTxBps, tmpfsFreeBytes, tmpfsTotalBytes, streamsRunning, processes, loadAvg1m, cpuCores` |
| `bye` | end of the stream | `reason`: `token-expired`, `max-duration` (1 h), `slow-client` (> 1000 queued events), `shutdown` |

- `: ping` every 15 s without events (so proxies keep the connection open).
- The stream ends when the token expires: the client reconnects with a fresh token and reloads its lists through REST — events are not replayed (no `Last-Event-ID`).
- Between core instances the events go through Redis pub/sub (channel `events`, document 04 §3), so every client sees everything. Without Redis the events reach only the clients of the same instance.
- At most 500 open streams per instance (`tvr.live.max-clients`) → 503. nginx in front of core must not buffer (`X-Accel-Buffering: no` is sent).

## 3. Public (subscriber) endpoints — on the edge domain

| Method | Path | Description |
|---|---|---|
| GET | `/pl/{token}.m3u?type=hls\|ts` | Personal playlist |
| GET | `/epg/{token}.xml.gz` | Filtered XMLTV |
| GET | `/live/{slug}/index.m3u8?token=` | HLS (authorized via `auth_request`) |
| GET | `/live/{slug}/{segment}.ts?s=&e=` | Segment (`secure_link`) |
| GET | `/ts/{slug}.ts?token=` | HTTP-TS |
| GET | `/get.php?username&password&type=m3u_plus` | Xtream compatibility (R1) |
| GET | `/player_api.php?username&password&action=` | Xtream compatibility (R1) |
| GET | `/live/{username}/{password}/{streamId}.{ts\|m3u8}` | Xtream compatibility (R1) |

## 4. Internal endpoints

| Method | Path | Called by | Description |
|---|---|---|---|
| GET | `/internal/auth` | nginx `auth_request` | Input headers: `X-Original-URI`, `X-Real-IP`, `User-Agent`. Response `200` + `X-Session-Id`, `X-Sign-Expires` / `401` / `403` + `X-Deny-Reason: EXPIRED\|LIMIT\|NOT_IN_PACKAGE\|IP\|GEO` |
| gRPC | `AgentControl.Connect(stream)` | tvr-agent | Bidirectional stream (see [03-architecture](03-architecture.md) §7) |

## 5. Webhooks (R2)

```json
POST {webhook_url}
X-TVR-Event: stream.failover
X-TVR-Signature: sha256=…   # HMAC-SHA256(body, secret)
{
  "id": "evt_01J…",
  "type": "stream.failover",
  "ts": "2026-10-05T18:21:04Z",
  "data": {"streamId": "5b7d…", "slug": "btv-hd", "from": 1, "to": 2, "reason": "NO_DATA"}
}
```

Retries: 1 min, 5 min, 30 min, 2 h, 6 h. A `2xx` response is expected within 5 s.

## 6. gRPC contract (excerpt)

```protobuf
syntax = "proto3";
package tvr.agent.v1;

service AgentControl {
  rpc Connect(stream AgentMessage) returns (stream CoreMessage);
}

message AgentMessage {
  oneof msg {
    Hello hello = 1;            // nodeId, versions, capabilities
    Heartbeat heartbeat = 2;
    NodeMetrics metrics = 3;
    StreamStatus status = 4;
    StreamEvent event = 5;
    LogLines logs = 6;
    AckDesiredState ack = 7;
  }
}

message CoreMessage {
  oneof msg {
    DesiredState desired = 1;   // full snapshot + version
    Command command = 2;        // RESTART, SWITCH_INPUT, KICK, SUBSCRIBE_LOGS, FETCH_LOGS
  }
}

message StreamStatus {
  string stream_id = 1;
  State state = 2;              // STOPPED, STARTING, ONLINE, DEGRADED, OFFLINE, ERROR
  string active_input_id = 3;
  uint64 bitrate_in = 4;
  uint32 cc_errors_per_min = 5;
  bool scrambled = 6;
  repeated PidInfo pids = 7;
  uint32 http_ts_clients = 8;
}
```

The full contract lives in the code: `TVRestreamer_Code/tvr-common/src/main/proto/tvr/agent/v1/agent.proto` (TVR-008). Java package `io.tvr.agent.proto.v1`; classes are generated by `./gradlew :tvr-common:build`.

| Direction | Message | When |
|---|---|---|
| agent → core | `Hello` | First message after connecting: node_id, versions, capabilities, last applied version, already running streams |
| agent → core | `Heartbeat`, `NodeMetrics` | Periodically (CPU, memory, network, tmpfs) |
| agent → core | `StreamStatus`, `StreamEvent` | Channel state; events (FAILOVER, INPUT_LOST, PROCESS_EXITED…) |
| agent → core | `LogLines` | Only after `SubscribeLogs` |
| agent → core | `AckDesiredState` | Confirms an applied version (+ per-channel errors) |
| core → agent | `DesiredState` | Full snapshot of the channels for this node (`StreamSpec` → `InputSpec`, `OutputSpec`) |
| core → agent | `Command` | `RestartStream`, `SwitchInput`, `KickClient`, `SubscribeLogs` |
| core → agent | `Welcome` | Reply to a valid `Hello` (TVR-017): `node_id`, heartbeat interval (5 s), seconds without a heartbeat before the node becomes `OFFLINE` (15) |

**Link and enrollment (TVR-017, simplified):** the agent opens the stream to `tvr.grpc.port` (9090) with metadata `authorization: Bearer <token>`; token = the agent's `TVR_AGENT_TOKEN` = core's `TVR_AGENT_ENROLL_TOKEN` (otherwise `UNAUTHENTICATED`). The first message must be a `Hello` with `node_name` (otherwise `FAILED_PRECONDITION` / `INVALID_ARGUMENT`); core finds the node by `node_id`, then by name, or creates it (`INGEST`) and sets it `ONLINE`. A heartbeat every 5 s updates `last_seen_at`; no heartbeat for 15 s → `OFFLINE` (FR-NOD-02). A new connection of the same node closes the old one with `ABORTED`. The agent reconnects with backoff 1 → 30 s. One-time keys and mTLS certificates (FR-NOD-01) come later.

**Desired state and reconciler (TVR-018, ADR-006):** right after `Welcome`, after every channel change through the API (after commit) and on a periodic check every 30 s (`tvr.grpc.desired-resync`) core sends a `DesiredState` — every non-deleted channel of the node (channels without `node_id` go to the only `INGEST`/`ALL_IN_ONE` node, if there is exactly one). The version grows only when the content changes: `max(previous + 1, current time in ms)`, so it also grows across core restarts; a stream gets a snapshot only if its version is newer than the last one sent on it. The agent runs channels that are `enabled`, `ALWAYS_ON` and have an enabled input, stops everything else it started itself, and restarts a channel only when what it runs changed (slug, inputs, HLS options). It answers with `AckDesiredState` (`applied=false` + per-channel `errors` on problems); failed channels are retried every 30 s. `Hello` and `Heartbeat` carry the applied version and the running channels. When the link is lost the channels keep running; the last snapshot is kept on the agent's disk and restored after a restart.

**Channel status (TVR-020, FR-STR-07):** the agent sends a `StreamStatus` for every channel of its snapshot on each change and all of them every 5 s (and right after a reconnect). Until the TS analyzer (S2) the state comes from the HLS output: `ONLINE` while the playlist is rewritten (within 2 × segment + 2 s; `bitrate_in` = size of the last 3 segments / their duration), `STARTING` during the start-up grace (input timeout + 2 × segment + 5 s), then `OFFLINE` with `last_error` (age of the data, input restarts, exit code — never FFmpeg output, it may contain the URL); `ERROR` for a channel that could not start; `STOPPED` once when the channel stops running there. Core accepts reports only for channels in the node's snapshot and keeps the last one in Redis `stream:status:<id>` (JSON, TTL 30 s); `since` changes only with the state.

**Compatibility:** field numbers and types never change; removed fields become `reserved`; every enum's zero value is `*_UNSPECIFIED`; an incompatible change means a new package `tvr.agent.v2`.
