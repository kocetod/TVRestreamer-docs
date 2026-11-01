# 04 · Data Model

[← Index](00-README.md) · [Български](../bg/04-data-model.md)

## 1. ER diagram

```mermaid
erDiagram
    NODE ||--o{ STREAM : "runs"
    STREAM ||--|{ STREAM_INPUT : "has"
    STREAM ||--o{ STREAM_OUTPUT : "has"
    STREAM }o--|| CATEGORY : "in"
    STREAM ||--o{ PACKAGE_STREAM : ""
    PACKAGE ||--o{ PACKAGE_STREAM : ""
    PACKAGE ||--o{ SUBSCRIBER_PACKAGE : ""
    SUBSCRIBER ||--o{ SUBSCRIBER_PACKAGE : ""
    SUBSCRIBER ||--o{ DEVICE : "has"
    SUBSCRIBER ||--o{ SESSION_HISTORY : ""
    STREAM ||--o{ STREAM_EVENT : ""
    STREAM ||--o{ SESSION_HISTORY : ""
    ADMIN_USER ||--o{ AUDIT_LOG : "performs"
    ADMIN_USER ||--o{ API_KEY : "owns"
    ALERT_RULE ||--o{ ALERT_EVENT : "fires"
    NOTIFICATION_CHANNEL ||--o{ ALERT_RULE : "notifies via"

    NODE {
        uuid id PK
        string name
        string role "INGEST|EDGE|ALL_IN_ONE"
        string public_host
        string status "ONLINE|OFFLINE|DRAIN"
        int max_streams
        string agent_version
        timestamptz last_seen_at
    }
    STREAM {
        uuid id PK
        string name
        string slug UK
        int lcn
        uuid category_id FK
        uuid node_id FK
        string logo_url
        string epg_id
        bool enabled
        string mode "ALWAYS_ON|ON_DEMAND"
        jsonb processing
        bool return_to_primary
        int return_after_sec
        int version
        timestamptz deleted_at
    }
    STREAM_INPUT {
        uuid id PK
        uuid stream_id FK
        int priority
        string type "HTTP_TS|HLS|UDP|SRT|RTMP|RTSP|DVB"
        string url
        jsonb options
        bool hot_standby
        int timeout_sec
        bool enabled
    }
    STREAM_OUTPUT {
        uuid id PK
        uuid stream_id FK
        string type "HLS|HTTP_TS|UDP|SRT|RTMP"
        jsonb options
        bool public
        bool enabled
    }
    CATEGORY {
        uuid id PK
        string name
        int sort_order
    }
    PACKAGE {
        uuid id PK
        string name
        string description
        bool active
    }
    PACKAGE_STREAM {
        uuid package_id FK
        uuid stream_id FK
        int sort_order
    }
    SUBSCRIBER {
        uuid id PK
        string username UK
        string password_hash
        string token UK
        string full_name
        string phone
        string email
        string status "ACTIVE|SUSPENDED|EXPIRED|BANNED"
        date expires_at
        int max_connections
        string limit_policy "REJECT_NEW|KICK_OLDEST"
        cidr[] allowed_ips
        string[] allowed_countries
        text note
    }
    SUBSCRIBER_PACKAGE {
        uuid subscriber_id FK
        uuid package_id FK
    }
    DEVICE {
        uuid id PK
        uuid subscriber_id FK
        string fingerprint
        string user_agent
        inet last_ip
        timestamptz last_seen_at
    }
    SESSION_HISTORY {
        bigint id PK
        uuid subscriber_id FK
        uuid stream_id FK
        uuid edge_node_id FK
        inet ip
        string country
        string user_agent
        timestamptz started_at
        timestamptz ended_at
        bigint bytes_sent
        string end_reason
    }
    STREAM_EVENT {
        bigint id PK
        uuid stream_id FK
        timestamptz ts
        string type "STATUS|FAILOVER|PROCESS_EXITED|…"
        uuid node_id
        uuid input_id
        string message
        jsonb data
    }
    ADMIN_USER {
        uuid id PK
        string email UK
        string password_hash
        string role "ADMIN|OPERATOR|SUPPORT|VIEWER"
        string totp_secret
        bool active
    }
    API_KEY {
        uuid id PK
        uuid owner_id FK
        string prefix
        string hash
        string[] scopes
        timestamptz expires_at
    }
    AUDIT_LOG {
        bigint id PK
        uuid actor_id FK
        string actor_email
        inet ip
        string action
        string entity_type
        string entity_id
        jsonb diff
        timestamptz ts
    }
    ALERT_RULE {
        uuid id PK
        string name
        string condition
        jsonb params
        uuid channel_id FK
        bool enabled
    }
    ALERT_EVENT {
        bigint id PK
        uuid rule_id FK
        string severity
        string state "FIRING|RESOLVED|ACK"
        jsonb data
        timestamptz ts
    }
    NOTIFICATION_CHANNEL {
        uuid id PK
        string type "TELEGRAM|EMAIL|WEBHOOK"
        jsonb config
    }
```

## 2. Table notes

| Table | Notes |
|---|---|
| `stream` | `version` increments on every change and is sent to the agent (desired state). `deleted_at` enables soft delete (BR-02). Indexes: `slug` (unique), `node_id`, `enabled`. |
| `stream_input.options` | Type-specific JSON, e.g. `{"headers":{"User-Agent":"..."}, "iface":"eth1", "ssm_source":"10.0.0.1", "srt":{"mode":"caller","latency":200,"passphrase":"***"}, "analyzeduration":1.5}` (`analyzeduration` — seconds FFmpeg probes the input, 0.5–30, default 1.5; TVR-021). Secrets are encrypted (AES-GCM, key from env). |
| `stream.return_to_primary`, `return_after_sec` | Return to primary (FR-FOV-05, V8, TVR-052): `true` and 30 s by default (5..3600). |
| `stream.processing` | `{"pnr":101, "audio":{"keep_langs":["bul","eng"]}, "drop_subtitles":true, "service_name":"BTV HD"}` |
| `stream_output.options` | HLS: `{"segment_sec":4, "list_size":6}`; UDP: `{"addr":"239.1.1.10","port":1234,"ttl":4,"iface":"eth1"}` |
| `subscriber.token` | 32-character base62, generated with `SecureRandom`. Unique index. |
| `subscriber.expires_at` | Last day of access (Europe/Sofia); `NULL` = never expires. Partial index for the nightly job (only `ACTIVE` with a date, BR-01). `password_hash` is optional (Argon2id) — only for Xtream-like access. |
| `device.fingerprint` | Hash of the device ID if the player sends one, otherwise of user agent + IP. Unique per subscriber (BR-06). |
| `session_history` | Monthly partitioning (`PARTITION BY RANGE (started_at)`), 90-day retention. |
| `stream_event` | Monthly partitioning (UTC, `stream_event_YYYY_MM`, function `tvr_stream_event_partition`), 180-day retention: every night next month is created and whole months that ended more than 180 days ago are dropped (TVR-029). `STATUS` = state change (`data.from`/`to`); `FAILOVER` = input switch (`data.from`/`to` — input ids, `reason`); `RECOVERED` = automatic return to a higher-priority input (same data, TVR-052). Indexes: `(stream_id, ts)` for a channel's history and `(ts DESC)` for the dashboard's latest events (V4, TVR-034). |
| `audit_log` | Append-only, 1-year retention (V6, TVR-040). `actor_email` keeps the admin's e-mail at the time of the change (`actor_id` becomes `NULL` when the admin is deleted); `NULL` = the system. `diff` = what changed (`{field: {from, to}}`), never secrets (passwords, tokens). Indexes: `(entity_type, entity_id, ts DESC)` for an object's history and `(ts DESC)` for the Audit screen (TVR-074). For now only subscriber changes are written. |
| `stream_logo` | Uploaded channel logo (V7, TVR-047), served at `/logos/{stream_id}.{png\|svg}?v=<sha256[0..8]>`; `stream.logo_url` holds that path or an external `http(s)` address. Saving another address drops the uploaded logo. |

## 3. Data in Redis

| Key | Type | TTL | Contents |
|---|---|---|---|
| `sess:{subscriberId}` | ZSET | 60 s | Active devices: `sessionKey → last request (ms)`; active for 30 s after it (TVR-045) |
| `sess:start:{subscriberId}` | ZSET | 60 s | `sessionKey → start (ms)`; `KICK_OLDEST` closes the oldest |
| `sess:info:{subscriberId}` | HASH | 60 s | `sessionKey → JSON(stream, ip, ua)` — for the session list (FR-SES-01) |
| `sess:kicked:{sessionKey}` | STRING | 2 min | A device closed by `KICK_OLDEST` — refused (`LIMIT`) so it does not kick the new one |
| `auth:{token}` | STRING (JSON) | 10 s | Cached subscriber rights: `{subscriberId, status, expiresAt, slugs, allowedIps, maxConnections, limitPolicy}`; `"-"` = unknown token (TVR-042). Dropped on every change of the subscriber |
| `stream:status:{id}` | STRING (JSON) | 30 s | Last status from the agent: state, activeInputId, bitrateIn, since, lastError, nodeId, at (TVR-020) |
| `node:metrics:{id}` | STRING (JSON) | 30 s | Last metrics of the server (the agent sends them every 5 s): CPU, RAM, network, work dir disk, processes, load (TVR-035); same as `node.metrics` |
| `rl:auth:{ip}` | STRING | 60 s | Rate limit for failed attempts |
| channel `events` | PUB/SUB | — | Events for UI SSE clients (TVR-030): message `<type>\n<JSON>`, e.g. `stream.status`; every core instance passes them to its own clients |

## 4. Subscriber lifecycle

```mermaid
stateDiagram-v2
    [*] --> ACTIVE: create
    ACTIVE --> SUSPENDED: manual suspend
    SUSPENDED --> ACTIVE: resume
    ACTIVE --> EXPIRED: expires_at < today (cron 00:00)
    EXPIRED --> ACTIVE: extend
    ACTIVE --> BANNED: violation
    SUSPENDED --> BANNED
    BANNED --> ACTIVE: manual (Admin only)
    EXPIRED --> [*]: delete after 180 days (optional)
```

## 5. Stream lifecycle

```mermaid
stateDiagram-v2
    [*] --> STOPPED
    STOPPED --> STARTING: start / first viewer (on-demand)
    STARTING --> ONLINE: valid TS + video/audio
    STARTING --> OFFLINE: no working input
    ONLINE --> DEGRADED: on backup input / errors > threshold
    DEGRADED --> ONLINE: back on primary / errors stop
    ONLINE --> OFFLINE: all inputs down
    DEGRADED --> OFFLINE
    OFFLINE --> ONLINE: input recovers
    ONLINE --> STOPPED: stop / idle timeout
    DEGRADED --> STOPPED
    OFFLINE --> STOPPED
    STARTING --> ERROR: configuration error
    ERROR --> STOPPED: fixed
```

## 6. Migrations

- Flyway, files `V{n}__{description}.sql` in `tvr-core/src/main/resources/db/migration`.
- Enumerated values are `varchar` + `CHECK` (not PostgreSQL `ENUM`) — a new value is added by a migration without locking the type.
- Tables have `created_at`/`updated_at` (trigger `tvr_touch_updated_at`).

| Version | Ticket | Contents |
|---|---|---|
| V1 | TVR-005 | `node`, `category`, `stream` (slug unique among non-deleted rows, soft delete), `stream_input` (priority 1–8, network URLs only — SEC-07), `stream_output` |
| V2 | TVR-006 | `admin_user` (case-insensitive unique email, roles ADMIN/OPERATOR/SUPPORT/VIEWER), `admin_refresh_token` (SHA-256 hash only, rotation) |
| V5 | TVR-037 | `package` (case-insensitive unique name), `package_stream` (`sort_order`, unique per package with a deferred check), `subscriber` (case-insensitive unique `username`, `token` — 32 base62 chars, statuses, `max_connections` 1–100, `limit_policy`, `allowed_ips cidr[]`, `allowed_countries` ISO 3166-1 alpha-2), `subscriber_package`, `device` (`fingerprint` unique per subscriber) |
| V7 | TVR-047 | `stream_logo` (one uploaded logo per channel: `content_type` PNG/SVG, `data bytea` ≤ 512 KB, `sha256`; cascade with the channel) |
- Every migration must be backward compatible with at least the previous code version (zero-downtime deploy).
