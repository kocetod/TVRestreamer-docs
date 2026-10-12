# 03 · Архитектура

[← Индекс](00-README.md) · [English](../en/03-architecture.md)

## 1. Принципи

1. **Control plane ≠ data plane.** `tvr-core` управлява конфигурация, абонати и авторизация. `tvr-agent` и FFmpeg пренасят видеото. Ако core падне, работещите канали и вече авторизираните сесии продължават.
2. **Видеото не минава през бизнес логиката.** Байтовете се движат от FFmpeg → TS превключвател (Netty, zero-copy буфери) → FFmpeg/nginx.
3. **Desired state + reconciliation.** Core изпраща на агента пълното желано състояние. Агентът сам привежда процесите в съответствие (подобно на Kubernetes), затова изгубени команди и рестарти не водят до разминаване.
4. **Един процес FFmpeg на вход и на пакетиране, никога на зрител.**
5. **Stateless edge.** Edge сървърите са само nginx с кеш и могат да се добавят хоризонтално.

## 2. C4 · Ниво 1 — Контекст

```mermaid
flowchart TB
    subgraph Хора
      ADM[Администратор / Техник / Поддръжка]
      SUB[Абонат<br/>STB · Smart TV · мобилен плейър]
    end
    subgraph Външни системи
      PROV[Доставчици на потоци<br/>HTTP/HLS · SRT · RTMP]
      HEAD[Локален headend<br/>UDP multicast · бъдещо DVB]
      EPG[EPG доставчик<br/>XMLTV]
      TG[Telegram / SMTP]
      BILL[Билинг / CRM]
    end
    TVR[[TvRestream платформа]]
    ADM -- уеб панел --> TVR
    SUB -- M3U, HLS, HTTP-TS --> TVR
    PROV -- потоци --> TVR
    HEAD -- потоци --> TVR
    EPG -- XMLTV --> TVR
    TVR -- известия --> TG
    BILL -- REST API / webhooks --> TVR
```

## 3. C4 · Ниво 2 — Контейнери

```mermaid
flowchart LR
    subgraph CP[Control plane — 1 сървър или VM]
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
      FF[FFmpeg процеси]
      SHM[(tmpfs /dev/shm/hls)]
      OR[nginx origin]
      AG -- стартира/следи --> FF
      FF -- UDP loopback --> AG
      AG -- TS --> FF
      FF -- HLS сегменти --> SHM
      OR -- чете --> SHM
    end
    subgraph ED[Edge node × M]
      NG[nginx edge<br/>secure_link · auth_request · proxy_cache]
    end
    CORE <-- gRPC двупосочен поток<br/>(инициира агентът) --> AG
    NG -- auth_request --> CORE
    NG -- proxy_pass/cache --> OR
    NG -- HTTP-TS proxy --> AG
    PLAYER[Плейъри на абонати] --> NG
    SRC[Източници] --> FF
    PROM -. scrape .-> AG
```

| Контейнер | Технология | Отговорност |
|---|---|---|
| **Admin UI** | React 18, TypeScript, Vite, Ant Design, TanStack Query, hls.js | Уеб панел |
| **tvr-core** | Java 21, Spring Boot 4.1, Spring Security, Spring Data JPA, Flyway, gRPC, Micrometer | REST API, авторизация на абонати, плейлисти, оркестрация на агентите, известия, статистика |
| **PostgreSQL** | 16 | Конфигурация, абонати, одит, история |
| **Redis** | 7 | Активни сесии, кеш на авторизацията, rate limit, pub/sub на статуси |
| **tvr-agent** | Java 21, Spring Boot (без web MVC), Netty, gRPC клиент | Супервизор на FFmpeg, TS превключвател и анализатор, HTTP-TS relay, метрики на node |
| **FFmpeg** | 7.x (+libsrt, openssl) | Приемане на входове, пакетиране в HLS, UDP/SRT/RTMP изходи |
| **nginx origin** | 1.26 | Сервира HLS от tmpfs към edge |
| **nginx edge** | 1.26 | TLS, авторизация, кеш, раздаване към зрителите |
| **Prometheus/Grafana** | — | Метрики и табла |

## 4. C4 · Ниво 3 — Компоненти на tvr-agent

```mermaid
flowchart LR
    subgraph AGENT[tvr-agent]
      SYNC[ControlChannel<br/>gRPC клиент · desired state]
      REC[Reconciler]
      SUP[ProcessSupervisor<br/>старт/стоп/рестарт, backoff]
      subgraph PIPE[StreamPipeline × канал]
        IW1[InputWorker #1<br/>FFmpeg → udp://127.0.0.1:p1]
        IW2[InputWorker #2 hot standby<br/>FFmpeg → udp://127.0.0.1:p2]
        SW[TsSwitcher<br/>Netty UDP · избира активен вход]
        AN[TsAnalyzer<br/>PAT/PMT/CC/PCR/битрейт]
        FO[FailoverController]
        PK[Packager<br/>FFmpeg → HLS в tmpfs]
        HT[HttpTsRelay<br/>fan-out към клиенти]
        UO[UDP/SRT изходи<br/>FFmpeg]
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
    AN -- статистики --> SYNC
    FO -- събития --> SYNC
    MET --> SYNC
```

### 4.1 Защо собствен TS превключвател (TsSwitcher)

Ако FFmpeg приема входа директно и прави HLS, при failover трябва да се рестартира целият процес. Това води до 5–10 сек прекъсване и „счупен“ плейлист. С TsSwitcher:

- входовете се приемат от отделни FFmpeg процеси (`-c copy -f mpegts udp://127.0.0.1:PORT`);
- TsSwitcher чете всички активни входове, но препраща само избрания към packager-а;
- при превключване изпраща чист TS (с повторно изпратени PAT/PMT), а packager-ът продължава без рестарт. При нужда се вмъква `EXT-X-DISCONTINUITY`;
- анализаторът работи върху същите пакети без допълнително копиране.

## 5. Потоци на данните за един канал

```mermaid
flowchart LR
    S1[(Основен източник)] --> I1[FFmpeg input #1]
    S2[(Резервен източник)] --> I2[FFmpeg input #2]
    I1 -- MPEG-TS/UDP lo --> SW{TsSwitcher}
    I2 -- MPEG-TS/UDP lo --> SW
    SW --> AN[[TsAnalyzer]]
    SW -- TS --> P[FFmpeg packager<br/>-c copy -f hls]
    P --> H[(/dev/shm/hls/slug/<br/>index.m3u8 + seg_N.ts)]
    SW -- TS --> R[HttpTsRelay :8090/ts/slug]
    SW -- TS --> U[FFmpeg UDP out<br/>239.1.1.x:1234]
    H --> O[nginx origin] --> E[nginx edge] --> V((Зрители HLS))
    R --> E2[nginx edge proxy] --> V2((Зрители HTTP-TS))
```

## 6. Ключови последователности

### 6.1 Създаване и стартиране на канал

```mermaid
sequenceDiagram
    autonumber
    actor T as Техник
    participant UI as Admin UI
    participant C as tvr-core
    participant DB as PostgreSQL
    participant A as tvr-agent
    participant F as FFmpeg
    T->>UI: Нов канал (име, входове, изходи)
    UI->>C: POST /api/v1/streams
    C->>DB: INSERT stream, inputs, outputs (version=1)
    C->>A: DesiredState(stream, v1) по gRPC
    A->>A: Reconciler: нов pipeline
    A->>F: старт input #1 (+ hot standby #2)
    A->>F: старт packager HLS
    F-->>A: първи TS пакети
    A->>A: TsAnalyzer: PAT/PMT OK, видео+аудио OK
    A-->>C: StreamStatus(ONLINE, input=1, bitrate)
    C-->>UI: SSE stream.status = ONLINE
```

### 6.2 Автоматичен failover

```mermaid
sequenceDiagram
    autonumber
    participant S1 as Основен източник
    participant A as tvr-agent (Analyzer + Failover)
    participant SW as TsSwitcher
    participant C as tvr-core
    participant N as Telegram
    S1--xA: спира да подава данни
    A->>A: 5 сек без пакети на input #1 (timeout)
    A->>SW: switch(input #2) — hot standby, < 100 ms
    SW->>SW: повторно PAT/PMT, discontinuity флаг
    A-->>C: Event FAILOVER(1→2, reason=NO_DATA)
    C->>N: „BTV: превключен на резервен (няма данни)“
    loop на всеки 10 сек
      A->>S1: проба на input #1
    end
    A->>A: input #1 стабилен 30 сек
    A->>SW: switch(input #1) — return to primary
    A-->>C: Event RECOVERED(2→1)
```

### 6.3 On-demand канал

```mermaid
sequenceDiagram
    autonumber
    participant P as Плейър
    participant E as Edge
    participant C as tvr-core
    participant A as tvr-agent
    P->>E: GET /live/slug/index.m3u8?token
    E->>C: auth_request
    C->>C: каналът е STOPPED (on-demand) → заявка за старт
    C->>A: StartStream(slug)
    C-->>E: 200
    E->>E: index.m3u8 липсва → retry до 6 сек (nginx try_files + upstream към агента)
    A-->>E: първи сегменти готови
    E-->>P: m3u8
    Note over A: 60 сек без зрители → стоп
```

## 7. Комуникация core ↔ agent

| Аспект | Решение |
|---|---|
| Транспорт | gRPC двупосочен поток, **инициира агентът** (работи зад NAT/firewall), mTLS |
| Регистрация | Enrollment token → core издава клиентски сертификат за агента |
| Съобщения core→agent | `DesiredState` (пълен snapshot + версия), `Command` (restart, switch input, kick HTTP-TS клиент) |
| Съобщения agent→core | `Heartbeat` (5 сек), `NodeMetrics` (5 сек), `StreamStatus` (при промяна + на 5 сек), `StreamEvent`, `LogLines` (при абонамент) |
| При загуба на връзка | Агентът продължава с последното известно състояние и се свързва отново с exponential backoff |

## 8. Структура на кода (monorepo)

```
tvrestream/
├── settings.gradle, build.gradle # Gradle multi-project (Java 21)
├── tvr-common/                  # DTO, proto (gRPC), общи утилити
│   └── src/main/proto/agent.proto
├── tvr-core/                    # Spring Boot приложение
│   └── src/main/java/io/tvr/core/
│       ├── stream/              # канали, входове, изходи
│       ├── subscriber/          # абонати, пакети, устройства
│       ├── auth/                # admin auth (JWT) + playback auth
│       ├── playlist/            # M3U, Xtream API, EPG
│       ├── session/             # сесии (Redis)
│       ├── node/                # регистрация и оркестрация на агенти
│       ├── monitoring/          # статуси, SSE, известия
│       ├── audit/               # одит
│       └── config/              # настройки, импорт/експорт
├── tvr-agent/                   # Spring Boot (non-web) + Netty
│   └── src/main/java/io/tvr/agent/
│       ├── control/             # gRPC клиент, reconciler
│       ├── process/             # ProcessSupervisor, FfmpegCommandBuilder
│       ├── ts/                  # TsSwitcher, TsAnalyzer, TsPacket парсер
│       ├── relay/               # HttpTsRelay (Netty HTTP)
│       └── metrics/
├── tvr-ui/                      # React + TypeScript
├── deploy/
│   ├── docker-compose.yml
│   ├── nginx/{origin,edge}.conf
│   └── ansible/                 # инсталация на node
└── docs/                        # тази документация
```

## 9. Технологичен стек и версии

| Слой | Избор | Алтернатива (отхвърлена) | Причина |
|---|---|---|---|
| Език/рамка | Java 21 + Spring Boot 4.1 | Python/FastAPI, Go | Избор на собственика; зрял екосистем; virtual threads |
| Медиа I/O | FFmpeg 7 | GStreamer | По-разпространен, по-лесен за дебъг, всички протоколи |
| TS обработка | Собствена (Netty) | Само FFmpeg | Бърз failover без рестарт, анализатор |
| БД | PostgreSQL 16 | MySQL | JSONB, надеждност |
| Кеш/сесии | Redis 7 | Hazelcast | Прост, TTL ключове, pub/sub |
| Edge | nginx | Varnish, собствен Java сървър | `secure_link`, `auth_request`, ефективен sendfile |
| UI | React + Ant Design | Vaadin, Thymeleaf | Богат админ UI, отделен от бекенда |
| Мониторинг | Micrometer + Prometheus + Grafana | Zabbix | Стандарт, готови табла |
| Деплой | Docker Compose + Ansible | Kubernetes | Достатъчно за 3–6 сървъра, по-лесна поддръжка |

## 10. Мрежови портове

| Порт | Услуга | Достъп |
|---|---|---|
| 443/80 | nginx edge (HLS, HTTP-TS, плейлисти) | Публичен |
| 443 | Admin UI + API (отделен домейн, напр. `admin.`) | Ограничен по IP / VPN |
| 9443 | gRPC core ↔ agent (mTLS) | Само сървърите |
| 8080 | tvr-core HTTP (вътрешен, зад nginx) | localhost |
| 8090 | HttpTsRelay на агента | Само edge |
| 8081 | nginx origin (HLS) | Само edge |
| 9100/9464 | Prometheus exporters | Само мониторинг |
| 20000–29999/udp | Loopback портове за TS между процесите | 127.0.0.1 |
