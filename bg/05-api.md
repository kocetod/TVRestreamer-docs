# 05 · API спецификация

[← Индекс](00-README.md) · [English](../en/05-api.md) · Машинно четим договор: [openapi.yaml](../api/openapi.yaml)

## 1. Общи правила

| Тема | Правило |
|---|---|
| Базов URL | `https://admin.example.com/api/v1` |
| Формат | JSON (UTF-8), дати в ISO-8601 UTC, ID са UUID |
| Автентикация | `Authorization: Bearer <JWT>` (админ сесия) или `X-API-Key: tvr_xxx` (интеграции) |
| Странициране | `?page=0&size=50&sort=name,asc` → `{items, page, size, total}` |
| Филтри | Query параметри, напр. `?status=ONLINE&nodeId=...&q=btv` |
| Грешки | RFC 9457 Problem Details: `{type, title, status, detail, errors:[{field, message}]}` |
| Версиониране | В пътя (`/v1`). Несъвместими промени → `/v2` |
| Оптимистично заключване | `version` в тялото; при конфликт → `409 Conflict` |
| Rate limit | 20 заявки/сек на API ключ; `429` с `Retry-After` |
| Реално време | `GET /api/v1/events` (Server-Sent Events) |

## 2. Endpoints

### 2.1 Автентикация на администратори

| Метод | Път | Описание |
|---|---|---|
| POST | `/auth/login` | `{email, password, totp?}` → `{accessToken, refreshToken}` |
| POST | `/auth/refresh` | Нов access token |
| POST | `/auth/logout` | Обезсилва refresh token |
| GET | `/auth/me` | Текущ потребител и роля |
| POST | `/auth/2fa/setup` · `/auth/2fa/verify` | Настройка на TOTP |

### 2.2 Канали

| Метод | Път | Описание | Роля |
|---|---|---|---|
| GET | `/streams` | Списък (филтри: `status, categoryId, nodeId, packageId, q`) | Viewer+ |
| POST | `/streams` | Създаване (с входове и изходи) | Operator+ |
| GET | `/streams/{id}` | Детайл + текущ статус | Viewer+ |
| PUT | `/streams/{id}` | Пълна редакция (`version` задължително) | Operator+ |
| DELETE | `/streams/{id}` | Меко изтриване | Operator+ |
| POST | `/streams/{id}/actions/start` · `stop` · `restart` | Управление (TVR-021): `start`/`stop` сменят `enabled` (остава след рестарт), `restart` рестартира процесите на агента. 202 + каналът; 409 — `restart` на спрян канал, без сървър или с несвързан сървър; 404 — непознато действие | Operator+ |
| POST | `/streams/{id}/actions/switch-input` | `{inputId, pin: bool}` | Operator+ |
| POST | `/streams/{id}/clone` | Клониране | Operator+ |
| GET | `/streams/{id}/analyzer` | PID-ове, кодеци, грешки, битрейт (на живо) | Viewer+ |
| GET | `/streams/{id}/metrics?from&to&step` | Времеви редове (битрейт, CC, зрители) | Viewer+ |
| GET | `/streams/{id}/events` | История на събитията | Viewer+ |
| GET | `/streams/{id}/logs?tail=500` | Последни редове лог (+ SSE `/logs/stream`) | Operator+ |
| GET | `/streams/{id}/thumbnail.jpg` | Последна снимка | Viewer+ |
| POST | `/streams/bulk` | `{ids[], action: start\|stop\|move\|addToPackage, params}` | Operator+ |
| POST | `/inputs/probe` | `{type, url, options}` → резултат от ffprobe (FR-INP-07) | Operator+ |

**Пример — създаване на канал:**

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

> Отговорът съдържа целия канал (входове и изходи с `id`). `version` започва от 0 и расте с 1 при всяка промяна. `status` е състоянието, докладвано от агента (TVR-020): `STARTING` → `ONLINE` (с `bitrateIn` в бита/сек, `activeInputId`, `since`) или `OFFLINE`/`ERROR` с `lastError`; `STOPPED`, когато каналът е изключен или няма доклад през последните 30 сек. `?status=` филтрира списъка по него (непозната стойност → 400). Грешки: 400 с `errors:[{field, message}]` (напр. `{"field": "inputs[0].url", "message": "Неподдържан протокол"}`), 409 при зает slug или стара `version`.

### 2.3 Категории и пакети

| Метод | Път | Описание |
|---|---|---|
| GET/POST | `/categories` | Списък (по `sortOrder`, после име) / създаване `{name, sortOrder?}` — без `sortOrder` отива последна |
| GET/PUT/DELETE | `/categories/{id}` | Детайл / редакция / изтриване (каналите остават без категория) |
| PUT | `/categories/order` | `[id, id, …]` — пълна подредба на всички категории; връща новия списък |
| GET/POST | `/packages` | Списък / създаване |
| GET/PUT/DELETE | `/packages/{id}` | Детайл / редакция / изтриване |
| PUT | `/packages/{id}/streams` | `[{streamId, sortOrder}]` — пълна подредба |

> Категории (TVR-013): четене — всяка роля, промяна — ADMIN/OPERATOR. Името е уникално без значение от главни/малки букви (409 с `errors[].field = name`), до 100 символа; `sortOrder` 0–100000. Отговорът съдържа `streamCount` (неизтрити канали). При `PUT /categories/order` всяка категория трябва да е точно веднъж — иначе 400.

### 2.4 Абонати

| Метод | Път | Описание | Роля |
|---|---|---|---|
| GET | `/subscribers` | Филтри: `status, packageId, expiresBefore, q` | Support+ |
| POST | `/subscribers` | Създаване | Support+ |
| GET/PUT | `/subscribers/{id}` | Детайл / редакция | Support+ |
| DELETE | `/subscribers/{id}` | Изтриване | Admin |
| POST | `/subscribers/{id}/actions/extend` | `{days}` или `{until}` | Support+ |
| POST | `/subscribers/{id}/actions/suspend` · `resume` · `ban` | Статус | Support+ |
| POST | `/subscribers/{id}/actions/regenerate-token` | Нов токен, кик на сесиите | Support+ |
| GET | `/subscribers/{id}/devices` · DELETE `/devices/{deviceId}` | Устройства | Support+ |
| GET | `/subscribers/{id}/sessions` | Активни сесии и история | Support+ |
| POST | `/subscribers/import` (CSV) · GET `/subscribers/export` | Масови операции | Admin |

### 2.5 Сесии

| Метод | Път | Описание |
|---|---|---|
| GET | `/sessions` | Активни сесии (филтри: `streamId, subscriberId, nodeId, country`) |
| DELETE | `/sessions/{sessionKey}` | Kick |
| DELETE | `/subscribers/{id}/sessions` | Kick на всички сесии на абонат |

### 2.6 Сървъри

| Метод | Път | Описание |
|---|---|---|
| GET | `/nodes` | Списък (по име): `status` (ONLINE/OFFLINE/DRAIN), `lastSeenAt`, `agentVersion`, `ffmpegVersion`, `connected` (има отворен gRPC поток); метрики — по-късно |
| POST | `/nodes/enrollment-tokens` | Ключ за регистрация на нов сървър (валиден 1 ч) |
| GET/PUT/DELETE | `/nodes/{id}` | Детайл / редакция / премахване |
| POST | `/nodes/{id}/actions/drain` · `undrain` | Режим поддръжка |

### 2.7 Мониторинг, известия, одит, настройки

| Метод | Път | Описание |
|---|---|---|
| GET | `/dashboard/summary` | Броячи за таблото |
| GET | `/events` | SSE: `stream.status`, `stream.failover`, `node.metrics`, `session.count`, `alert` |
| GET/POST/PUT/DELETE | `/alert-rules`, `/notification-channels` | Правила и канали за известия |
| POST | `/notification-channels/{id}/test` | Тестово известие |
| GET | `/alerts?state=FIRING` · POST `/alerts/{id}/ack` | Инциденти |
| GET | `/audit?actorId&entityType&from&to` | Одит лог |
| GET/PUT | `/settings` | Глобални настройки |
| POST | `/import/m3u` · `/import/astra` | Импорт (връща preview; `?commit=true` записва) |
| GET/POST | `/config/export` · `/config/import` | Пълен backup на конфигурацията |
| GET | `/reports/{type}?from&to` | `top-streams`, `peak-viewers`, `uptime`, `subscribers` |
| GET/POST/DELETE | `/admin-users`, `/api-keys` | Администратори и ключове |

## 3. Публични (абонатски) endpoints — на edge домейна

| Метод | Път | Описание |
|---|---|---|
| GET | `/pl/{token}.m3u?type=hls\|ts` | Персонален плейлист |
| GET | `/epg/{token}.xml.gz` | Филтриран XMLTV |
| GET | `/live/{slug}/index.m3u8?token=` | HLS (авторизация чрез `auth_request`) |
| GET | `/live/{slug}/{segment}.ts?s=&e=` | Сегмент (`secure_link`) |
| GET | `/ts/{slug}.ts?token=` | HTTP-TS |
| GET | `/get.php?username&password&type=m3u_plus` | Xtream съвместимост (R1) |
| GET | `/player_api.php?username&password&action=` | Xtream съвместимост (R1) |
| GET | `/live/{username}/{password}/{streamId}.{ts\|m3u8}` | Xtream съвместимост (R1) |

## 4. Вътрешни endpoints

| Метод | Път | Извиква | Описание |
|---|---|---|---|
| GET | `/internal/auth` | nginx `auth_request` | Входни headers: `X-Original-URI`, `X-Real-IP`, `User-Agent`. Отговор `200` + `X-Session-Id`, `X-Sign-Expires` / `401` / `403` + `X-Deny-Reason: EXPIRED\|LIMIT\|NOT_IN_PACKAGE\|IP\|GEO` |
| gRPC | `AgentControl.Connect(stream)` | tvr-agent | Двупосочен поток (виж [03-architecture](03-architecture.md) §7) |

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

Повторни опити: 1 мин, 5 мин, 30 мин, 2 ч, 6 ч. Очаква се отговор `2xx` до 5 сек.

## 6. gRPC договор (извадка)

```protobuf
syntax = "proto3";
package tvr.agent.v1;

service AgentControl {
  rpc Connect(stream AgentMessage) returns (stream CoreMessage);
}

message AgentMessage {
  oneof msg {
    Hello hello = 1;            // nodeId, версии, capabilities
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
    DesiredState desired = 1;   // пълен snapshot + version
    Command command = 2;        // RESTART, SWITCH_INPUT, KICK, SUBSCRIBE_LOGS
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

Пълният договор е в кода: `TVRestreamer_Code/tvr-common/src/main/proto/tvr/agent/v1/agent.proto` (TVR-008). Java пакет `io.tvr.agent.proto.v1`; класовете се генерират при `./gradlew :tvr-common:build`.

| Посока | Съобщение | Кога |
|---|---|---|
| агент → core | `Hello` | Първо съобщение след свързване: node_id, версии, capabilities, последна приложена версия, вече работещи потоци |
| агент → core | `Heartbeat`, `NodeMetrics` | Периодично (CPU, памет, мрежа, tmpfs) |
| агент → core | `StreamStatus`, `StreamEvent` | Състояние на канал; събития (FAILOVER, INPUT_LOST, PROCESS_EXITED…) |
| агент → core | `LogLines` | Само след `SubscribeLogs` |
| агент → core | `AckDesiredState` | Потвърждение на приложена версия (+ грешки по канали) |
| core → агент | `DesiredState` | Пълен snapshot на каналите за този node (`StreamSpec` → `InputSpec`, `OutputSpec`) |
| core → агент | `Command` | `RestartStream`, `SwitchInput`, `KickClient`, `SubscribeLogs` |
| core → агент | `Welcome` | Отговор на валиден `Hello` (TVR-017): `node_id`, интервал на heartbeat (5 сек), след колко сек без heartbeat node става `OFFLINE` (15) |

**Връзка и регистрация (TVR-017, опростено):** агентът отваря потока към `tvr.grpc.port` (9090) с метаданни `authorization: Bearer <token>`; token = `TVR_AGENT_TOKEN` на агента = `TVR_AGENT_ENROLL_TOKEN` на core (иначе `UNAUTHENTICATED`). Първото съобщение трябва да е `Hello` с `node_name` (иначе `FAILED_PRECONDITION` / `INVALID_ARGUMENT`); core намира node по `node_id`, после по име, или създава нов (`INGEST`) и го прави `ONLINE`. Heartbeat на всеки 5 сек обновява `last_seen_at`; без heartbeat 15 сек → `OFFLINE` (FR-NOD-02). Нова връзка на същия node затваря старата с `ABORTED`. Агентът се свързва отново с backoff 1 → 30 сек. Еднократните ключове и mTLS сертификатите (FR-NOD-01) идват по-късно.

**Desired state и reconciler (TVR-018, ADR-006):** веднага след `Welcome`, след всяка промяна на канал през API (след commit) и при периодична проверка на 30 сек (`tvr.grpc.desired-resync`) core изпраща `DesiredState` — всички неизтрити канали на node-а (каналите без `node_id` отиват на единствения `INGEST`/`ALL_IN_ONE` node, ако е само един). Версията расте само при промяна на съдържанието: `max(предишна + 1, текущо време в ms)`, затова расте и след рестарт на core; snapshot се праща по поток само ако версията е по-нова от последно изпратената. Агентът пуска каналите с `enabled=true`, `ALWAYS_ON` и включен вход, спира всичко останало, което сам е пуснал, и рестартира канал само ако се промени това, което изпълнява (slug, входове, HLS опции). Отговаря с `AckDesiredState` (`applied=false` + `errors` по канали при проблем); неуспелите канали се пробват отново на 30 сек. `Hello` и `Heartbeat` носят приложената версия и работещите канали. При загуба на връзката каналите продължават; последният snapshot се пази на диска на агента и се възстановява при рестарт.

**Статус на канала (TVR-020, FR-STR-07):** агентът праща `StreamStatus` за всеки канал от snapshot-а си при всяка промяна и за всички на всеки 5 сек (и веднага след повторно свързване). До TS анализатора (S2) състоянието идва от HLS изхода: `ONLINE`, докато плейлистът се обновява (в рамките на 2 × сегмент + 2 сек; `bitrate_in` = размерът на последните 3 сегмента / продължителността им), `STARTING` по време на стартовия толеранс (timeout на входа + 2 × сегмент + 5 сек), после `OFFLINE` с `last_error` (колко време няма данни, рестарти на входа, exit code — никога изход на FFmpeg, може да съдържа URL); `ERROR` за канал, който не е успял да тръгне; `STOPPED` еднократно, когато каналът спре да работи там. Core приема доклади само за канали от snapshot-а на node-а и пази последния в Redis `stream:status:<id>` (JSON, TTL 30 сек); `since` се сменя само при смяна на състоянието.

**Съвместимост:** номерата и типовете на полетата не се сменят; изтрити полета стават `reserved`; нулевата стойност на всеки enum е `*_UNSPECIFIED`; несъвместима промяна = пакет `tvr.agent.v2`.
