# DFSP — Frontend: Реализационная спецификация (v1.2)
---

## 0) Границы и цели

* Реализуем **SPA на React + TypeScript + Vite**.
* Браузер выполняет: генерацию ключей, шифрование/расшифровку, подписи EIP‑712, PoW, загрузку/скачивание через IPFS/gateway.
* Сервер не получает приватные ключи и открытые файлы.
* MVP‑функции: Register/Login (PoP), Upload, Share (per‑recipient), Download (useOnce), Verify, Settings (backup).

**Критические SLO (в UI):**

* p95 «init download» < 800 мс (получить encK и старт стрима)
* p95 «UI action latency» < 100 мс (видимый отклик кнопок)
* Initial JS ≤ ~300 КБ gzip

---

## 1) Технологический стек и версии

* **React 18.2**, **TypeScript ≥ 5.3**, **Vite ≥ 5**
* **Zustand ≥ 4** (state), **React Router ≥ 6.23**
* **Tailwind CSS ≥ 3.4**, **shadcn/ui** (Button, Card, Input, Dialog, Tooltip, Progress)
* **lucide-react** (иконки), **sonner** (тосты)
* **ethers v6** (keccak256, EIP‑712), **noble-secp256k1** (fallback)
* **WebCrypto (SubtleCrypto)**: AES‑GCM, HKDF, RSA‑OAEP
* **ipfs-http-client** (опц.) или `fetch` к Kubo `/api/v0/add`
* Тесты: **Vitest**, **msw**, **Playwright**

**package.json (фрагмент):**

```json
{
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "preview": "vite preview",
    "test": "vitest -u",
    "e2e": "playwright test"
  }
}
```

---

## 2) Структура проекта (жёстко зафиксированная)

```
src/
  app/                # корневой layout, маршрутизатор, error boundaries
  pages/              # экраны (Files, Upload, Details, Share, Download, Verify, Settings, Auth)
  components/         # UI-компоненты (Card, Button, Field, FileItem, GrantItem, Copyable, QR?, etc.)
  state/              # Zustand slices + selectors
  api/                # API-клиент (fetch-обёртка, SDK-функции)
  crypto/             # обёртки WebCrypto: aes.ts, hkdf.ts, rsa.ts, keccak.ts
  workers/            # crypto.worker.ts, pow.worker.ts, upload.worker.ts
  ipfs/               # uploader.ts (Kubo /api/v0/add), progress.ts
  utils/              # валидации, форматтеры, base64, bytes
  hooks/              # usePoW, useUploader, useDownloader, useKeys
  styles/             # tailwind.css, tokens.css
  config/             # env.ts, featureFlags.ts
```

---

## 3) Типы и контракты данных (TS)

```ts
// Ключи/аккаунт
export type User = { id: string; addr: `0x${string}`; name?: string }
export type KeyBackupMeta = { created: string; kdf: string; cipher: string }

// Файлы/гранты
export type FileMeta = { id: string; name: string; size: number; mime: string; cid: string; checksum: string; createdAt: string }
export type Grant = { capId: string; fileId: string; granteeId: string; expiresAt: string; maxDownloads: number; used: number; revokedAt?: string | null }

// Download init
export type DownloadInit = { encK: string; gatewayUrl: string }

// Ошибка API
export class ApiError extends Error {
  constructor(readonly code: string, readonly http: number, readonly traceId?: string) { super(code) }
}
```

---

## 4) Key Store (реализация)

**Хранилище:** IndexedDB через `idb-keyval`.

**Слоты:**

* `rsa_priv_encrypted` (Uint8Array)
* `rsa_pub_spki_pem` (string)
* `eoa_priv_encrypted` (Uint8Array)
* `eoa_addr` (string)
* `meta` (KeyBackupMeta)

**Шифрование приватных ключей:**

```ts
async function deriveKEK(pass: string, salt: Uint8Array): Promise<CryptoKey> {
  const pbkdf = await crypto.subtle.importKey('raw', new TextEncoder().encode(pass), 'PBKDF2', false, ['deriveKey'])
  return crypto.subtle.deriveKey({ name: 'PBKDF2', salt, iterations: 210_000, hash: 'SHA-256' }, pbkdf, { name:'AES-GCM', length:256 }, false, ['encrypt','decrypt'])
}
```

**API Key Store:**

```ts
export const Keys = {
  create(pass: string): Promise<void>,
  unlock(pass: string): Promise<{rsaPriv: CryptoKey, eoaPrivRaw32: Uint8Array}>,
  lock(): void,
  exportBackup(): Promise<Blob>,
  importBackup(file: File, pass: string): Promise<void>,
  hasBackup(): Promise<boolean>
}
```

**UX‑правило:** пока `hasBackup()==false` — кнопка **Share** disabled, показываем Tooltip «Сначала сохраните .dfspkey».

---

## 5) Crypto Manager (реализация)

**Чанк = 4 МБ** (константа `CHUNK_BYTES = 4 * 1024 * 1024`).

**Генерация идентификаторов:**

```ts
function genFileId(ownerAddr: string): Uint8Array {
  const rnd = crypto.getRandomValues(new Uint8Array(16))
  const ts = new DataView(new ArrayBuffer(8)); ts.setBigUint64(0, BigInt(Date.now()), true)
  return keccak256Bytes(concatBytes(addrToBytes(ownerAddr), rnd, new Uint8Array(ts.buffer))) // Uint8Array(32)
}
```

**HKDF‑nonce:** `nonce_i = HKDF_SHA256(IKM=K_file, salt=fileId, info=LE32(i))[:12]`

**Интерфейс шифрования:**

```ts
export type EncryptResult = { cid: string; checksumB64: string; fileIdB64: string }
export async function encryptAndUpload(file: File, ownerAddr: string): Promise<EncryptResult>
```

**Последовательность в `encryptAndUpload`:**

1. `K_file = random(32)`; `fileId = genFileId(ownerAddr)`; `checksum = Keccak256()`.
2. Читаем `file.stream()` → `upload.worker` режет на чанки и шифрует через `crypto.worker`.
3. Каждый зашифрованный блок отправляем в IPFS `/api/v0/add` (multiple part) → отмечаем прогресс, обновляем checksum (только по ciphertext частям).
4. Получаем `CID` и считаем `checksum_final`.
5. Возвращаем `{cid, checksumB64, fileIdB64}`.

**Доставка ключа:**

```ts
export async function makeEncKForRecipients(K_file: Uint8Array, recipients: {id:string, rsaPem:string}[]): Promise<Record<string,string>>
```

возвращает `encK_map: { granteeId: base64 }`.

---

## 6) Uploader (IPFS `/api/v0/add`)

**Прямая загрузка (предпочтительно):**

```ts
const res = await fetch(`${VITE_IPFS_API}/api/v0/add?pin=true&wrap-with-directory=false`, { method:'POST', body: formDataStream })
// читаем NDJSON-ответ, последний объект содержит { Hash: <CID> }
```

**CORS**: в ответах Kubo должны быть `Access-Control-Allow-Origin: <FE-origin>`.

**Конкурентность и ретраи:**

* Пул из 3–4 параллельных частей; каждая часть: «зашифровать→отправить→очистить из памяти».
* Повтор: на `TypeError`/сетевые/5xx — до 3 раз, backoff 500→1000→2000 мс.

**Прогресс:** эмитим `onProgress(loadedBytes/totalBytes)` из `upload.worker`.

**Альтернатива через Backend**: если прямой `/add` недоступен — отправляем зашифрованные блоки на API `POST /upload/chunk` (MVP — избегаем из‑за нагрузки на API).

---

## 7) Download (инициация и поток)

**Инициация:**

```ts
const { encK, gatewayUrl } = await api.download.init(capId) // GET /download/{capId}
const K_file = await RSA.decrypt(encK) // WebCrypto RSA-OAEP(SHA-256)
```

**Поток:**

* Делаем `fetch(gatewayUrl, { headers: { Range: 'bytes=0-' }})` и читаем ReadableStream.
* На каждый полученный кусок → дешифр чанка через `crypto.worker` (IV по HKDF) → записываем в WritableStream к `FileSystemWritableFileStream` или собираем в `Blob`.
* Параллельные диапазоны: можно 2–3 одновременных Range‑запроса (ускорение на быстрых сетях).

**Верификация:** после завершения считаем checksum и сверяем с `/verify/{fileId}`.

---

## 8) PoW Solver (worker)

**Контракт:**

```ts
// main thread
const res = await powWorker.solve({ challenge, difficulty, ttlMs })
// worker API
onmessage = async ({data}) => { const {challenge, difficulty} = data; postMessage({ nonce: search(challenge,difficulty) }) }
```

**Оптимизации:**

* Куски по 65 536 итераций; между батчами `postMessage({progress})` и `if (Date.now()>deadline) cancel`.
* Использовать встроенный `crypto.subtle.digest('SHA-256', ...)` или wasm‑sha256.

---

## 9) API SDK (строго типизированный)

```ts
export const api = {
  auth: {
    challenge(): Promise<{nonce: string, exp: string}>
    register(body: { rsa_public_spki_pem: string; eth_address: string; display_name?: string; challenge: any; signature: string }): Promise<{access:string,refresh:string}>
    login(body: { eth_address: string; challenge: any; signature: string }): Promise<{access:string,refresh:string}>
  },
  files: {
    create(meta: { fileId: string; name: string; size: number; mime: string; cid: string; checksum: string }): Promise<{typedData:any, request_id:string}>
    submit(signed: any): Promise<{txHash:string, status:string}>
    mine(): Promise<FileMeta[]>
    get(id: string): Promise<{meta: FileMeta, versions:any[], grants: Grant[]}>
  },
  share: {
    create(fileId: string, body: { users: string[]; ttl_days: number; max_dl: number; encK_map: Record<string,string> }): Promise<{capIds:string[], txHash:string}>
  },
  download: {
    init(capId: string): Promise<DownloadInit>
  },
  verify: {
    byFileId(fileId: string): Promise<{ onchain:{cid:string, checksum:string}, status:'match'|'mismatch' }>
  }
}
```

**Единая обработка ошибок:**

```ts
async function toApiError(r: Response): Promise<ApiError> { const j = await r.json().catch(()=>({})) ; return new ApiError(j.code||'INTERNAL', r.status, j.trace_id) }
```

---

## 10) Zustand (слайсы, интерфейсы, действия)

```ts
// session.ts
type SessionState = { access?: string; refresh?: string; me?: User; unlocked: boolean }
const useSession = create<SessionState & { setTokens(a?:string,r?:string):void; setMe(u?:User):void; setUnlocked(b:boolean):void }>(...)

// files.ts
type FilesState = { list: FileMeta[]; byId: Record<string, { meta: FileMeta; grants: Grant[] }> }
const useFiles = create<FilesState & { loadMine():Promise<void>; get(id:string):Promise<void> }>(...)

// pow.ts
type PoWState = { challenge?: { c: string; d: number; exp: string }; nonce?: number }
const usePoW = create<PoWState & { fetchChallenge(ep: string):Promise<void>; solve():Promise<number> }>(...)
```

**Правила:** селекторы узкие (`useStore(s=>s.list)`), чтобы не перерисовывать весь UI.

---

## 11) Страницы (требования и сценарии)

### 11.1 Auth (Login/Register)

* Кнопка «Create account»: генерим EOA+RSA, спрашиваем пароль, сохраняем зашифрованные ключи, предлагаем сразу скачать `.dfspkey`.
* `POST /auth/challenge` → подписываем EIP‑712 → `POST /auth/register` → сохраняем `access/refresh` в Zustand.
* Повторный вход (login) — без генерации ключей: читаем из IndexedDB, разблокируем паролем.

**Ошибки:** 401 (неверная подпись) → уведомление; 429 (много попыток) → таймер.

### 11.2 Files

* Таблица/карточки: имя, размер, дата, CID, кнопки [Details] [Share] [Verify].
* Кнопка [Upload] ведёт на страницу загрузки.

### 11.3 Upload

* dnd‑зона, выбор файла, вывод лимитов.
* При старте: прогресс‑бар, счётчик МБ/сек.
* По завершении: `POST /files` → `typedData` → подпись → `/meta-tx/submit` → статус «Registered».

**Ошибки:** >200 МБ → дизаблим кнопку; 5xx → повтор. Прерывание → «Продолжить позже» (если реализуем резюмирование на уровне `/add`).

### 11.4 Details

* Отображает meta (CID, checksum, size, mime), список версий, выданные гранты.

### 11.5 Share

* Автодополнение пользователей; поля TTL (дни), `max_dl` (шт.).
* Кнопка [Share] активна только если есть бэкап.
* Шифруем `K_file` в `encK_map`, отправляем `/files/{id}/share`, показываем capIds.

### 11.6 Download

* При заходе по capId: сразу запускаем PoW, затем `GET /download/{capId}` → скачивание/расшифровка.
* Показываем «init» время (цель — <800 мс), прогресс скачивания.

### 11.7 Verify

* Поле `fileId`/CID → запрос `/verify/{fileId}` → бейдж `match/mismatch`.

### 11.8 Settings (Keys backup)

* Кнопка «Скачать .dfspkey»; кнопка «Импорт бэкапа» (drag&drop файла → валидация формата, пароль → восстановление).

---

## 12) Безопасность (FE‑правила)

* Ключи только в IndexedDB, всегда под AES‑GCM (KEK из PBKDF2 ≥ 210k итераций).
* После разблокировки держим KEK/приватники только в памяти; по idle 15 мин → авто‑lock.
* CSP без `unsafe-inline`; X‑Frame‑Options: DENY; Referrer‑Policy: no-referrer (конфиг прокси/заголовки — со стороны backend/proxy).
* Не использовать `dangerouslySetInnerHTML`.
* Использовать `crypto.getRandomValues` только для случайностей.

---

## 13) Производительность (конкретные нормы)

* Web Workers: crypto/pow/upload — всегда, чтобы главный поток был свободен.
* Одновременно не держать > 16 МБ в памяти (чанк + параллельные операции).
* Concurrency: upload 3–4, download 2–3 параллельных диапазона.
* Code‑splitting по страницам, импорт тяжёлых модулей лениво.

**Мониторинг (в Dev):** логируем длительность шифрования 1 чанка и скорость загрузки.

---

## 14) Телеметрия (без PII)

События (отправка в backend или локальный лог Dev):

```json
{"t":"upload_start","fileSize":10485760}
{"t":"upload_chunk","i":3,"dt_ms":42}
{"t":"upload_done","cid":"bafy...","sec":5.3}
{"t":"download_init","capId":"0x..","ms":120}
{"t":"download_done","bytes":26214400,"sec":5.3}
```

`user_id` не передаём; `trace_id` можно добавлять для саппорта.

---

## 15) Тестирование (чек‑листы)

**Unit (Vitest):**

* HKDF: стабильные IV на одинаковых входах; разные IV на разных индексах.
* AES‑GCM: encrypt→decrypt возвращает исходные байты.
* RSA‑OAEP: encrypt→decrypt для 32‑байтового K_file.
* EIP‑712: подпись восстанавливает адрес владельца.
* PoW: корректный подсчёт ведущих нулевых бит; остановка по ttl.

**Integration (msw):**

* Upload→/files→/meta‑tx/submit (мокаем CID и typedData).
* Download: выдаём `encK`/`gatewayUrl`, читаем диапазонами, проверяем сборку Blob.

**E2E (Playwright):**

* Полный сценарий Upload→Share→Download→Revoke→Verify.
* Негативные: истёк PoW; превышена квота; лимит использований; файл > 200 МБ.

---

## 16) Реализационные заметки (важные мелочи)

* **Checksum** считаем **только по ciphertext** (plain‑данных в памяти не держим после шифрования).
* **encK_map** хранится на сервере, но это уже шифртекст RSA‑OAEP — на FE дополнительных шифров не нужно.
* При ошибках сети **не дублируем** уже отправленные части — ведём локальный реестр успешных чанков.
* На странице Download показываем «готовим файл…», как только получили первые байты — пишем «скачивание началось».

---

## 17) Feature‑flags (JSON формат)

```json
{
  "bearerLinks": false,
  "groups": false,
  "hpke": false,
  "pwa": false,
  "darkTheme": false
}
```

Флаги получаем из `/config` backend или из `VITE_FLAGS` при сборке.

---

## 18) Псевдокод ключевых функций

**Шифрование + загрузка:**

```ts
export async function encryptAndUpload(file: File, owner: string): Promise<EncryptResult> {
  const K = rand32(); const fileId = genFileId(owner); const keccak = new Keccak()
  const stream = file.stream().getReader(); const parts: Uint8Array[] = []
  let i=0; for await (const chunk of iterate(stream, CHUNK_BYTES)) {
    const iv = hkdfIv(K, fileId, i++)
    const ct = await aesGcmEncrypt(K, iv, chunk)
    keccak.update(ct)
    parts.push(ct)
    postProgress(...)
  }
  const cid = await ipfsAdd(parts)
  return { cid, checksumB64: b64(keccak.final()), fileIdB64: b64(fileId) }
}
```

**PoW:**

```ts
function search(challenge: Uint8Array, d: number): number { for (let n=0; n<2**32; n++){ if (lz(sha256(challenge, u32be(n)))>=d) return n } return -1 }
```

---

## 19) Сборка и конфиг (Vite)

* `VITE_API_BASE`, `VITE_IPFS_API`, `VITE_BUILD_VERSION`, `VITE_FLAGS`.
* Source‑map включаем только на staging/dev.
* PWA выключена на MVP.

---

## 20) DoR / DoD (Frontend)

**DoR:**

* API форматы подтверждены; CORS на Kubo настроен; макеты экранов согласованы.

**DoD:**

* Все страницы покрыты E2E; перфоманс бюджет соблюдён; безопасность ключей подтверждена; демонстрационный сценарий стабильно воспроизводим.

---

*Конец спецификации.*
