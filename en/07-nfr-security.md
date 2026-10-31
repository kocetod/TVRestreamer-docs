# 07 · Non-Functional Requirements and Security

[← Index](00-README.md) · [Български](../bg/07-nfr-security.md)

## 1. Non-functional requirements

| ID | Category | Requirement | Target | How measured |
|---|---|---|---|---|
| NFR-01 | Scale | Subscribers in the database | 10,000 (×4 headroom) | Load test |
| NFR-02 | Scale | Concurrent viewers | 2,000 (HLS + HTTP-TS) | Load test (k6 + hls-bench) |
| NFR-03 | Scale | Streams in the system / per ingest server | 500 / 250 | Load test |
| NFR-04 | Performance | Stream start (always-on) to ONLINE | ≤ 5 s p95 | Metric `stream_start_seconds` |
| NFR-05 | Performance | Hot-standby failover | ≤ 3 s p95 | Chaos test |
| NFR-06 | Performance | Cold failover | ≤ 8 s p95 | Chaos test |
| NFR-07 | Performance | `/internal/auth` | p99 ≤ 20 ms, ≥ 2,000 RPS | Load test |
| NFR-08 | Performance | Admin API | p95 ≤ 200 ms | Micrometer |
| NFR-09 | Performance | M3U generation (300 streams) | ≤ 100 ms | Load test |
| NFR-10 | Performance | Status delay in UI | ≤ 2 s | E2E test |
| NFR-11 | Performance | Live latency (HLS) | 10–20 s (4 s × 3 segments + buffer) | Manual |
| NFR-12 | Availability | Platform (delivery) | 99.9% monthly | Uptime monitoring |
| NFR-13 | Availability | Core outage does not stop viewing | Active sessions continue ≥ 1 h | Chaos test |
| NFR-14 | Availability | Loss of 1 edge | Players reconnect to another edge ≤ 30 s | Chaos test |
| NFR-15 | Resources | CPU per stream (copy, no transcode) | ≤ 3% of 1 core | Profiling |
| NFR-16 | Resources | Agent RAM | ≤ 1 GB at 250 streams (excluding tmpfs) | Profiling |
| NFR-17 | Maintainability | Deploying a new core version | No viewing interruption | Release procedure |
| NFR-18 | Maintainability | Test coverage | ≥ 70% of domain logic, ≥ 90% of TS parser | JaCoCo |
| NFR-19 | Usability | UI in Bulgarian and English, desktop ≥ 1280 px, tablet | i18n | Review |
| NFR-20 | Observability | All components export Prometheus metrics and JSON logs | — | Review |
| NFR-21 | Compliance | GDPR: minimal personal data, subscriber data export and deletion, IP logs ≤ 90 days | — | Review |

## 2. Capacity planning (2,500 subscribers)

**Assumptions:** 60% peak concurrency → **1,500 viewers**; 4 Mbps average bitrate (SD/HD/FHD mix); 300 streams; 30% of streams with hot standby.

| Resource | Calculation | Result |
|---|---|---|
| Egress (peak) | 1,500 × 4 Mbps | **6 Gbps** (+30% headroom → 8 Gbps) |
| Ingress (ingest) | 300 × 5 Mbps × 1.3 | **≈ 2 Gbps** |
| Ingest CPU | 300 × (1 input + 1 packager) × ~1.5% core + 90 standby × 1% | **≈ 10 cores** |
| tmpfs for HLS | 300 × 8 seg × 4 s × 5 Mbps / 8 | **≈ 6 GB** |
| Redis | 2,000 sessions × ~1 KB | < 50 MB |
| PostgreSQL | 90 days of sessions ≈ 5M rows | ≈ 5 GB |

### Recommended hardware (launch)

| Role | Count | Specification |
|---|---|---|
| Control plane (core, PG, Redis, Grafana) | 1 (+1 standby in R2) | 4–8 vCPU, 16 GB RAM, 200 GB NVMe |
| Ingest | 2 (active + backup, or 2 × 150 streams) | 16 cores, 64 GB RAM, 2 × 10 Gbps, Ubuntu 24.04 |
| Edge | 2–3 (N+1) | 8 cores, 32 GB RAM, 1–2 × 10 Gbps (or 25 Gbps) |

> For launch, an **ALL_IN_ONE** setup on 1–2 powerful servers (32 cores, 128 GB, 2 × 10G) is also possible. Roles can be split later without code changes.

## 3. Security

### 3.1 Threat model (STRIDE, summary)

| Threat | Example | Mitigation |
|---|---|---|
| **S**poofing | Stolen admin password | 2FA for Admin, lockout after 5 attempts, admin panel only from allowed IPs/VPN |
| **S**poofing | Sharing a subscriber link | Session limit, short-lived signed segment URLs, token regeneration, IP binding (opt.) |
| **T**ampering | Forged segment URL | HMAC-SHA256 signature with expiry (njs on the edge, TVR-044) |
| **R**epudiation | "I didn't delete it" | Append-only audit log with diff |
| **I**nformation disclosure | Leak of provider URLs | Source URLs never shown to subscribers; secrets encrypted in DB; Support role cannot see inputs |
| **D**enial of service | Flooding `/internal/auth` or playlists | nginx rate limiting (`limit_req`), 10 s auth cache, fail2ban on 403 |
| **E**levation of privilege | Support making Admin changes | RBAC at API level (Spring Security `@PreAuthorize`), tests for every role |

### 3.2 Security requirements

| ID | Requirement |
|---|---|
| SEC-01 | TLS 1.2+ everywhere public (Let's Encrypt/ACME); HSTS on the admin domain |
| SEC-02 | mTLS between core and agents; an agent's certificate is revoked when the node is removed |
| SEC-03 | Passwords with Argon2id; API keys stored only as hashes (shown once) |
| SEC-04 | JWT access 15 min, refresh 7 days with rotation; refresh in an httpOnly, Secure, SameSite=Strict cookie |
| SEC-05 | Secrets (source passwords, SRT passphrase, signing key) — AES-256-GCM, key from env/secret file, never in git |
| SEC-06 | Validate all input (Bean Validation). FFmpeg commands are built as an **argument list**, never through a shell, to prevent command injection via URLs |
| SEC-07 | Restrict allowed input URL schemes (`http, https, udp, rtp, srt, rtmp, rtsp`). `file:`, `concat:`, `subfile:` etc. are forbidden (prevents reading local files through FFmpeg) |
| SEC-08 | Security headers in the UI (CSP, X-Frame-Options, Referrer-Policy) |
| SEC-09 | Dependencies: OWASP Dependency-Check / Renovate in CI; Trivy image scanning |
| SEC-10 | The agent runs as unprivileged user `tvr`; FFmpeg sandboxed (systemd `ProtectSystem`, `NoNewPrivileges`) |
| SEC-11 | Backups are encrypted and stored off-server |
| SEC-12 | Logs contain no passwords, tokens (masked `tok_****abcd`) or full URLs with credentials |
