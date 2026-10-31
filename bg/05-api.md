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
| POST | `/streams/{id}/actions/switch-input` | `{inputId, pin: bool}` — pin спира автоматиката; активният вход с pin=false = откачане; статусът: `status.inputPinned` (TVR-036) | Operator+ |
| POST | `/streams/{id}/clone` | Клониране | Operator+ |
| GET | `/streams/{id}/analyzer` | Битрейт, PID-ове с кодеци и езици, CC грешки (на живо, на 5 сек; TVR-026) | Viewer+ |
| GET | `/streams/{id}/metrics?from&to&step` | Времеви редове (битрейт, CC, зрители) | Viewer+ |
| GET | `/streams/{id}/events` | История на събитията, най-новите първо: `FAILOVER`, `STATUS`, … Филтри `type` (няколко), `from`, `to` (ISO-8601), `page`, `size` | Viewer+ |
| GET | `/streams/{id}/logs?tail=500` | Последни редове лог от агента (1–1000, TVR-023; SSE `/logs/stream` — по-късно) | Operator+ |
| PUT | `/streams/{id}/logo` | Качване на лого (multipart `file`, PNG/SVG ≤ 512 KB, TVR-047) → `{logoUrl}`; записва се в `logoUrl` с канала | Operator+ |
| GET | `/logos/{id}.{png\|svg}?v=` | Каченото лого — публично, кешира се от edge-а | публичен |
| GET | `/streams/{id}/thumbnail.jpg` | Последна снимка | Viewer+ |
| POST | `/streams/bulk` | `{ids[], action: start\|stop\|move\|addToPackage, params}` | Operator+ |
| POST | `/inputs/probe` | `{type, url, timeoutSec?, options?}` → резултат от ffprobe до 10 сек (FR-INP-07, TVR-022) | Operator+ |

> Лога (TVR-047, FR-PLY-04): `logoUrl` е `http(s)` адрес или пътят на качено лого (`/logos/{id}.png?v=…`). Качването не променя канала (версията не расте): формата слага върнатия URL в полето и го записва с останалото; запис с друг адрес трие качения файл. SVG със скриптове, обработчици на събития, `javascript:`, вграден HTML, entity-та или връзки към други адреси се отказва (400); SVG се сервира с `Content-Security-Policy: …; sandbox` и `nosniff`. С текущото `?v=` отговорът е `Cache-Control: public, max-age=31536000, immutable` (ново качване = нов URL), `ETag` = SHA-256. В M3U пътят става абсолютен с адреса за абонатите (`tvg-logo`). Сервира се от core през edge-а (кеш), origin-а и nginx-а на панела; във Vite — proxy за `/logos`.

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

**Анализатор (TVR-026):**

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

> TsSwitcher на агента анализира активния вход: битрейт на целия TS и на всеки PID (средно за 5 сек), кодеци и езици от PMT, CC грешки, scrambling. Идва със статуса на канала (на 5 сек) и се пази в Redis 30 сек. `at: null` (и празни `pids`) = няма данни: каналът е спрян, още тръгва или агентът не докладва. 404 за непознат канал.

**Лог на канала (TVR-023):**

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

> Core пита агента на сървъра на канала (gRPC `FetchLogs`) и чака до 5 сек. Агентът пази последните 1 000 реда на канал (изход на FFmpeg + събития старт/изход/рестарт), и след спиране на канала — до рестарт на агента. URL-ите в редовете са с маскирани пароли/токени. `level` е предположение по думите в реда. Грешки: 400 (`tail` извън 1–1000), 404, 409 (няма сървър / сървърът не е свързан), 504 (агентът не отговори до 5 сек).

**Тест на вход (TVR-022):**

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

> ffprobe се пуска на машината на tvr-core с твърд лимит 9 сек — отговорът идва до 10 сек. Грешки: 400 с `errors` (правилата за вход на канал; DVB не се тества); **422** с `code` (`TIMEOUT`, `CONNECTION_REFUSED`, `HOST_NOT_FOUND`, `NETWORK_UNREACHABLE`, `NOT_FOUND`, `ACCESS_DENIED`, `HTTP_ERROR`, `INVALID_DATA`, `NO_STREAMS`, `PROTOCOL_NOT_ALLOWED`, `FAILED`), `reason` (последният ред от ffprobe) и `probeMs` — `detail` е текст на български, `code` е за превода в панела; 429 при повече от 4 едновременни теста; 503 с `code: FFPROBE_MISSING`.

### 2.3 Категории и пакети

| Метод | Път | Описание |
|---|---|---|
| GET/POST | `/categories` | Списък (по `sortOrder`, после име) / създаване `{name, sortOrder?}` — без `sortOrder` отива последна |
| GET/PUT/DELETE | `/categories/{id}` | Детайл / редакция / изтриване (каналите остават без категория) |
| PUT | `/categories/order` | `[id, id, …]` — пълна подредба на всички категории; връща новия списък |
| GET/POST | `/packages` | Списък (по име) / създаване `{name, description?, active?}` |
| GET/PUT/DELETE | `/packages/{id}` | Детайл / редакция / изтриване (`?force=true`, ако пакетът е на абонати) |
| GET/PUT | `/packages/{id}/streams` | Каналите на пакета по ред / нов пълен списък `[{streamId, sortOrder?}]` — добавяне, махане и подредба наведнъж; връща новия списък |

> Категории (TVR-013): четене — всяка роля, промяна — ADMIN/OPERATOR. Името е уникално без значение от главни/малки букви (409 с `errors[].field = name`), до 100 символа; `sortOrder` 0–100000. Отговорът съдържа `streamCount` (неизтрити канали). При `PUT /categories/order` всяка категория трябва да е точно веднъж — иначе 400.

> Пакети (TVR-038): четене — всяка роля, промяна — ADMIN/OPERATOR. Името е уникално без значение от главни/малки букви (409, `errors[].field = name`), до 100 символа; описание до 1000; `active=false` — пакетът не дава достъп (TVR-041/042). Отговорът съдържа `streamCount` (неизтрити канали) и `subscriberCount`. `PUT /packages/{id}/streams`: всеки канал най-много веднъж, само неизтрити (иначе 400 с `errors[i].streamId`); редът е по `sortOrder`, без него — по позицията в масива; записва се като 0, 1, 2…; до 2000 канала. Изтрит (меко) канал не се показва, но връзката остава за възстановяване. `DELETE` на пакет с абонати без `force=true` → 409 с `code = PACKAGE_HAS_SUBSCRIBERS` и `subscriberCount`; изтриването маха само връзките — каналите остават. Плейлистите се правят при заявка (TVR-041), затова промяната важи веднага (FR-PKG-02). Списъкът `/streams?packageId=` връща каналите на пакета.

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
| POST | `/subscribers/actions/expire` | Нощното изтичане сега (TVR-046): ACTIVE с минал последен ден → EXPIRED → `{today, expired}` | Admin |
| GET | `/subscribers/{id}/history` | История от одит лога: кой, какво, кога (TVR-040) | Support+ |
| GET | `/subscribers/{id}/devices` · DELETE `/devices/{deviceId}` | Устройства | Support+ |
| GET | `/subscribers/{id}/sessions` | Активни сесии и история | Support+ |
| POST | `/subscribers/import` (CSV) · GET `/subscribers/export` | Масови операции | Admin |

> Абонати (TVR-039): `POST`/`PUT` с `{username, password?, fullName?, phone?, email?, expiresAt?, maxConnections?, limitPolicy?, packageIds?, allowedIps?, allowedCountries?, note?}`; статусът се сменя само с действията. `username` — `[A-Za-z0-9._@-]{3,64}`, уникално без значение от регистъра (409); паролата се пази като Argon2id и не се връща (`hasPassword`); `""` я маха. Отговорът носи `token` (32 base62 символа), `playlistUrl` (`{TVR_PLAYBACK_SUBSCRIBER_URL или TVR_PLAYBACK_BASE_URL}/pl/{token}.m3u`), `daysLeft`, `packages`. `expiresAt` е последният ден с достъп (Europe/Sofia), `null` — без край; не може да е в миналото (освен непроменена при редакция). `extend`: точно едно от `days` (1–3650, от текущата дата или от днес, ако е минала/няма) и `until`; изтекъл абонат става ACTIVE. `suspend`: ACTIVE → SUSPENDED; `resume`: SUSPENDED → ACTIVE, BANNED → ACTIVE само Admin, EXPIRED → 409 (ползвайте extend); `ban`: всеки → BANNED; `regenerate-token`: нов токен, старият спира веднага. Всяка промяна трие кеша `auth:<token>` в Redis. Списъкът: `q` (име, телефон, имейл, потребител или точен токен), `status`, `packageId`, `expiresBefore` (включително), `sort`. Роли: Support+; DELETE — Admin; Viewer няма достъп. Автоматично изтичане (TVR-046, BR-01): всеки ден в 00:00 Europe/Sofia и при старт на core ACTIVE абонатите с минал последен ден стават EXPIRED (запис `EXPIRE` от „система“ в историята), кешът `auth:<token>` и устройствата им се трият — гледането спира до ~10 сек (кеш на edge-а) + сегментите в буфера; SUSPENDED/BANNED не се пипат. Кикването на сесии — TVR-066.

### 2.5 Сесии

| Метод | Път | Описание |
|---|---|---|
| GET | `/sessions` | Активни сесии (филтри: `streamId, subscriberId, nodeId, country`) |
| DELETE | `/sessions/{sessionKey}` | Kick |
| DELETE | `/subscribers/{id}/sessions` | Kick на всички сесии на абонат |

### 2.6 Сървъри

| Метод | Път | Описание |
|---|---|---|
| GET | `/nodes` | Списък (по име): `status` (ONLINE/OFFLINE/DRAIN), `lastSeenAt`, `agentVersion`, `ffmpegVersion`, `connected` (има отворен gRPC поток), `metrics` — последните метрики на агента (като `node.metrics`; `null`, ако не са дошли през последните 30 сек, TVR-035) |
| POST | `/nodes/enrollment-tokens` | Ключ за регистрация на нов сървър (валиден 1 ч) |
| GET/PUT/DELETE | `/nodes/{id}` | Детайл / редакция / премахване |
| POST | `/nodes/{id}/actions/drain` · `undrain` | Режим поддръжка |

### 2.7 Мониторинг, известия, одит, настройки

| Метод | Път | Описание |
|---|---|---|
| GET | `/dashboard/summary` | Таблото (TVR-034): канали по състояние, канали с проблем (до 20), сървъри, последните 10 събития за 24 ч, `egressBps` — мрежов изход на всички сървъри с пресни метрики (TVR-035); зрители/абонати — `null` до съответните тикети |
| GET | `/events?types=` | SSE (TVR-030): `stream.status`, `stream.failover`, `stream.event`, `node.status`, `node.metrics`; по-късно `session.count`, `alert`. Вижте по-долу | Viewer+ |
| GET/POST/PUT/DELETE | `/alert-rules`, `/notification-channels` | Правила и канали за известия |
| POST | `/notification-channels/{id}/test` | Тестово известие |
| GET | `/alerts?state=FIRING` · POST `/alerts/{id}/ack` | Инциденти |
| GET | `/audit?actorId&entityType&from&to` | Одит лог |
| GET/PUT | `/settings` | Глобални настройки |
| POST | `/import/m3u` · `/import/astra` | Импорт (връща preview; `?commit=true` записва) |
| GET/POST | `/config/export` · `/config/import` | Пълен backup на конфигурацията |
| GET | `/reports/{type}?from&to` | `top-streams`, `peak-viewers`, `uptime`, `subscribers` |
| GET/POST · PUT/DELETE `/{id}` | `/admins` | Потребители на панела и техните роли (TVR-048, ADMIN) |
| GET/POST/DELETE | `/api-keys` | API ключове |

> Роли (TVR-048, FR-ADM-02, документ 02 §14) се проверяват от core при всяка заявка: OPERATOR само чете абонатите; SUPPORT променя абонати (без DELETE и без ръчното изтичане), няма `/nodes` и получава каналите с входове без `url`/`options` (без `/logs`, TC-12); VIEWER чете всичко без потребителите на панела, с `url` на входовете и `token`/`playlistUrl` на абонатите = `null`. `/admins`: не за себе си (роля, активен, изтриване → 409), остава поне един активен ADMIN (409); промяна отнема refresh токените на потребителя — новите права важат до 15 мин (ролята е в access токена).

**Реално време — `GET /api/v1/events` (TVR-030, FR-MON-02, NFR-10 ≤ 2 сек).** Server-Sent Events (`text/event-stream`). Токенът е в `Authorization: Bearer …` или — за `EventSource` в браузъра, който не праща заглавки — в `?access_token=` (само за този път). `?types=` избира типовете (със запетая; `stream.*`, `node.*`); без него идват всички, непознат тип → 400.

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

| Събитие | Кога | `data` |
|---|---|---|
| `ready` | веднага след свързване | `at`, `until` (кога потокът ще свърши), `types` |
| `stream.status` | всеки отчет на агента (≈ 5 сек и при промяна) и Stop през API | `streamId, state, previousState, changed, activeInputId, bitrateIn, since, lastError, nodeId, at`; `changed` = ново състояние или вход |
| `stream.failover` | агентът смени входа | `streamId, type: FAILOVER, nodeId, inputId, message, data{from, to, from_index, to_index, reason}, at` |
| `stream.event` | друго събитие на агента (`PROCESS_EXITED` …) | като `stream.failover` |
| `node.status` | сървър стана ONLINE / OFFLINE | `nodeId, name, status, lastSeenAt, at` |
| `node.metrics` | метрики от агента на 5 сек (TVR-035); `cpuPercent` −1 = още не е измерено | `nodeId, at, cpuPercent, memUsedBytes, memTotalBytes, netRxBps, netTxBps, tmpfsFreeBytes, tmpfsTotalBytes, streamsRunning, processes, loadAvg1m, cpuCores` |
| `bye` | край на потока | `reason`: `token-expired`, `max-duration` (1 ч), `slow-client` (> 1000 чакащи събития), `shutdown` |

- `: ping` на 15 сек без събития (проксита не затварят връзката).
- Потокът свършва, когато токенът изтече: клиентът се свързва отново с нов токен и презарежда списъците през REST — събитията не се преиграват (няма `Last-Event-ID`).
- Между инстанциите на core събитията минават през Redis pub/sub (канал `events`, документ 04 §3), така че всеки клиент вижда всичко. Без Redis събитията стигат само до клиентите на същата инстанция.
- Най-много 500 отворени потока на инстанция (`tvr.live.max-clients`) → 503. nginx пред core не трябва да буферира (`X-Accel-Buffering: no` се праща).

## 3. Публични (абонатски) endpoints — на edge домейна

| Метод | Път | Описание |
|---|---|---|
| GET | `/pl/{token}.m3u?type=hls\|ts` | Персонален плейлист |
| GET | `/epg/{token}.xml.gz` | Филтриран XMLTV |
| GET | `/live/{slug}/index.m3u8?token=` | HLS (авторизация чрез `auth_request`) |
| GET | `/live/{slug}/{segment}.ts?s=&e=` | Сегмент: подпис от плейлиста (HMAC-SHA256, 60 сек); без/подправен → 403, изтекъл → 410 (TVR-044) |
| GET | `/ts/{slug}.ts?token=` | HTTP-TS |
| GET | `/get.php?username&password&type=m3u_plus` | Xtream съвместимост (R1) |
| GET | `/player_api.php?username&password&action=` | Xtream съвместимост (R1) |
| GET | `/live/{username}/{password}/{streamId}.{ts\|m3u8}` | Xtream съвместимост (R1) |

> Плейлист (TVR-041): публичен, токенът в пътя е ключът. Строи се при всяка заявка: каналите от **активните** пакети на абоната, всеки по веднъж, неизтрити, включени или „при поискване“, с публичен включен изход от избрания тип (`type=hls` по подразбиране, `type=ts` — само каналите с HTTP-TS); ред по LCN, без номер — след тях по име. `#EXTINF:-1 tvg-id="{epgId|slug}" tvg-name tvg-logo group-title="{категория}" tvg-chno="{lcn}",{име}` и `{база}/live/{slug}/index.m3u8?token={token}`. Плейлист получава само ACTIVE абонат, чийто последен ден не е минал; иначе 403, непознат токен — 404 (кратък текст, не problem+json). Браузър (Accept: text/html) получава `text/plain`, плейърите — `audio/x-mpegurl`; `Cache-Control: no-store`. В dev линкът е през edge-а на :8000 (`TVR_PLAYBACK_SUBSCRIBER_URL`, TVR-043), който препраща `/pl/` към tvr-core (origin-ът на :8081 също). Проверката на `?token=` при самото пускане идва с TVR-042/043.

## 4. Вътрешни endpoints

| Метод | Път | Извиква | Описание |
|---|---|---|---|
| GET | `/internal/auth` | nginx `auth_request` | Входни headers: `X-Original-URI` (заявката на плейъра с `?token=`), `X-Real-IP`, `X-Internal-Token` (ако е зададен). Отговор `200` + `X-Subscriber-Id`, `X-Session-Id`, `X-Sign-Expires` / `401` / `403` + `X-Deny-Reason: TOKEN\|EXPIRED\|SUSPENDED\|BANNED\|NOT_IN_PACKAGE\|IP\|LIMIT` (`GEO` — TVR-097) |
| gRPC | `AgentControl.Connect(stream)` | tvr-agent | Двупосочен поток (виж [03-architecture](03-architecture.md) §7) |

> `/internal/auth` (TVR-042): токенът и каналът идват от `X-Original-URI` (`/live/{slug}/index.m3u8?token=` или `/ts/{slug}.ts?token=`); за ръчни проверки — и `?token=&slug=&ip=`. Проверките по ред: токен (няма/непознат → 401 `TOKEN`) → статус (`SUSPENDED`, `BANNED`, `EXPIRED`) → последният ден не е минал (`EXPIRED`, без да чака нощната задача) → каналът е в **активен** пакет на абоната и не е изтрит (`NOT_IN_PACKAGE`, TC-07) → IP в allow list-а, ако има такъв (`IP`). Правата се кешират в Redis `auth:<token>` за 10 сек (`tvr.access.cache-ttl`), вкл. непознати токени (`"-"`); всяка промяна по абоната през API трие ключа веднага, промените по пакети/канали се виждат до 10 сек. `X-Sign-Expires` = сега + 60 сек — информативно: edge-ът смята срока на подписа сам (`TVR_SIGN_TTL`), защото кешираният отговор носи старо време (TVR-044); С `TVR_INTERNAL_TOKEN` заявка без съвпадащ `X-Internal-Token` получава 404. Измерено в облака (2 ядра): ~1 650 заявки/сек, p99 ≈ 14 ms.

> Лимит на устройствата (TVR-045): устройство = абонат + `X-Real-IP` + `User-Agent` (BR-06; `sessionKey` = 22 знака base64url от SHA-256, не показва IP/UA) — смяната на канал от същото устройство не е нова сесия. Само заявка, минала всички останали проверки, се брои като активност; устройството е активно 30 сек след последната такава (`tvr.access.session-window`, FR-SES-03). С 10-секундния кеш на edge-а гледащо устройство стига до core поне на ~10 сек; ключът на кеша включва user agent-а, за да стигне до core и второ устройство зад същия IP. Ново устройство над `maxConnections`: `REJECT_NEW` → 403 `LIMIT`; `KICK_OLDEST` → затваря устройството, започнало първо, и го отказва 2 мин (`tvr.access.kick-block`), иначе то би изгонило новото при следващата си заявка. Затвореното спира до ~10 сек (кеш) + 60 сек (подписани сегменти). Проверката е атомарна (Lua в Redis). При недостъпен Redis лимитът не се проверява (гледането не спира). `200` носи и `X-Session-Id`. Нов токен (regenerate-token) изтрива сесиите на абоната (BR-04).

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
