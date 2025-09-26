# DFSP — Полная спецификация проекта (v1.0)
---

## 0) TL;DR (коротко)

**DFSP** — это приватный обмен файлами через интернет. Файлы шифруются **на стороне клиента** (в браузере) перед отправкой в **IPFS** (распределённое хранилище). Доступ к ключу шифрования выдаётся и контролируется **смарт‑контрактами** (on‑chain), а все действия выполняются без привычных криптокошельков — через **meta‑transactions (ERC‑2771)**: пользователь подписывает сообщение, а наш релейер отправляет транзакцию за него.

Ключевая идея: даже если файл физически находится в публичной сети IPFS, без **ключа** его нельзя прочитать. Мы управляем *выдачей ключей* и *лимитами* полностью on‑chain.

---

## 1) Назначение документа

Этот документ — **спецификация разработки DFSP**. Здесь описано *как именно* устроена система на уровне модулей, какие интерфейсы между ними, какие данные ходят, какие проверки делаются и что происходит при ошибках. Документ предназначен для руководства и команды разработки: формулировки максимально простые, но технически точные.

## 2) Глоссарий

* **IPFS** — распределённое хранилище, адресация по **CID** (контент‑хэш).
* **CID** — идентификатор содержимого файла в IPFS.
* **On‑chain** — логика на блокчейне (смарт‑контракты).
* **Meta‑transaction** — пользователь подписывает сообщение; наш релейер платит газ и отправляет on‑chain.
* **ERC‑2771** — стандарт «доверенного форвардера», корректно передаёт реального адресата операции.
* **encK** — зашифрованный ключ файла (ключом получателя).
* **TTL** — срок действия гранта (права).
* **useOnce** — одноразовая выдача ключа.
* **Anchoring** — периодическая фиксация корня (Merkle root) off‑chain журнала событий в блокчейне.

---

## 3) Объём работ (MVP) и что вне MVP

### 3.1 В скоупе MVP

* Загрузка файлов в IPFS с шифрованием **на клиенте**.
* Регистрация метаданных и checksum на смарт‑контракте `FileRegistry`.
* Выдача прав доступа `AccessControlDFSP` (TTL, лимиты выдач `encK`, revoke, useOnce).
* Скачивание с проверкой права и выдачей `encK` + расшифровка в браузере.
* Страница Verify (сверка CID/checksum с он‑чейн).
* Anchoring (1×/час): фиксация корня журнала событий.
* Антиабьюз: квоты + лёгкий Proof‑of‑Work.
* Наблюдаемость: метрики/логи + базовый дашборд.

### 3.2 Вне MVP (но с «крючками»)

* Публичные ссылки (bearer capabilities).
* Группы (скрытый состав через Merkle‑root).
* HPKE/X25519 для доставки ключей.
* PWA/тёмная тема/мобильные браузеры и Safari (desktop/mobile).
* Репликация IPFS/пиннинг‑сервисы, CAR‑бэкапы.

---

## 4) Архитектура: подробная спецификация

### 4.1 Карта модулей (кто за что отвечает)

* **Frontend (браузер):** шифрует/расшифровывает файлы, хранит ключи, подписывает EIP‑712, решает мини‑задачу PoW, общается с API, показывает интерфейс.
* **Backend API:** проверяет права, выдаёт PoW‑челленджи, принимает метаданные файлов, готовит и отправляет мета‑транзакции через релейер, отдаёт `encK` и маршрутизирует скачивание через приватный gateway, ведёт журнал событий.
* **Relayer & Workers:** очереди задач для он‑чейн операций (grant/revoke/register/useOnce/anchor), ретраи, контроль nonce, приоритеты.
* **Смарт‑контракты (локальная сеть Hardhat):** `FileRegistry` (мета файлов), `AccessControlDFSP` (права), `Anchoring` (фиксация журнала), `MinimalForwarder` (ERC‑2771).
* **IPFS (Kubo):** хранит зашифрованные чанки, возвращает по CID. Пин включён. Доступ идёт через наш API‑gateway.
* **Postgres:** хранит пользователей, файлы, версии, гранты, события, анкоры.
* **Прокси/TLS (Caddy):** HTTPS, проксирование API и IPFS‑gateway, безопасные заголовки.

### 4.2 Границы модулей и интерфейсы (как они говорят друг с другом)

**Общее правило:** все внешние интерфейсы — это простые JSON‑запросы по HTTPS. Внутри — чёткие контракты и идемпотентные операции.

**FE ↔ API (REST):**

* Аутентификация: `POST /auth/*` (см. §8), заголовок `Authorization: Bearer <access>` для всех защищённых вызовов.
* Идемпотентность: для важных операций FE отправляет заголовок `X-Request-Id` (UUIDv4), API гарантирует «выполнить один раз».
* Синхронизация статусов: после мета‑tx API возвращает `txHash` и поле `status` (`queued|sent|mined|failed`). FE периодически опрашивает `GET /tx/{id}` или получает обновления по SSE/WebSocket (опционально, MVP — polling).

**API ↔ Relayer (очереди):**

* Формат задачи: `{type, payload, request_id, priority}`; типы: `register|grant|useOnce|revoke|updateCid|anchor`.
* SLA очередей: `relayer.high` обрабатывается раньше `relayer.default`.

**Relayer ↔ Chain (JSON‑RPC):**

* Релейер собирает `ForwardRequest` и подпись пользователя, отправляет через контракт `MinimalForwarder`.
* Стратегия газа: локальная сеть — номинально; фиксируем `gasUsed` для метрик.

**API ↔ IPFS (Kubo HTTP API):**

* Загрузка: `add` (через ipfs‑client в API при валидациях/статистике) — в MVP сам Upload выполняет FE прямо в Kubo, но API может делать `pin/add`/`pin/rm`.
* Скачивание: приватный роут `/gateway/ipfs/<cid>` с проверками доступа; API проксирует поток из Kubo.

**API ↔ Postgres:**

* Все операции через ORM, транзакции там, где нужно обеспечить консистентность (`grants.used` инкремент и т.п.).
* Миграции через alembic; схемы ниже (§4.9).### 4.3 Потоки данных (две «плоскости»)
* **Data Plane (содержимое файлов):** в браузере шифруем → отправляем в IPFS → скачиваем из IPFS через наш gateway → расшифровываем в браузере. Никакого открытого содержимого через наш сервер не проходит.
* **Control Plane (права и учёт):** в контрактах лежат правила (кто и сколько раз может получить ключ). API проверяет право, считает квоты/PoW, инициирует он‑чейн записи.

**Диаграмма последовательности: Download (упрощённо текстом)**

1. FE → API: `GET /download/{capId}` + `PoW`.
2. API: проверка TTL/лимитов/квот → `enqueue(useOnce)` (priority=high) → ответ `{encK, gatewayUrl}`.
3. FE: расшифровывает `encK` → получает `K_file` → скачивает через `gatewayUrl` → расшифровывает чанки.
4. Worker: шлёт `useOnce` через `MinimalForwarder` → on‑chain `Used(capId)`.

Схема:

```
[Browser/FE] --REST--> [Backend API] --queue--> [Relayer] --RPC--> [Hardhat Chain]
     |                         |                               ^
     |                         +----SQL----> [Postgres]        |
     |                         +----HTTP---> [IPFS Kubo] <-----+
     +-----------download via gateway (with checks)------------+
```

### 4.4 Frontend: как устроен внутри

**Подсистемы и структуры данных**

1. **Key Store (IndexedDB)**

   * Запись `keys`: `{ rsa_priv_encrypted, rsa_pub_spki_pem, eoa_priv_encrypted, created_at }`.
   * Шифрование приватных ключей: пароль пользователя → `PBKDF2(password, salt, 210k, SHA‑256)` → `KEK` → `AES‑GCM(KEK, iv)`.
   * Бэкап `.dfspkey`: JSON контейнер `{ salt, iv, ciphertext }` + версия формата.
   * UX‑правило: без бэкапа кнопка **Share** неактивна; Upload разрешён.

2. **Crypto Manager**

   * Генерация `K_file`: `crypto.getRandomValues(32 bytes)`.
   * Разбиение на чанки: по ~4 МБ; последний чанк короче — это нормально.
   * HKDF‑nonce на чанк: `nonce_i = HKDF_SHA256(IKM=K_file, salt=fileId, info=LE(chunkIndex))[:12]`.
   * Шифрование: `AES‑GCM-256(plaintext_chunk, key=K_file, iv=nonce_i)` → `(ciphertext_i, tag_i)`.
   * Итоговый checksum: `keccak256(concat(ciphertext_0,...,ciphertext_n))`.
   * **Память:** используем поточный подход (читали → зашифровали → отправили → очищаем буфер), чтобы не держать весь файл в RAM.

   **Псевдокод шифрования чанков:**

   ```
   fileId = keccak256(owner_addr || random16() || now())
   K_file = randomBytes(32)
   checksum_hasher = Keccak256()
   for (i, chunk in readFileStream(file, 4MB)):
       iv = HKDF_SHA256(IKM=K_file, salt=fileId, info=uint32LE(i)).slice(0,12)
       (cipher, tag) = AES_GCM_Encrypt(K_file, iv, chunk)
       emitToIPFS(cipher || tag)
       checksum_hasher.update(cipher)
   checksum = checksum_hasher.final()
   ```

3. **Uploader**

   * Конкурентность: до N=3 параллельных чанков (регулируется по сети/CPU).
   * Повторы: экспоненциальный бэкофф на сетевые ошибки; контроль дубликатов по локальным хешам чанков.
   * Прогресс: считаем из числа успешно добавленных байт.

4. **Signer (EIP‑712)**

   * Принимает `typedData` от API и вызывает `ethereum.request({ method: 'eth_signTypedData_v4', ... })` (в нашем случае — local EOA без внешнего кошелька; реализовано через библиотеку подписи).

5. **PoW Solver**

   * Получает `{challenge, difficulty, ttl}`.
   * Ищет `nonce` 32‑бит, чтобы `SHA256(challenge||nonce)` имел `difficulty` ведущих нулевых бит.
   * Если `ttl` истёк — запрашивает новый челлендж.

6. **Downloader/Decryptor**

   * Расшифровка `encK` приватным RSA → `K_file`.
   * Скачивание по `gatewayUrl` с поддержкой `Range` (возобновление).
   * Дешифрование чанков потоком; в конце сверяем `checksum` с он‑чейн (через `Verify`).

7. **State Store (Zustand)**

   * Сохраняем текущие прогрессы, статусы мета‑tx, активные PoW, список моих файлов и выданных мне грантов.

**Сценарии ошибок и UX‑решения**

* Нет бэкапа → Share недоступен (тултип объясняет почему и как сделать бэкап).
* Upload прервался → предлагается «продолжить»; уже загруженные чанки не повторяем.
* Ошибка PoW/квоты → читаем код/сообщение, показываем понятный баннер с таймером до сброса квот.

### 4.5 Backend API: как работает по слоям

**Слои и ответственность**

* **Auth:** challenge→подпись→JWT. Срок жизни `access` (например, 15 мин), `refresh` (например, 24 ч). Храним минимум данных — без PII.
* **Files:** принимает метаданные и возвращает typedData на `register/updateCid`.
* **Share/Grants:** валидирует параметры грантов, проверяет владельца, кладёт он‑чейн операции.
* **Download:** решает PoW, проверяет `canDownload`/TTL/лимиты/квоты, отдаёт `encK` и ссылку на gateway.
* **Gateway:** стримит контент из IPFS после проверок, выставляет безопасные заголовки, поддерживает `Range`.
* **PoW Controller:** управляет сложностью (диапазон [16;24]) на основе среднего времени решения.
* **Events/Anchors:** пишет события и выдаёт данные для воркера анкоринга.

**Эндпоинты (форматы, примеры)**

* `POST /auth/challenge` → `{ nonce: base64, exp: ISO8601 }`

* `POST /auth/register` (body): `{ rsa_public_spki_pem, eth_address, display_name?, challenge, signature }` → `{ access, refresh }`

* `POST /auth/login` → `{ access, refresh }`

* `POST /auth/refresh` → `{ access }`

* `POST /files` (идемп): `{ fileId, name, size, mime, cid, checksum }` → `{ typedData, request_id }`

* `POST /meta-tx/submit` → `{ signedForwardRequest }` → `{ txHash, status }`

* `GET /files?mine=true` → `[ {id, name, size, mime, cid, checksum, created_at} ]`

* `GET /files/{id}` → `{ meta, versions, grants }`

* `POST /files/{id}/version` → `{ typedData }`

* `POST /files/{id}/share` → `{ users:[userId], ttl_days, max_dl, encK_map:{ userId: base64 } }` → `{ capIds:[...], txHash }`

* `POST /grants/{capId}/revoke` → `{ txHash }`

* `GET /grants/{capId}` → `{ capId, fileId, grantee, expiresAt, maxDownloads, used, revokedAt }`

* `GET /download/{capId}` (+PoW headers/body) → `{ encK: base64, gatewayUrl: "/gateway/ipfs/<cid>" }`

* `GET /verify/{fileId}` → `{ onchain: { cid, checksum }, status: "match|mismatch" }`

* `GET /anchors/latest` → `{ period_id, merkle_root, tx_hash }`

**Валидации и правила**

* `size` ≤ 200 МБ (MVP), mime — из списка безопасных (для заголовка).
* `checksum` — 32 байта (hex/base64), `fileId` — 32 байта.
* `ttl_days` — 1..365; `max_dl` — 1..1000.
* Идемпотентность: `X-Request-Id` на `POST /files` и на операции мета‑tx.

**Gateway: Range и безопасность**

* Поддержка заголовка `Range: bytes=start-end`; если отсутствует — отдаём с начала.
* Заголовки: `Content-Disposition: attachment; filename*=UTF-8''<safe-name>`, `X-Content-Type-Options: nosniff`, `Accept-Ranges: bytes`.
* Никогда не отдаём файл `inline`.

**Коды ошибок**

* 400 (валидация), 401 (нет токена/подписи), 403 (нет права/TTL/лимиты), 404 (файл/грант), 409 (гонка/idempotency), 429 (PoW/квоты), 5xx (временные).

### 4.6 Relayer & Workers: очередь он‑чейн действий

**Структура задачи (payload)**

```json
{
  "type": "useOnce",
  "request_id": "uuid-v4",
  "priority": "high",
  "forward_request": { "from":"0x..","to":"0x..","value":"0","gas":"100000","nonce":"1","data":"0x.." },
  "signature": "0x.."
}
```

**Алгоритм обработки**

1. Забрать из очереди по приоритету.
2. Проверить идемпотентность по `request_id` в БД (если уже `txHash` есть — завершить успехом).
3. Отправить через `MinimalForwarder`.
4. Ждать `mined` (или таймаут → ретрай до 5 раз с экспонентой).
5. Сохранить `txHash`, время, `gasUsed`.

**Nonce/ретраи**

* Ловим `nonce too low`/`underpriced` → увеличиваем газ/повторяем.
* При повторе обязательно проверяем, не прошла ли транзакция ранее (по `request_id`).

**Приоритеты**

* Очередь `relayer.high`: `useOnce`, `revoke`.
* Очередь `relayer.default`: `register`, `grant`, `updateCid`.

**Anchoring воркер**

* Раз в час собирает события из `events` за период, строит Merkle‑root (детали см. ниже) и вызывает `anchorMerkleRoot`.

### 4.7 Смарт‑контракты: что именно делают

**Интерфейсы (упрощённый Solidity‑скелет)**

```solidity
struct FileMeta { address owner; string cid; bytes32 checksum; uint64 size; string mime; uint64 createdAt; }
struct Grant   { address grantor; address grantee; bytes32 fileId; uint64 expiresAt; uint32 maxDownloads; uint32 used; uint64 revokedAt; }

contract FileRegistry {
  mapping(bytes32 => FileMeta) public meta;
  mapping(bytes32 => FileMeta[]) public versions;
  event FileRegistered(bytes32 indexed fileId, address indexed owner, string cid, bytes32 checksum);
  event FileVersioned(bytes32 indexed fileId, string newCid, bytes32 newChecksum);
  function register(bytes32 fileId, string calldata cid, bytes32 checksum, uint64 size, string calldata mime) external;
  function updateCid(bytes32 fileId, string calldata newCid, bytes32 newChecksum, uint64 newSize, string calldata newMime) external;
}

contract AccessControlDFSP {
  mapping(bytes32 => Grant) public grants; // capId -> Grant
  event Granted(bytes32 indexed capId, bytes32 indexed fileId, address indexed grantee, uint64 expiresAt, uint32 maxDownloads);
  event Revoked(bytes32 indexed capId);
  event Used(bytes32 indexed capId);
  function grant(bytes32 fileId, address grantee, uint64 ttlSec, uint32 maxDownloads) external returns (bytes32 capId);
  function revoke(bytes32 capId) external;
  function canDownload(address user, bytes32 fileId) external view returns (bool);
  function useOnce(bytes32 capId) external; // инкрементирует used, если не истёк/не отозван/не превышен лимит
}

contract Anchoring {
  event Anchored(bytes32 root, uint64 periodId);
  function anchorMerkleRoot(bytes32 root, uint64 periodId) external;
}
```

**Идентификаторы**

* `fileId` — bytes32 (см. FE‑алгоритм генерации).
* `capId` — bytes32, детерминированно из (`fileId`,`grantee`,`grantor`,`nonceGrant`) или генерируем в контракте (например, `keccak256` от полей).

**Инварианты и проверки**

* `register`: может вызвать только владелец `fileId` (первый вызов закрепляет владельца).
* `updateCid`: только владелец `fileId`.
* `grant`: только владелец `fileId`; `ttlSec > 0`, `maxDownloads > 0`.
* `useOnce`: грант не отозван, `now < expiresAt`, `used < maxDownloads` → `used++` и `emit Used`.
* `revoke`: только `grantor`.

**Безопасность**

* Все внешние вызовы идут через ERC‑2771; используем `_msgSender()`.
* События индексированы по ключевым полям (fileId, grantee, capId) для быстрой выборки.

**Газ (оценочно, локально)**

* `register`/`grant`/`useOnce` — лёгкие операции; для MVP фиксируем фактический `gasUsed` в метриках релейера.

### 4.8 IPFS (Kubo): хранение и скачивание

**Загрузка (от FE к Kubo):**

* Потоковая отправка чанков; по завершении — получаем **CID** всего файла.
* Включён `pin` — чтобы данные не удалились при GC.

**Проверки и статистика:**

* API по CID может запросить `stat` у Kubo: размер, наличие, пин‑статус.
* Denylist: при попытке загрузить ciphertext с уже известным `checksum` — блокируем на уровне API до регистрации.

**Скачивание (через Gateway):**

* API проверяет право и лимиты → проксирует `GET` на Kubo.
* Поддержка `Range` для возобновляемых скачиваний.
* Заголовки безопасности исключают открытие файла «в браузере».

**Unpin политика:**

* По умолчанию **не** делаем автоматически.
* Админ может вручную инициировать `pin rm` (крайние случаи/запросы).### 4.9 База данных: какие данные и зачем
  **DDL (упрощённо)**

```sql
create table users (
  id uuid primary key,
  eth_address text unique not null,
  rsa_public text not null,
  display_name text,
  created_at timestamptz not null default now()
);

create table files (
  id bytea not null primary key, -- 32 байта fileId
  owner_id uuid not null references users(id),
  name text not null,
  size bigint not null,
  mime text not null,
  cid text not null,
  checksum bytea not null, -- 32 байта
  created_at timestamptz not null default now()
);
create unique index files_checksum_uq on files(checksum);
create index files_owner_idx on files(owner_id, created_at desc);

create table file_versions (
  id bigserial primary key,
  file_id bytea not null references files(id),
  cid text not null,
  checksum bytea not null,
  size bigint not null,
  mime text not null,
  created_at timestamptz not null default now()
);

create table grants (
  id bigserial primary key,
  cap_id bytea unique not null,
  file_id bytea not null references files(id),
  grantor_id uuid not null references users(id),
  grantee_id uuid not null references users(id),
  expires_at timestamptz not null,
  max_dl int not null,
  used int not null default 0,
  revoked_at timestamptz
);
create index grants_user_exp_idx on grants(grantee_id, expires_at);

create table events (
  id bigserial primary key,
  type text not null,
  file_id bytea,
  actor_user_id uuid,
  ts timestamptz not null,
  payload_hash bytea not null,
  period_id bigint
);

create table anchors (
  period_id bigint primary key,
  merkle_root bytea not null,
  tx_hash text not null,
  anchored_ts timestamptz not null
);
```

**Замечания по данным**

* `bytea(32)`: храним бинарно, а не как hex‑строку — экономим место и ускоряем индексы.
* `payload_hash`: хэшируем полезную нагрузку события для приватности и консистентности.
* Идемпотентность: по `request_id` ведём отдельную таблицу `idempotency_keys(request_id, created_at)` (опционально) для быстрой проверки повторов.

**Типовые запросы**

* «Мои файлы»: по `owner_id` с пагинацией по `created_at`.
* «Мои права»: `grants` по `grantee_id`, где `now()<expires_at` и `used<max_dl` и `revoked_at is null`.
* «Журнал файла»: `events` по `file_id` сортировка по `ts`.### 4.10 Прокси/TLS (Caddy)
  **Функции**
* Заворачивает всё в HTTPS (Let’s Encrypt) на `<IP>.sslip.io`.
* Проксирует API и gateway, добавляет security‑заголовки, пишет JSON‑логи (без IP).

**Фрагмент Caddyfile**

```
:443 {
  tls you@domain.tld
  encode gzip

  @api path /api/*
  reverse_proxy @api api:8000

  @gateway path /ipfs/*
  reverse_proxy @gateway ipfs:8080

  header {
    X-Content-Type-Options "nosniff"
    Referrer-Policy "no-referrer"
    X-Frame-Options "DENY"
  }

  log {
    output file /var/log/caddy/access.log
    format json
  }
}
```

**Ошибки и деградации**

* Если API недоступен — отдаём 502 с человеко‑понятным JSON телом.
* Если IPFS не отвечает — 504 и совет «попробовать позже», API параллельно ставит задачу проверки узла IPFS.

### 4.11 Состояние гранта и гонки

**Состояния гранта:** `ACTIVE` → (`REVOKED` | `EXPIRED`) → `CLOSED` (когда `used >= maxDownloads`).

**Переходы:**

* `grant()` создаёт `ACTIVE` (`used=0`).
* `useOnce()` при проверке условий делает `used++`; если `used == maxDownloads` → `CLOSED`.
* `revoke()` ставит `REVOKED` (новые выдачи блокируются).
* В любое время, если `now > expiresAt` → `EXPIRED` (проверяется при обращении).

**Гонки `useOnce`:**

* API ставит оптимистическую задачу `useOnce` (priority=high), но `encK` отдаёт сразу.
* Если два параллельных запроса попытаются превысить `maxDownloads`, контракт пропустит только первые корректные вызовы; остальные получат ошибку, а API синхронизирует UI (покажет, что лимит исчерпан).

**Идемпотентность:**

* Для `useOnce` и других операций везде используем `request_id`. Повтор того же `request_id` безопасен.

### 4.12 PoW: точный алгоритм

**Выдача челленджа:** `challenge = random(16B)`, `difficulty ∈ [16..24]`, `ttl = 30s`.

**Критерий:** найти 32‑битный `nonce`, чтобы `SHA256(challenge || nonce)` имел ≥ `difficulty` ведущих нулевых **бит** (не шестнадцатеричных символов).

**Проверка на API:**

```
H = SHA256(challenge || nonce)
if leadingZeroBits(H) >= difficulty and challenge.notExpired(): OK else 429
```

**Автоподстройка сложности (каждые ~200 успешных решённых):**

* Считаем EWMA среднего времени решения `T_avg` (цель 100–150 мс).
* Если `T_avg < 80ms` → `difficulty = min(difficulty+1, 24)`.
* Если `T_avg > 200ms` → `difficulty = max(difficulty-1, 16)`.
* Для перегретых эндпоинтов (`/share`, `/download`) применяем локальное повышение на +1..+2.

### 4.13 Диаграмма последовательности: Upload (подробно)

1. FE: генерирует `K_file`, `fileId`; режет файл на чанки ~4 МБ.
2. Для каждого чанка: HKDF‑nonce → AES‑GCM → отправка в IPFS.
3. По завершении: считает общий `checksum` и получает `CID` от IPFS.
4. FE → API: `POST /files` (мета) → API возвращает `typedData` для `register`.
5. FE подписывает EIP‑712 → `POST /meta-tx/submit` → API ставит задачу релейеру.
6. Relayer → Chain: `register` → событие `FileRegistered`.
7. FE отображает статус «Зарегистрировано» + CID.

### 4.14 Диаграмма последовательности: Share/Download

**Share:**

1. FE владельца: для каждого получателя шифрует `K_file` в `encK_user` (RSA‑OAEP‑2048).
2. FE → API: `POST /files/{id}/share` → API кладёт `grant` в очередь.
3. Relayer: выполняет `grant` → `Granted(capId, ...)`.

**Download:**

1. FE получателя: запрашивает PoW, решает, вызывает `GET /download/{capId}`.
2. API: проверяет право/лимиты/квоты → `enqueue(useOnce)` (high) → отдаёт `{encK, gatewayUrl}`.
3. FE: расшифровывает `encK` → `K_file` → скачивает через gateway и расшифровывает чанки.
4. Relayer: выполняет `useOnce` → `Used(capId)`; API/FE синхронизируют статус.

### 4.15 Логирование и трассировка (по модулям)

* В каждом запросе генерируем `trace_id` и пробрасываем в логи FE→API→Workers.
* Поля журнала: `ts, level, trace_id, user_id?, file_id_hash, cap_id_hash, action, duration_ms, result`.
* Ошибки всегда содержат внутренний `error_code` и краткое человеко‑понятное сообщение.

---

## 5) Сквозные сценарии (E2E)

### 5.1 Upload (загрузка)

1. Пользователь выбирает файл.
2. Браузер генерирует **`K_file`**, бьёт файл на чанки (~4 МБ), шифрует каждый **AES‑GCM‑256** с уникальным `nonce` (см. §6.2).
3. Параллельно отправляет чанки в **IPFS**, получает **`CID`**.
4. Вычисляет общий `checksum = keccak256(all_ciphertexts_concat)`.
5. Отправляет в API метаданные (`fileId`, name, size, mime, cid, checksum).
6. API формирует **typedData** для `FileRegistry.register` → FE подписывает **EIP‑712** → API отправляет через релейер meta‑tx.
7. Пользователь видит «файл зарегистрирован», CID и статус на он‑чейн.

### 5.2 Share (выдать доступ)

1. Владелец выбирает получателя(ей), TTL и лимит выдач.
2. Браузер шифрует `K_file` публичным ключом получателя **(RSA‑OAEP‑2048/SHA‑256)**, формируя `encK_map`.
3. API формирует/отправляет meta‑tx `AccessControlDFSP.grant`(ы).
4. Смарт‑контракт записывает грант(ы) с `expiresAt`, `maxDownloads`.

### 5.3 Download (скачивание и расшифровка)

1. Получатель заходит по ссылке/в карточку файла, проходит PoW (≈100–150 мс).
2. API проверяет право (`canDownload`), учитывает квоты/PoW, **оптимистически** ставит `useOnce` в приоритетную очередь.
3. API отдаёт `encK` и путь к IPFS (через наш gateway).
4. Браузер расшифровывает `encK` своим приватным RSA‑ключом → получает `K_file` → скачивает и расшифровывает чанки локально.
5. Релейер подтверждает `useOnce` on‑chain; UI синхронизирует «использовано».

### 5.4 Revoke/Expire (отзыв или истечение)

* Новые выдачи `encK` **блокируются** (право отозвано или TTL истёк). Уже выданный ранее `K_file` **технически не отозвать** (см. §6.5), возможна ротация ключа через новую версию файла.

### 5.5 Verify (проверка)

* Пользователь/внешняя сторона вводит `fileId`/CID → сервис сверяет CID и checksum с записью `FileRegistry`.

### 5.6 Anchoring (часовой)

* Воркеры агрегируют события (off‑chain журнал) → строят Merkle‑root → раз в час отправляют `anchorMerkleRoot(root, periodId)` в контракт `Anchoring`.

---

## 6) Криптография (простым языком)

### 6.1 Что шифруем и зачем

* Шифруем **только содержимое файла** до отправки в IPFS.
* Ключ файла (`K_file`) хранится **только у владельца**; получателям он выдаётся **в зашифрованном виде**.

### 6.2 Шифрование файла (по шагам)

* Алгоритм: **AES‑GCM‑256** (современный стандарт «шифрование + контроль целостности»).
* Файл режется на чанки ≈ **4 МБ** (удобно для параллельной отправки и контроля ошибок).
* Для каждого чанка нужен **уникальный** `nonce` (номер/соль). Мы получаем его детерминированно через HKDF:

  * `fileId = keccak256(owner_addr || random_16B || time)` (bytes32)
  * `K_file` — случайный 256‑битный ключ
  * `nonce(chunkIndex) = HKDF(salt=fileId, info=chunkIndex, IKM=K_file)[:12]`
* На выходе каждого чанка: `{ciphertext_i, tag_i}`.
* Для контроля целостности всего файла считаем `checksum = keccak256(all_ciphertexts_concat)` и храним его **on‑chain**.

### 6.3 Доставка ключа `K_file` получателю

* Каждый получатель хранит у себя **пару RSA‑ключей** (созданных в браузере при регистрации).
* Владелец шифрует `K_file` публичным RSA получателя (RSA‑OAEP‑2048/SHA‑256) → **`encK_user`**.
* На скачивании получатель расшифровывает `encK_user` своим приватным RSA → получает `K_file`.

### 6.4 Ключи пользователя и бэкап

* Ключи (RSA и EOA) хранятся **локально** в браузере (IndexedDB), приватные части зашифрованы.
* Бэкап в файл `.dfspkey`: контейнер JSON, KDF **PBKDF2‑SHA‑256** ≥ 210k итераций → AES‑GCM.
* **Блокирующее правило:** *нельзя шарить* (Share), пока пользователь не создал бэкап — защищаем от потери доступа.

### 6.5 Про «отозвать ключ» (важное ограничение)

* Если у получателя уже есть `K_file`, запретить ему читать старую копию **невозможно** — криптография так не работает.
* Наш «Revoke» означает **«перестать выдавать `encK` новым запросам»** и (опционально) **unpin** старой версии в IPFS (админ‑ручка).

---

## 7) Смарт‑контракты

### 7.1 Общие принципы

* MVP — без прокси/апгрейдов. Миграции = новые адреса.
* Используем **ERC‑2771 MinimalForwarder (OpenZeppelin)**; в контрактах используем корректный `_msgSender()`.
* Он‑чейн allowlist релейеров **не нужен** (подписи пользователя защищают).

### 7.2 `FileRegistry`

**Состояние**:

* `mapping(bytes32 fileId => FileMeta{owner, cid(string), checksum(bytes32), size, mime, createdAt})`
* `mapping(fileId => Version[])`

**Методы**:

* `register(fileId, cid, checksum, size, mime)`
* `updateCid(fileId, newCid, newChecksum, newSize, newMime)`
* `metaOf(fileId)` / `ownerOf(fileId)`

**События**:

* `FileRegistered(fileId, owner, cid, checksum)`
* `FileVersioned(fileId, newCid, newChecksum)`

### 7.3 `AccessControlDFSP`

**Состояние**:

* `mapping(bytes32 capId => Grant{grantor, grantee, fileId, expiresAt, maxDownloads, used, revokedAt?})`

**Методы**:

* `grant(fileId, grantee, ttlSec, maxDownloads) returns (capId)`
* `revoke(capId)`
* `canDownload(user, fileId)`
* `useOnce(capId)`
* (Крючок на будущее) `grantBearer(fileId, ttlSec, maxDownloads)`

**События**:

* `Granted(capId, fileId, grantee, expiresAt, maxDownloads)`
* `Revoked(capId)`
* `Used(capId)` (опционально)

**Семантика**:

* `TTL` и `maxDownloads` проверяются до выдачи `encK`.
* `useOnce` учитывает фактическое количество выдач `encK`.

### 7.4 `Anchoring`

* `anchorMerkleRoot(bytes32 root, uint64 periodId)` только owner.
* Событие `Anchored(root, periodId)`.

### 7.5 ERC‑2771 / MinimalForwarder

* FE подписывает **EIP‑712** typedData (см. §12.A), BE валидирует дедлайн **off‑chain** до отправки.

---

## 8) Backend API (FastAPI)

### 8.1 Аутентификация (без email; key‑only)

* `POST /auth/challenge` → `{nonce, exp}`
* `POST /auth/register` → `{rsa_public (SPKI PEM), eth_address, display_name?, challenge, signature}` → `{access, refresh}`
* `POST /auth/login` → `{eth_address, challenge, signature}` → `{access, refresh}`
* `POST /auth/refresh` → `{access}`

### 8.2 Файлы

* `POST /files` → `{fileId, name, size, mime, cid, checksum}` → `{typedData}`
* `POST /meta-tx/submit` → `{signedForwardRequest}` → `{txHash, status}`
* `GET /files?mine=true` → список метаданных
* `GET /files/{id}` → `{meta, versions, grants}`
* `POST /files/{id}/version` → typedData для `updateCid`

### 8.3 Шаринг/скачивание

* `POST /files/{id}/share` → `{users:[...], ttl_days, max_dl, encK_map:{userId:base64}}` → meta‑tx `grant`
* `POST /grants/{capId}/revoke` → meta‑tx `revoke`
* `GET /grants/{capId}` → состояние гранта
* `GET /download/{capId}` → проверка права → PoW/квоты → **оптимистический** `useOnce` в приоритетную очередь → `{encK, ipfsPath}`

### 8.4 Verify/Anchors

* `GET /verify/{fileId}` → сверка с он‑чейн
* `GET /anchors/latest` → `{period_id, merkle_root, tx_hash}`

### 8.5 Ошибки/коды

* 400/401/403/404/409/429/500/502

### 8.6 Безопасность API и gateway

* Заголовки: `X-Content-Type-Options: nosniff`, строгий `Content-Disposition: attachment; filename*=UTF-8''...`
* **Byte‑range** поддерживается, `inline` выключаем для потенциально опасных типов.
* CORS минимально необходимый (только наш фронтенд).

---

## 9) Relayer & Workers

* Очереди Celery: `relayer.high` (useOnce, revoke), `relayer.default` (register, grant, version), `anchor`.
* Идемпотентность: `request_id` (UUIDv4); повторный запрос → no‑op.
* Ретраи: экспоненциальные, до 5 попыток (ошибки сети/nonce/gasprice).
* Мониторинг: p95 подтверждений, процент удач.

---

## 10) База данных (Postgres)

**Таблицы**

* `users(id UUID pk, eth_address text uniq, rsa_public text, display_name text, created_at timestamptz)`
* `files(id bytea(32) pk, owner_id UUID, name text, size bigint, mime text, cid text, checksum bytea(32), created_at timestamptz)`
* `file_versions(id bigserial pk, file_id bytea(32), cid text, checksum bytea(32), size bigint, mime text, created_at timestamptz)`
* `grants(id bigserial pk, cap_id bytea(32) uniq, file_id bytea(32), grantor_id UUID, grantee_id UUID, expires_at timestamptz, max_dl int, used int, revoked_at timestamptz)`
* `events(id bigserial pk, type text, file_id bytea(32), actor_user_id UUID, ts timestamptz, payload_hash bytea(32), period_id bigint)`
* `anchors(period_id bigint pk, merkle_root bytea(32), tx_hash text, anchored_ts timestamptz)`

**Индексы**

* `files(owner_id, created_at)`, `grants(grantee_id, expires_at)`, `events(file_id, ts)`, `files(checksum) UNIQUE` (denylist).

---

## 11) IPFS и Gateway

### 11.1 Kubo конфиг (фрагмент)

```json
{
  "Routing": { "Type": "dht" },
  "Addresses": { "API": "/ip4/0.0.0.0/tcp/5001", "Gateway": "/ip4/0.0.0.0/tcp/8080" },
  "Datastore": { "GCPeriod": "24h", "StorageGCWatermark": 90 }
}
```

### 11.2 Политики

* Пин включён, GC раз в сутки.
* Gateway только через наш API‑роут (проверка прав → проксирование).
* Unpin по умолчанию **не делаем**; доступна админ‑ручка.

---

## 12) Форматы и примеры

### A) EIP‑712 typedData (пример)

```json
{
  "domain": {"name":"MinimalForwarder","version":"0.0.1","chainId":1337,"verifyingContract":"0x..."},
  "types": {"ForwardRequest":[
    {"name":"from","type":"address"},
    {"name":"to","type":"address"},
    {"name":"value","type":"uint256"},
    {"name":"gas","type":"uint256"},
    {"name":"nonce","type":"uint256"},
    {"name":"data","type":"bytes"}
  ]},
  "primaryType":"ForwardRequest",
  "message":{"from":"0x...","to":"0x...","value":"0","gas":"100000","nonce":"1","data":"0x..."}
}
```

### B) PoW‑челендж (пример)

```json
{"challenge":"base64(16B)","difficulty":18,"ttl":30}
```

Клиент находит `nonce` (32‑бит), чтобы `SHA256(challenge||nonce)` имел ≥18 ведущих нулевых бит (цель ~100–150 мс).

---

## 17) Фронтенд (React + Vite + TS)

* **Страницы:** Register/Login; Files; Upload; File Details; Share; Download; Verify; Settings (keys backup).
* **Состояния:** IndexedDB (`keys`, `backups`, `preferences`), Zustand для сессии/списков/PoW.
* **Потоки:** Upload → шифрование → IPFS → `/files` → подпись → meta‑tx; Share → `encK_map` → grant; Download → `encK` → расшифровка.
* **Браузеры MVP:** Desktop Chrome/Firefox/Edge (Safari — после MVP). Мобилки — после MVP.

---

## 18) Процессы разработки и DevOps

### 18.1 Окружения

* `local` (разраб), `mvp` (единый стенд для демо/приёмки). Stage пока **не делаем**.

### 18.2 Деплой (Docker Compose)

Сервисы: `proxy(caddy)`, `api`, `db(postgres)`, `redis`, `ipfs(kubo)`, `chain(hardhat)`, `relayer`, `anchor`, `frontend`.

### 18.3 CI/CD (GitHub Actions)

* Lint: Python (ruff/black/isort), TS (eslint), Solidity (solhint).
* Tests: unit + contracts + API + E2E (Playwright headless).
* Security: `bandit`, `slither`, SCA (Dependabot), контейнер‑скан (Trivy).
* CD: build образов → push GHCR → деплой по SSH/compose.
* Branching: trunk‑based, feature branches + PR, CODEOWNERS, required checks.

### 18.4 Конфиги/секреты

* `.env.example` (см. Приложение D);
* Секреты — в GitHub Encrypted Secrets; доступ по принципу наименьших привилегий.

---
