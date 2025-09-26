# DFSP — Backend: Глубокая спецификация (v1.0)

*Цель: подробно и простым языком описать устройство backend‑части DFSP: процессы, интерфейсы, данные, проверки, ошибки, наблюдаемость и тестирование. Этот документ — рабочая спецификация для команды разработки и руководства.*

---

## 0) Область документа

* Только **backend** (API, очереди/воркеры, релейер, взаимодействие с контрактами и IPFS, БД, наблюдаемость).
* Frontend, смарт‑контракты и high‑level продукт описаны в общей спецификации; здесь — детализация бэкенда.

---

## 1) Архитектура рантайма (что за процессы и как связаны)

**Процессы:**

1. **api** — FastAPI приложение (gunicorn/uvicorn workers). REST, JWT, валидации, gateway, PoW, квоты.
2. **worker.relayer** — Celery‑воркер высокого и обычного приоритета; отправляет meta‑tx в сеть.
3. **worker.anchor** — Celery‑воркер, собирает off‑chain события в меркль‑корень, пишет в контракт `Anchoring`.
4. **redis** — брокер/бэкенд очередей Celery и кеш квот/PoW.
5. **postgres** — основная БД для метаданных и журналов.
6. **ipfs.kubo** — узел IPFS, хранилище зашифрованных данных.
7. **chain.hardhat** — локальная блокчейн сеть (JSON‑RPC provider для web3.py).

**Зависимости:**

* api → postgres, redis, ipfs, worker.relayer (через очередь), chain (чтение статусов), worker.anchor (через события).
* worker.relayer → chain (запись), postgres (фиксация tx), redis (очередь).
* worker.anchor → postgres (читать события), chain (запись).

**Потоки:**

* **Control plane**: REST ↔ API ↔ (Celery) ↔ воркеры ↔ Chain.
* **Data plane**: Gateway API ↔ IPFS (стримы).

---

## 2) Конфигурация и секреты

* Формат: `pydantic-settings` (Typed Settings) + `.env`.
* Основные параметры:

  * `POSTGRES_DSN`, `REDIS_URL`.
  * `RPC_URL`, `RELAYER_PRIVATE_KEY`.
  * `IPFS_API`, `IPFS_GATEWAY_INTERNAL`.
  * `JWT_SECRET`, `JWT_ACCESS_TTL=15m`, `JWT_REFRESH_TTL=24h`.
  * `QUOTAS_*` (см. ниже), `POW_DIFFICULTY_MIN=16`, `POW_DIFFICULTY_MAX=24`.
  * `ANCHOR_PERIOD_MIN=60`.
* Секреты — в переменных окружения; локально — `.env`, на стенде — секрет‑хранилище GitHub.

---

## 3) Модель безопасности (backend)

* **TLS** на прокси (Caddy), API только за HTTPS.
* **JWT Bearer**: статeless доступ к REST. Claims: `sub` (user_id), `addr` (eth_address), `scopes`.
* **CORS**: разрешаем только фронтенд‑origin; блокируем `*`.
* **Ввод/вывод**: строгие схемы pydantic, экранирование имён файлов, `nosniff` и `attachment` для gateway.
* **Ключи пользователя** на сервер не попадают; на сервере только публичные RSA и encK.
* **Логи**: без IP/PII; ID хэшируем для корреляции.

---

## 4) Аутентификация (challenge → подпись → JWT)

**Цель:** подтвердить, что пользователь владеет EOA‑ключом (без email/пароля).

**Шаги:**

1. `POST /auth/challenge` → API создаёт nonce (16 байт), TTL=5 минут, привязывает к адресу и сохраняет (postgres или redis) с ключом `auth:nonce:<addr>`.
2. FE подписывает **EIP‑712** (typedData) со своим EOA и отправляет `POST /auth/login` или `POST /auth/register`.
3. API восстанавливает адрес из подписи, сверяет с заявленным `eth_address`, проверяет TTL nonce и одноразовость.
4. На `register` дополнительно сохраняет `rsa_public_spki_pem` и профиль.
5. API выпускает `access` и `refresh` JWT. Ротация refresh при обновлении.

**Схема EIP‑712:** домен `MinimalForwarder` или собственный `AuthDomain` (MVP — домен форвардера допускается, проверка off‑chain).

**Ошибки:**

* 400 (битые данные), 401 (подпись/nonce), 409 (повторный register), 429 (много попыток).

---

## 5) Идемпотентность

* Заголовок `X-Request-Id: <uuid-v4>` на всех изменяющих POST/PUT.
* Таблица `idempotency_keys(request_id, user_id, created_at)` с TTL‑очисткой.
* На повтор того же `request_id` — возвращаем *первый сохранённый результат* (или ссылку на него).

---

## 6) Квоты и rate limiting

**Инструменты:** Redis (атомарные инкременты, TTL), ключи вида `quota:<user_id>:<scope>:<yyyy-mm-dd>`.

**Скользящее окно (пример):**

* meta‑tx/day ≤ 50 → инкремент при каждой операции, TTL до конца суток по UTC.
* download_bytes/day ≤ 2 ГБ → считаем по заголовкам Range и фактически отданным байтам (с накоплением).
* concurrent_uploads ≤ 2 → счётчик активных загрузок с TTL‑«сердцебиением».

**Ответ при превышении:** 429 + JSON `{ code: "QUOTA_EXCEEDED", reset_at: ISO8601 }`.

---

## 7) Proof‑of‑Work (PoW) контроллер

**Зачем:** сдерживать бот‑нагрузку без IP и капч.

**Выдача челленджа:**

* `challenge = random(16B)`, `difficulty ∈ [16..24]`, TTL=30s.
* Таблица `pow_challenges(id, user_id, endpoint, challenge, difficulty, exp_at)` или Redis с TTL.

**Проверка:**

* На защищённых `POST/GET` API требует `(challenge, nonce)` и проверяет `leadingZeroBits(SHA256(challenge||nonce)) >= difficulty` и не истёк ли TTL.

**Автонастройка сложности:**

* EWMA среднего времени решения `T_avg` per‑endpoint.
* Если `T_avg < 80ms` → поднять на 1 (до max 24). Если `T_avg > 200ms` → снизить на 1 (до min 16).
* На «горячих» эндпоинтах (`/share`, `/download`) локальный буст +1..+2.

**Ошибки:** 429 `{ code: "POW_INVALID" | "POW_EXPIRED" }`.

---

## 8) Модуль файлов и gateway

**Загрузка (метаданные):** `POST /files` принимает `{ fileId, name, size, mime, cid, checksum }`, проверяет лимиты и форматы, готовит `typedData` для `FileRegistry.register`.

**Версии:** `POST /files/{id}/version` → `updateCid`.

**Gateway скачивания:** `GET /gateway/ipfs/<cid>` (внутренний роут, отдаётся только после `GET /download/{capId}`).

**Безопасность и заголовки:**

* `Content-Disposition: attachment; filename*=UTF-8''<safe>`.
* `X-Content-Type-Options: nosniff`, `Accept-Ranges: bytes`.
* Никогда `inline`.

**Range:** обрабатываем `Range: bytes=start-end`. При ошибке диапазона → 416.

---

## 9) Шаринг и выдача ключа (encK)

**Share:**

* Владелец отправляет `encK_map` (RSA‑OAEP‑2048 per‑recipient), `ttl_days`, `max_dl`.
* API валидирует, что `owner` совпадает, кладёт задачу `grant` в `relayer.default`.

**Download:**

* API проверяет `canDownload` (on‑chain чтение или кеш), TTL, `used < maxDownloads`, квоты и PoW.
* Кладёт `useOnce` в `relayer.high` (оптимистично) и **сразу** отдаёт `{ encK, gatewayUrl }`.
* Если on‑chain `useOnce` потом отвергнут (гонка/лимит), UI увидит фактический `used` и покажет, что лимит исчерпан.

---

## 10) Релейер и он‑чейн операции

**Celery задачи:**

* `register`, `updateCid`, `grant` → `relayer.default`.
* `useOnce`, `revoke` → `relayer.high`.

**Формат payload:** `{ type, request_id, forward_request, signature }`.

**Алгоритм:**

1. Проверить по `request_id` — не выполнено ли ранее.
2. Отправить `MinimalForwarder.execute(...)` через web3.py.
3. Ждать receipt (1 подтверждение достаточно для локальной сети).
4. Сохранить `tx_hash`, `status`, `gasUsed`.
5. Ошибки `nonce too low`/`underpriced` → ретраи с увеличением gas price.

**Таблица статусов:** `tx_requests(request_id, type, tx_hash, status(enum), created_at, updated_at, gas_used)`.

---

## 11) Anchoring‑воркер (часовой)

**Сборка журнала:**

* Берём `events` за период `period_id = floor(ts / 3600)`.
* Лист `leaf = keccak256(file_id || payload_hash || ts || type)`.
* Меркль‑дерево (двойной keccak по парам). Корень `root`.

**Он‑чейн:** `anchorMerkleRoot(root, periodId)`.

**Проверки:**

* Если нет событий — пропускаем период (логируем «empty period»).

---

## 12) База данных (детали)

**Схема (основные таблицы):** см. общий документ. Дополнительно:

* `idempotency_keys(request_id uuid pk, user_id uuid, created_at timestamptz)`.
* `tx_requests(...)` (см. выше).
* `pow_challenges(id uuid pk, user_id uuid, endpoint text, challenge bytea, difficulty int, exp_at timestamptz)` или Redis.

**Консистентность:**

* Источник истины по правам/использованию — **on‑chain**. БД хранит кэш для UI.
* Периодический синк: воркер парсит события контрактов и обновляет `grants.used/revoked_at` и пр.

**Индексы и FK:** см. общий документ + индексы на `tx_requests(status, created_at desc)`.

---

## 13) Интерфейсы API (полезные примеры)

### 13.1 Auth

**POST /auth/challenge**

```json
{
  "nonce": "b64...",
  "exp": "2025-09-26T12:00:00Z"
}
```

**POST /auth/register**

```json
{
  "rsa_public_spki_pem": "-----BEGIN PUBLIC KEY-----...",
  "eth_address": "0xabc...",
  "display_name": "Alice",
  "challenge": {"nonce":"b64...","exp":"..."},
  "signature": "0xSIG..."
}
```

→ `200 {"access":"...","refresh":"..."}`

### 13.2 Files

**POST /files**

```json
{
  "fileId":"base64-32",
  "name":"report.pdf",
  "size": 10485760,
  "mime":"application/pdf",
  "cid":"bafy...",
  "checksum":"base64-32"
}
```

→ `200 {"typedData":{...}, "request_id":"..."}`

**POST /meta-tx/submit**

```json
{"signedForwardRequest": {"request":{...}, "signature":"0x..."}}
```

→ `200 {"txHash":"0x..","status":"queued"}`

### 13.3 Share/Download

**POST /files/{id}/share**

```json
{
  "users":["uuid-grantee-1","uuid-grantee-2"],
  "ttl_days":7,
  "max_dl":1,
  "encK_map": {"uuid-grantee-1":"b64...","uuid-grantee-2":"b64..."}
}
```

**GET /download/{capId}**
→ `200 {"encK":"b64...","gatewayUrl":"/gateway/ipfs/bafy..."}`

**Ожидаемые ошибки**: 403 (нет права/TTL), 409 (лимит), 429 (PoW/квоты).

---

## 14) Ошибки и коды

| Код | code                 | Когда                               |
| --- | -------------------- | ----------------------------------- |
| 400 | VALIDATION_ERROR     | Неверные поля/форматы               |
| 401 | AUTH_FAILED          | Неверная подпись/nonce/нет токена   |
| 403 | FORBIDDEN            | Нет права (TTL/лимит/revoke)        |
| 404 | NOT_FOUND            | Файл/грант/ресурс отсутствует       |
| 409 | CONFLICT             | Гонка/useOnce, повторный request_id |
| 429 | QUOTA_EXCEEDED       | Превышены квоты                     |
| 429 | POW_INVALID/EXPIRED  | PoW не прошёл или истёк             |
| 500 | INTERNAL             | Неожиданная серверная ошибка        |
| 502 | UPSTREAM_BAD_GATEWAY | IPFS/chain недоступен               |
| 504 | UPSTREAM_TIMEOUT     | Таймаут IPFS/chain                  |

---

## 15) Наблюдаемость (метрики, логи, алерты)

**Метрики (Prometheus):**

* `http_requests_total{route,code}`
* `http_request_duration_seconds_bucket{route}`
* `celery_tasks_total{type,status}`
* `relayer_tx_gas_used{type}`
* `pow_solve_time_seconds_bucket{endpoint}`
* `gateway_bytes_sent_total`

**Логи (JSON):** `ts, level, trace_id, user_id?, action, result, duration_ms, error_code?`.

**Алерты:**

* 5xx > 1% за 5 минут.
* lag `relayer.high` > 30s.
* p95 `/download` init > 800ms 10 минут подряд.
* spike `QUOTA_EXCEEDED`/`POW_INVALID`.

---

## 16) Тестирование backend

**Юнит‑тесты:** pydantic‑схемы, валидации, PoW проверка, генерация typedData.

**Интеграционные:**

* Mock IPFS/Kubo (http сервер), Mock Chain (ganache/hardhat node).
* Полный путь: `/files` → meta‑tx → `register` (через воркер).
* Gateway Range: проверки корректных/некорректных диапазонов.

**Property‑based (Hypothesis):**

* PoW: случайные `challenge`, проверка корректности `leadingZeroBits`.
* Идемпотентность: один и тот же `request_id` при сетевых сбоях.

**Нагрузочные:**

* Locust/k6: `/download` RPS, влияние PoW и квот, лаг очереди `relayer.high`.

---

## 17) Производительность и ресурсы

* API: gunicorn 2–4 воркера uvicorn, keep‑alive 60s.
* Таймауты: IPFS 10–30s (конфигурируемо), Chain 10s, Gateway stream idle 120s.
* Размеры: upload ≤ 200 МБ, одновременных загрузок ≤ 2, скачивания считаем по Range.

---

## 18) Развёртывание

* Docker образы: `api`, `worker`, `ipfs`, `db`, `redis`, `chain`, `proxy`.
* Health‑checks: `/healthz` (API), Celery ping, IPFS `/api/v0/id`.
* Логи в stdout (для сборки Promtail/Fluent).

---

## 19) Структура исходников (backend)

```
backend/
  app/
    api/            # роуты FastAPI
    schemas/        # pydantic модели запрос/ответ
    services/       # бизнес‑логика (share, download, verify, pow)
    repos/          # доступ к БД
    security/       # auth, jwt, idempotency
    ipfs/           # клиенты IPFS и gateway
    relayer/        # формирование и отправка meta‑tx (web3.py)
    workers/        # Celery задачи (relayer, anchor, sync)
    contracts/      # ABI/адреса и хелперы
    settings.py
    telemetry/      # метрики/логи/трейсинг
  tests/
    unit/
    integration/
    property/
  Dockerfile
  pyproject.toml
  alembic/
```

---

## 20) Псевдокод ключевых мест

**Проверка PoW:**

```python
h = sha256(challenge + nonce.to_bytes(4, 'big')).digest()
if leading_zeros(h) < difficulty:
    raise HTTPException(429, detail={"code":"POW_INVALID"})
```

**Download контроллер (упрощённо):**

```python
def download(cap_id, pow):
    verify_pow(pow)
    grant = repo.get_grant(cap_id)
    if not can_download(grant):
        raise HTTPException(403, detail={"code":"FORBIDDEN"})
    enqueue_use_once(cap_id)
    encK = repo.get_encK(grant.grantee_id, grant.file_id)
    url = f"/gateway/ipfs/{repo.get_cid(grant.file_id)}"
    return {"encK": encK, "gatewayUrl": url}
```

**Relayer задача:**

```python
def send_meta_tx(req):
    if repo.exists_tx(req.request_id):
        return
    tx = forwarder.execute(req.forward_request, req.signature)
    receipt = wait(tx)
    repo.save_tx(req.request_id, tx.hash, receipt.status, receipt.gasUsed)
```

---

## 21) Миграции и версии

* Alembic для схемы БД: один каталог миграций, версии по семантике релизов.
* Контракты без прокси: хранить адреса/версии в `.env` и в таблице `contracts(version, address, deployed_at)`.
* Совместимость API: мажорные изменения — через новые маршруты или флаг `version` в схеме.

---

## 22) Риски и меры (backend)

* **IPFS недоступен** → таймауты/ретраи, 504, алерт.
* **Очередь переполнена** → алерт по лагу, деградация PoW/квот.
* **Долгие подтверждения** → оптимистический `useOnce`, UI показывает «идёт подтверждение».
* **Искажение idempotency** → отдельная таблица ключей + детерминированные ответы.

---

## 23) Готово к разработке (DoR) — чек‑лист

* [ ] Согласованы поля всех эндпоинтов и коды ошибок.
* [ ] Настроены окружения и секреты.
* [ ] Приняты лимиты/квоты/сложность PoW.
* [ ] Зафиксированы адреса контрактов в `.env` для MVP.

## 24) Готово к приёмке (DoD) — чек‑лист

* [ ] Все эндпоинты покрыты интеграционными тестами.
* [ ] Нагрузочные тесты на `/download` и PoW.
* [ ] Метрики и алерты в Grafana.
* [ ] Демо‑сценарии выполняются стабильно.

---

*Конец документа.*
