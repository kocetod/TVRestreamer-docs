# 01 · Vision and Product Requirements Document (PRD)

[← Index](00-README.md) · [Български](../bg/01-vision-prd.md)

## 1. Problem

Operators who restream IPTV channels to their own subscribers usually:

- use **Astra/Flussonic** for streams and a **separate middleware** (Xtream UI, Ministra, etc.) for subscribers. That means two systems, two licences and an integration between them;
- have no automatic switch to a backup source, so a channel goes down and stays down until someone notices;
- have no visibility into who is watching what or how many concurrent connections exist, so one account gets shared by many people;
- pay per-server or per-channel licences, which become a significant cost at 200–300 channels.

## 2. Vision

> **One system where you add a channel in 30 seconds, it keeps the channel alive on its own, and the subscriber gets one protected link that just works.**

## 3. Goals and success metrics

| # | Goal | Metric (KPI) | Target |
|---|---|---|---|
| G1 | Stable channels | Monthly channel uptime | ≥ 99.5% per channel, ≥ 99.9% platform |
| G2 | Fast recovery | Time to switch to backup source | < 3 s (hot standby), < 8 s (cold) |
| G3 | Fast start | Time from click to picture (HLS) | < 3 s for a running channel |
| G4 | Anti-sharing | Subscribers above device limit | 100% of violations blocked |
| G5 | Easy operation | Time to add a channel | < 1 min, no SSH |
| G6 | Visibility | Time to alert on a down channel | < 30 s |
| G7 | Scale | Concurrent viewers | 2,500 subscribers / up to 2,000 concurrent streams |

## 4. Scope

### 4.1 In scope (Astra-like features needed for restreaming)

- Channels (streams) with **multiple inputs** by priority, plus automatic failover and return to the primary source.
- Inputs: **HTTP(S)-TS, HLS, UDP/RTP (unicast/multicast), SRT, RTMP, RTSP**. The architecture is ready for **DVB-S/S2** (R3).
- Processing without transcoding: program selection from MPTS, audio/subtitle/PID selection and filtering, service renaming.
- Outputs: **HLS**, **HTTP-TS**, **UDP/RTP multicast**, **SRT** (listener/caller), **RTMP push** (R2).
- **Stream analyzer**: bitrate, CC errors, PCR, scrambled, video and audio presence, resolution, codecs.
- **Monitoring and alerts**: Telegram, email, webhook.
- **Subscribers, packages, devices, sessions**: built-in middleware.
- **Personal playlists** (M3U/M3U8) with logos and EPG URL (XMLTV passthrough).
- **Multiple servers (nodes)**: central management, channel placement, edge servers.
- **Admin users with roles**, 2FA and audit log.
- **REST API** for everything, plus webhooks.
- Channel import from M3U and Astra configuration (JSON).

### 4.2 Out of scope

| Feature | Reason | When |
|---|---|---|
| DVR / Catch-up / Timeshift | Explicitly excluded | — |
| Transcoding | Not needed for restreaming, high hardware cost | R3 (GPU NVENC) |
| DVB-S/S2/T/C reception | Needs hardware, complexity | R3 |
| CAS / descrambling (BISS, CAM) | Legal and technical risks | R3, to be decided |
| Payments and billing | Separate system, integration via API | R3 |
| Own player/app | Subscribers use existing players | R3 |
| VOD / movies | Not restreaming | W |
| Reseller panel | Not requested | R3 |

## 5. Personas

| Persona | Description | Main needs |
|---|---|---|
| **Operator / Owner** (Admin) | Runs the business and infrastructure | Full control, statistics, stability, low cost |
| **Technician** (Operator) | Keeps channels running 24/7 | Fast channel add/edit, alerts, analyzer, logs |
| **Support** | Serves subscribers by phone | Subscriber search, sessions, device reset, subscription extension |
| **Subscriber** | Watches TV on STB, Smart TV or phone | One link that works, fast zapping, EPG |
| **Integrator** (API client) | Billing or CRM system | API for subscribers and packages, webhooks |

## 6. Key user journeys

1. **Add a channel:** The technician enters a name and source URL, adds a backup URL, picks a package and saves. The channel starts within 5 s and appears in the playlists of subscribers with that package.
2. **New subscriber:** Support creates a subscriber, selects a package, a term and a limit of 2 devices, then sends the M3U link.
3. **Source down:** The primary source stops. Within 3 s the agent switches to the backup, the technician receives a Telegram alert, and subscribers see no interruption (or a pause under 3 s).
4. **Shared account:** A subscriber with a limit of 2 opens a 3rd device. The request is refused (HTTP 403 / "device limit" video), and the violation is written to the audit log.

## 7. Assumptions and constraints

- The operator **holds the rights** to retransmit the channels. The platform does not resolve licensing questions.
- Sources already use codecs that end devices support (H.264/H.265, AAC/MP2/AC3).
- Servers run Linux (Ubuntu 24.04 LTS) with at least a 10 Gbps uplink for edge.
- Team: 1 developer + Claude, plus a test environment and real streams provided by the owner.

## 8. Dependencies

- FFmpeg 7.x (with libsrt, openssl), nginx 1.26+ (`secure_link`, `auth_request`), PostgreSQL 16, Redis 7.
- Telegram Bot API and SMTP for alerts.

## 9. Open questions

| # | Question | Owner | Due |
|---|---|---|---|
| Q1 | Number of channels at launch and after 1 year? (estimate 150–300) | Owner | Sprint 0 |
| Q2 | Is HTTP-TS needed, or do all devices support HLS? | Owner | Sprint 0 |
| Q3 | Will there be multicast output to an own network (cable/LAN)? | Owner | Sprint 0 |
| Q4 | Integration with existing billing? Which one and what API? | Owner | R1 |
| Q5 | Geo/IP restrictions (Bulgaria only)? | Owner | R1 |
| Q6 | Playlist branding and channel logos: where do they come from? | Owner | MVP |
