# 03 · Architecture

[← Index](00-README.md) · [Български](../bg/03-architecture.md)

## 1. Principles

1. **Control plane ≠ data plane.** `tvr-core` handles configuration, subscribers and authorization. `tvr-agent` and FFmpeg carry the video. If core goes down, running streams and already-authorized sessions keep going.
2. **Video never passes through business logic.** Bytes flow FFmpeg → TS switcher (Netty, zero-copy buffers) → FFmpeg/nginx.
3. **Desired state + reconciliation.** Core sends the agent the full desired state. The agent brings its processes in line on its own (Kubernetes-style), so lost commands and restarts do not cause drift.
4. **One FFmpeg process per input and per packager, never per viewer.**
5. **Stateless edge.** Edge servers are just nginx with cache and scale horizontally.

## 2. C4 · Level 1 — Context

```mermaid
flowchart TB
    subgraph People
      ADM[Admin / Technician / Support]
      SUB[Subscriber<br/>STB · Smart TV · mobile player]
    end
    subgraph External systems
      PROV[Stream providers<br/>HTTP/HLS · SRT · RTMP]
      HEAD[Local headend<br/>UDP multicast · future DVB]
      EPG[EPG provider<br/>XMLTV]
      TG[Telegram / SMTP]
      BILL[Billing / CRM]
    end
    TVR[[TvRestream platform]]
    ADM -- web panel --> TVR
    SUB -- M3U, HLS, HTTP-TS --> TVR
    PROV -- streams --> TVR
    HEAD -- streams --> TVR
    EPG -- XMLTV --> TVR
    TVR -- alerts --> TG
    BILL -- REST API / webhooks --> TVR
```

## 3. C4 · Level 2 — Containers

```mermaid
flowchart LR
    subgraph CP[Control plane — 1 server or VM]
      UI[Admin UI<br/>React SPA]
      CORE[tvr-core<br/>Spring Boot 4 · Java 21]
      PG[(PostgreSQL 16)]
      RD[(Redis 7)]
      PROM[Prometheus + Grafana]
      UI -- REST/SSE --> CORE
      CORE --> PG
      CORE --> RD
      PROM -. scrape .-> CORE
    end
    subgraph IN[Ingest node × N]
      AG[tvr-agent<br/>Spring Boot · Netty]
      FF[FFmpeg processes]
      SHM[(tmpfs /dev/shm/hls)]
      OR[nginx origin]
      AG -- starts/watches --> FF
      FF -- UDP loopback --> AG
      AG -- TS --> FF
      FF -- HLS segments --> SHM
      OR -- reads --> SHM
    end
    subgraph ED[Edge node × M]
      NG[nginx edge<br/>secure_link · auth_request · proxy_cache]
    end
    CORE <-- gRPC bidirectional stream<br/>(agent-initiated) --> AG
    NG -- auth_request --> CORE
    NG -- proxy_pass/cache --> OR
    NG -- HTTP-TS proxy --> AG
    PLAYER[Subscriber players] --> NG
    SRC[Sources] --> FF
    PROM -. scrape .-> AG
```

| Container | Technology | Responsibility |
|---|---|---|
| **Admin UI** | React 18, TypeScript, Vite, Ant Design, TanStack Query, hls.js | Web panel |
| **tvr-core** | Java 21, Spring Boot 4.1, Spring Security, Spring Data JPA, Flyway, gRPC, Micrometer | REST API, subscriber authorization, playlists, agent orchestration, alerts, statistics |
| **PostgreSQL** | 16 | Configuration, subscribers, audit, history |
| **Redis** | 7 | Active sessions, auth cache, rate limiting, status pub/sub |
| **tvr-agent** | Java 21, Spring Boot (no web MVC), Netty, gRPC client | FFmpeg supervisor, TS switcher and analyzer, HTTP-TS relay, node metrics |
| **FFmpeg** | 7.x (+libsrt, openssl) | Input ingest, HLS packaging, UDP/SRT/RTMP outputs |
| **nginx origin** | 1.26 | Serves HLS from tmpfs to edge |
| **nginx edge** | 1.26 | TLS, authorization, cache, delivery to viewers |
| **Prometheus/Grafana** | — | Metrics and dashboards |

## 4. C4 · Level 3 — tvr-agent components

```mermaid
flowchart LR
    subgraph AGENT[tvr-agent]
      SYNC[ControlChannel<br/>gRPC client · desired state]
      REC[Reconciler]
      SUP[ProcessSupervisor<br/>start/stop/restart, backoff]
      subgraph PIPE[StreamPipeline × stream]
        IW1[InputWorker #1<br/>FFmpeg → udp://127.0.0.1:p1]
        IW2[InputWorker #2 hot standby<br/>FFmpeg → udp://127.0.0.1:p2]
        SW[TsSwitcher<br/>Netty UDP · selects active input]
        AN[TsAnalyzer<br/>PAT/PMT/CC/PCR/bitrate]
        FO[FailoverController]
        PK[Packager<br/>FFmpeg → HLS in tmpfs]
        HT[HttpTsRelay<br/>fan-out to clients]
        UO[UDP/SRT outputs<br/>FFmpeg]
      end
      MET[NodeMetrics<br/>CPU/RAM/NIC · Micrometer]
      LOG[LogCollector<br/>FFmpeg stderr → ring buffer]
    end
    SYNC --> REC --> SUP
    SUP --> IW1 & IW2 & PK & UO
    IW1 --> SW
    IW2 --> SW
    SW --> AN
    AN --> FO
    FO -- switch --> SW
    SW -- udp://127.0.0.1:pout --> PK
    SW --> HT
    SW --> UO
    AN -- statistics --> SYNC
    FO -- events --> SYNC
    MET --> SYNC
```

### 4.1 Why a custom TS switcher (TsSwitcher)

If FFmpeg ingests the input directly and produces HLS, failover means restarting the whole process. That causes a 5–10 s interruption and a "broken" playlist. With TsSwitcher:

- inputs are received by separate FFmpeg processes (`-c copy -f mpegts udp://127.0.0.1:PORT`);
- TsSwitcher reads all active inputs but forwards only the selected one to the packager;
- on switch it emits a clean TS (re-sending PAT/PMT), and the packager continues without restart. `EXT-X-DISCONTINUITY` is inserted when needed;
- the analyzer runs on the same packets with no extra copy.

## 5. Data flow for one stream

```mermaid
flowchart LR
    S1[(Primary source)] --> I1[FFmpeg input #1]
    S2[(Backup source)] --> I2[FFmpeg input #2]
    I1 -- MPEG-TS/UDP lo --> SW{TsSwitcher}
    I2 -- MPEG-TS/UDP lo --> SW
    SW --> AN[[TsAnalyzer]]
    SW -- TS --> P[FFmpeg packager<br/>-c copy -f hls]
    P --> H[(/dev/shm/hls/slug/<br/>index.m3u8 + seg_N.ts)]
    SW -- TS --> R[HttpTsRelay :8090/ts/slug]
    SW -- TS --> U[FFmpeg UDP out<br/>239.1.1.x:1234]
    H --> O[nginx origin] --> E[nginx edge] --> V((HLS viewers))
    R --> E2[nginx edge proxy] --> V2((HTTP-TS viewers))
```

## 6. Key sequences

### 6.1 Creating and starting a stream

```mermaid
sequenceDiagram
    autonumber
    actor T as Technician
    participant UI as Admin UI
    participant C as tvr-core
    participant DB as PostgreSQL
    participant A as tvr-agent
    participant F as FFmpeg
    T->>UI: New stream (name, inputs, outputs)
    UI->>C: POST /api/v1/streams
    C->>DB: INSERT stream, inputs, outputs (version=1)
    C->>A: DesiredState(stream, v1) over gRPC
    A->>A: Reconciler: new pipeline
    A->>F: start input #1 (+ hot standby #2)
    A->>F: start HLS packager
    F-->>A: first TS packets
    A->>A: TsAnalyzer: PAT/PMT OK, video+audio OK
    A-->>C: StreamStatus(ONLINE, input=1, bitrate)
    C-->>UI: SSE stream.status = ONLINE
```

### 6.2 Automatic failover

```mermaid
sequenceDiagram
    autonumber
    participant S1 as Primary source
    participant A as tvr-agent (Analyzer + Failover)
    participant SW as TsSwitcher
    participant C as tvr-core
    participant N as Telegram
    S1--xA: stops sending data
    A->>A: 5 s without packets on input #1 (timeout)
    A->>SW: switch(input #2) — hot standby, < 100 ms
    SW->>SW: re-send PAT/PMT, discontinuity flag
    A-->>C: Event FAILOVER(1→2, reason=NO_DATA)
    C->>N: "BTV: switched to backup (no data)"
    loop every 10 s
      A->>S1: probe input #1
    end
    A->>A: input #1 stable for 30 s
    A->>SW: switch(input #1) — return to primary
    A-->>C: Event RECOVERED(2→1)
```

### 6.3 On-demand stream

```mermaid
sequenceDiagram
    autonumber
    participant P as Player
    participant E as Edge
    participant C as tvr-core
    participant A as tvr-agent
    P->>E: GET /live/slug/index.m3u8?token
    E->>C: auth_request
    C->>C: stream is STOPPED (on-demand) → start request
    C->>A: StartStream(slug)
    C-->>E: 200
    E->>E: index.m3u8 missing → retry up to 6 s (nginx try_files + upstream to agent)
    A-->>E: first segments ready
    E-->>P: m3u8
    Note over A: 60 s without viewers → stop
```

## 7. Core ↔ agent communication

| Aspect | Decision |
|---|---|
| Transport | gRPC bidirectional stream, **agent-initiated** (works behind NAT/firewall), mTLS |
| Enrollment | Enrollment token → core issues a client certificate for the agent |
| core→agent messages | `DesiredState` (full snapshot + version), `Command` (restart, switch input, kick HTTP-TS client) |
| agent→core messages | `Heartbeat` (5 s), `NodeMetrics` (5 s), `StreamStatus` (on change + every 5 s), `StreamEvent`, `LogLines` (on subscription) |
| On connection loss | The agent keeps running with the last known state and reconnects with exponential backoff |

## 8. Code structure (monorepo)

```
tvrestream/
├── settings.gradle, build.gradle # Gradle multi-project (Java 21)
├── tvr-common/                  # DTOs, proto (gRPC), shared utilities
│   └── src/main/proto/agent.proto
├── tvr-core/                    # Spring Boot application
│   └── src/main/java/io/tvr/core/
│       ├── stream/              # streams, inputs, outputs
│       ├── subscriber/          # subscribers, packages, devices
│       ├── auth/                # admin auth (JWT) + playback auth
│       ├── playlist/            # M3U, Xtream API, EPG
│       ├── session/             # sessions (Redis)
│       ├── node/                # agent enrollment and orchestration
│       ├── monitoring/          # statuses, SSE, alerts
│       ├── audit/               # audit
│       └── config/              # settings, import/export
├── tvr-agent/                   # Spring Boot (non-web) + Netty
│   └── src/main/java/io/tvr/agent/
│       ├── control/             # gRPC client, reconciler
│       ├── process/             # ProcessSupervisor, FfmpegCommandBuilder
│       ├── ts/                  # TsSwitcher, TsAnalyzer, TsPacket parser
│       ├── relay/               # HttpTsRelay (Netty HTTP)
│       └── metrics/
├── tvr-ui/                      # React + TypeScript
├── deploy/
│   ├── docker-compose.yml
│   ├── nginx/{origin,edge}.conf
│   └── ansible/                 # node installation
└── docs/                        # this documentation
```

## 9. Technology stack and versions

| Layer | Choice | Alternative (rejected) | Reason |
|---|---|---|---|
| Language/framework | Java 21 + Spring Boot 4.1 | Python/FastAPI, Go | Owner's choice; mature ecosystem; virtual threads |
| Media I/O | FFmpeg 7 | GStreamer | More widespread, easier to debug, all protocols |
| TS processing | Custom (Netty) | FFmpeg only | Fast failover without restart, analyzer |
| DB | PostgreSQL 16 | MySQL | JSONB, reliability |
| Cache/sessions | Redis 7 | Hazelcast | Simple, TTL keys, pub/sub |
| Edge | nginx | Varnish, custom Java server | `secure_link`, `auth_request`, efficient sendfile |
| UI | React + Ant Design | Vaadin, Thymeleaf | Rich admin UI, decoupled from backend |
| Monitoring | Micrometer + Prometheus + Grafana | Zabbix | Standard, ready dashboards |
| Deployment | Docker Compose + Ansible | Kubernetes | Enough for 3–6 servers, simpler operations |

## 10. Network ports

| Port | Service | Access |
|---|---|---|
| 443/80 | nginx edge (HLS, HTTP-TS, playlists) | Public |
| 443 | Admin UI + API (separate domain, e.g. `admin.`) | Restricted by IP / VPN |
| 9443 | gRPC core ↔ agent (mTLS) | Servers only |
| 8080 | tvr-core HTTP (internal, behind nginx) | localhost |
| 8090 | Agent HttpTsRelay | Edge only |
| 8081 | nginx origin (HLS) | Edge only |
| 9100/9464 | Prometheus exporters | Monitoring only |
| 20000–29999/udp | Loopback ports for TS between processes | 127.0.0.1 |
