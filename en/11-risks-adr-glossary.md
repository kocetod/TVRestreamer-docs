# 11 · Risks, Architecture Decision Records (ADR) and Glossary

[← Index](00-README.md) · [Български](../bg/11-risks-adr-glossary.md)

## 1. Risk register

| ID | Risk | Prob. | Impact | Mitigation | Owner |
|---|---|:-:|:-:|---|---|
| R-01 | No retransmission rights → legal claims | M | H | Agreements with providers/rights holders before launch; the platform has no features for circumventing protection | Owner |
| R-02 | Unstable provider sources | H | H | Hot standby, ≥ 2 providers for key streams, alerts | Technician |
| R-03 | TsSwitcher is harder than expected (PCR/timestamps on switch) | M | M | Spike in S2; fallback: cold failover with packager restart (already in MVP) | Dev |
| R-04 | Some players don't handle HLS discontinuities | M | M | `EXT-X-DISCONTINUITY` + tests with TiviMate, VLC, Smart TV (Samsung/LG), MAG | QA |
| R-05 | Egress traffic exceeds capacity | M | H | NIC monitoring, fast edge addition (Ansible), burst agreement with the DC | DevOps |
| R-06 | Account sharing despite the limit | H | M | Session limit, short-lived signatures, IP binding (R2), suspicious-subscriber report | Owner |
| R-07 | Control plane outage | L | H | Edge caches auth; grace mode; backups; R2 — HA (Patroni + 2 × core) | DevOps |
| R-08 | DDoS against edge | M | H | Rate limiting, fail2ban, DC DDoS protection, ingest hidden behind edge | DevOps |
| R-09 | Dependence on one developer | H | M | Documentation, ADRs, tests, clean code, runbooks | Owner |
| R-10 | FFmpeg changes between versions | L | M | Pinned version in the image, integration tests before upgrading | Dev |

## 2. Architecture decisions (ADR)

### ADR-001 · FFmpeg for media I/O, Java for logic
- **Context:** Many protocols (HTTP, HLS, UDP, SRT, RTMP, RTSP) and reliable HLS packaging are required.
- **Decision:** FFmpeg processes (`-c copy`) managed by a Java agent. Video does not pass through the JVM, except the TS switcher (see ADR-003).
- **Consequences:** + mature, fast I/O; + easy debugging (the command can be run by hand); − 1–2 processes per stream, needs solid supervision.

### ADR-002 · Java 21 + Spring Boot 3 for core and agent
- **Context:** The owner's choice. REST, security, ORM, gRPC and metrics are needed.
- **Decision:** Spring Boot 3.3 (superseded by 4.1 — see ADR-010), virtual threads for blocking I/O, Netty for the TS path.
- **Consequences:** + one backend language, rich ecosystem; − higher RAM than Go (acceptable).

### ADR-003 · Custom TS switcher and analyzer (Netty)
- **Context:** Restarting FFmpeg on failover causes a 5–10 s interruption and "breaks" HLS.
- **Decision:** Inputs send TS over UDP loopback to the agent. The agent selects the active input, analyzes packets (PAT/PMT/CC/PCR) and forwards to the packager.
- **Consequences:** + failover < 3 s; + Astra-like analyzer for "free"; − custom MPEG-TS code (covered by property-based tests); − extra copy through loopback (negligible: ~5 Mbps per stream).

### ADR-004 · HLS as the primary output, served by nginx
- **Context:** 2,000 viewers, ~6–8 Gbps.
- **Decision:** FFmpeg writes HLS into tmpfs; nginx origin + nginx edge with `proxy_cache_lock`. HTTP-TS is served by a Netty relay in the agent, proxied by the edge.
- **Consequences:** + nginx is the most efficient file server (sendfile); + edge scales horizontally; − HLS adds 10–20 s latency (acceptable for IPTV).

### ADR-005 · Authorization: `auth_request` for playlists, `secure_link` for segments
- **Context:** ~375 playlist requests/s and ~1,500 segment requests/s at peak; core must not be in the hot path.
- **Decision:** Only playlist requests go through core (10 s cache on the edge). Segments are verified locally with HMAC.
- **Consequences:** + core handles < 400 RPS; + edge keeps working during short core outages; − access revocation is delayed by ≤ 10 s (cache) + 60 s (signature lifetime).
- **Implementation (TVR-044):** the signature is HMAC-SHA256 in njs (created in the playlist, checked for every segment) instead of `secure_link_md5` — the built-in module supports MD5 only. The principle (local check without core) is the same.

### ADR-006 · Desired state + reconciliation between core and agent
- **Decision:** Core sends a full snapshot with a version. The agent is idempotent.
- **Consequences:** + resilient to lost messages and restarts; − larger payload (negligible below 500 streams).

### ADR-007 · Docker Compose + Ansible instead of Kubernetes
- **Decision:** Few servers (3–6), and host networking plus multicast are required.
- **Consequences:** + simple operations; − manual scaling (acceptable).

### ADR-008 · No transcoding until R3
- **Decision:** `-c copy` only. Transcoding will be added as a separate "processing profile" type with GPU.
- **Consequences:** + cheap hardware, low latency; − streams are delivered at the original bitrate and codec.

### ADR-009 · Gradle instead of Maven
- **Context:** The owner already has a Gradle project in IntelliJ.
- **Decision:** Gradle multi-project (tvr-common, tvr-core, tvr-agent) with a version catalog.
- **Consequences:** + no tool change; + faster incremental builds; − slightly different conventions from Maven (minor).

### ADR-010 · Spring Boot 4.1 instead of 3.3
- **Context:** Spring Boot 3.3 and 3.5 are out of free (OSS) support (3.5 since 30 June 2026). The current line is 4.1 (4.1.1, August 2026).
- **Decision:** tvr-core and tvr-agent use Spring Boot 4.1.x (Spring Framework 7, Jakarta EE 11, JUnit 6, Testcontainers 2). The version lives in one place: `gradle/libs.versions.toml`.
- **Consequences:** + support and security fixes; + built-in structured (JSON) logging; − some starters are renamed (e.g. `spring-boot-starter-webmvc`) and Jackson 3 — Boot 3 examples online do not always apply.

## 3. Glossary

| Term | Meaning |
|---|---|
| **Restream** | Retransmitting a stream from a source to many viewers without changing the content |
| **Ingest** | Receiving the incoming stream |
| **Edge** | Server that delivers content to end viewers |
| **Origin** | Server the edge fetches content from |
| **MPEG-TS (TS)** | Transport Stream — the container used in television (188-byte packets) |
| **SPTS / MPTS** | Single/Multi Program Transport Stream — one or many channels in one stream |
| **PID** | Packet Identifier — identifier of an elementary stream in TS |
| **PAT / PMT / SDT** | TS tables: program list, PIDs per program, service names |
| **PNR** | Program Number (service ID) |
| **CC error** | Continuity Counter error — a lost or duplicated TS packet |
| **PCR** | Program Clock Reference — the stream clock |
| **Scrambled** | Encrypted (coded) stream |
| **HLS** | HTTP Live Streaming — `.m3u8` playlist + `.ts`/`.m4s` segments |
| **HTTP-TS** | Continuous MPEG-TS over an HTTP connection |
| **SRT** | Secure Reliable Transport — protocol for reliable transport over the internet |
| **Multicast / IGMP** | One-to-many delivery on a local network / subscription protocol |
| **Failover** | Automatic switch to a backup source |
| **Hot standby** | The backup source is received continuously, ready for instant switching |
| **LCN** | Logical Channel Number — channel number in the list |
| **EPG / XMLTV** | Electronic Program Guide / its file format |
| **M3U** | Playlist format for IPTV players |
| **Xtream Codes API** | De-facto standard for IPTV panels, supported by most players |
| **Slate** | Fallback picture when there is no signal |
| **Middleware** | System for managing subscribers and access |
| **Desired state** | A description of the target state that the system converges to on its own |
