# 冷錢包 Open API 串接指南（DigiQuick）

- 版本：v0.6（草稿）
- 日期：2026-11-22
- 對應規格：[`SPEC.md`](SPEC.md) v0.6

> **狀態：規劃中，API 還沒有實作。** 欄位在實作過程中可能微調，有改會更新版本號。
> 串過熱錢包的話：簽章方式幾乎一樣（第 3 節標出差異），webhook 驗證方式一樣。

## 修訂紀錄

| 版本 | 日期 | 內容 |
|---|---|---|
| v0.1 | 2026-11-20 | 初版 |
| v0.2 | 2026-11-20 | 每個發行方、每位投資人各是一個客戶；路徑 `/v1/clients/{clientId}/…` |
| v0.3 | 2026-11-22 | 發行方也由 DigiQuick 用 API 申請；金鑰可以由 Aegis 產生或由你們提供公鑰；發行方可以把投資人自己的外部地址當 CMTAT 對象（allowlist、銷毀）；錯誤回應加 `retryable`；webhook 改「至少送一次」；寫清楚同時送出時可能的結果 |
| v0.4 | 2026-11-22 | webhook 改成「送達可能亂序，用 seq 判斷新舊」；`TIMEOUT` 不再是最終狀態；寫清楚同時送出時本金、鎖定與 gas 的結果；KYT 沒有評分時回 502；持有人每頁 25 筆 |
| v0.5 | 2026-11-22 | 結果不明的請求改由 Aegis 處理後標記失敗（`REQUEST_ABANDONED`），之後用新的 referenceId 重送；同時送出的結果改成依「合計是否超過可用量」說明；webhook 接收範例把去重與 seq 比較放在同一個交易 |
| v0.6 | 2026-11-23 | 結果不明的請求處理後只會有一個最終狀態（`REJECTED` 或 `FAILED／REQUEST_ABANDONED`）；同時送出的結果改成「其他執行條件成立時可以成功」；webhook 接收範例改成實際可執行的寫法；審核進度（`review.aegisStage`）用查詢取得，不另發 webhook |


## 1. 冷錢包 Open API 提供什麼

### 1.1 主體怎麼對應

| DigiQuick 的主體 | 冷錢包裡 |
|---|---|
| DigiQuick 平台 | 一個 **partner**，一組 API 憑證，可以代表底下所有已連結的客戶 |
| 每個發行方公司 | 一個**發行方客戶**（company） |
| 每位投資人 | 一個**投資人客戶**（individual） |

- 發行方和投資人客戶都由 DigiQuick 用 API 申請，Aegis 核准後建立。Aegis 也可以替 DigiQuick 設成自動核准，API 與結果相同，只是不用等人；但新錢包仍然要 Aegis 核准。
- 這些客戶都只透過 API 操作，沒有可以登入冷錢包前台的使用者。
- 呼叫時在路徑指定客戶：`/v1/clients/{clientId}/…`。申請時帶的 `externalId`（你們自己的 ID）會在查詢與 webhook 中帶回。
- 發行方可以操作投資人的託管錢包與投資人自己的外部地址（例如發行代幣給投資人、銷毀投資人的代幣）；發行方之間、投資人之間不能互相操作。

### 1.2 能做的事

| 類別 | 能做的事 |
|---|---|
| 客戶 | 申請發行方與投資人客戶、查詢已連結的客戶 |
| 錢包 | 建立託管錢包、登記投資人自己的外部地址、把外部地址加進託管錢包的提領白名單 |
| CMTAT | 上傳條款、部署、指派角色、allowlist、發行、銷毀、burn and mint、修改合約資訊 |
| 提領 | 投資人把代幣提領到自己的外部地址 |
| 查詢與通知 | 錢包與餘額、合約與持有人鏈上狀態、請求進度；webhook |

### 1.3 跟熱錢包最大的不同

- **每一筆錢包與代幣操作都要 Aegis 人工審核**才會執行，API 只負責發起。客戶申請則依 Aegis 替 DigiQuick 設定的模式：人工核准或自動核准。
- **寫入是非同步的**：建立成功回 `202`，結果等 webhook 或查詢；正式環境是離線簽章，一筆可能要好幾個小時。
- 暫停、凍結、強制轉移、停用合約只能由 Aegis 在後台操作，發生在你們的代幣上時會收到 webhook。
- 新錢包一律要 Aegis 核准；客戶申請在人工核准模式下也要等 Aegis。投資人多時要預留作業時間。

## 2. 接入前準備

| # | DigiQuick 準備 | Aegis 提供 |
|---|---|---|
| 1 | 二選一：(A) 自己產生 RSA 金鑰，給 Aegis **公鑰**；(B) 請 Aegis 產生，Aegis 會給你們一次私鑰檔 | API key（以及 B 的私鑰檔） |
| 2 | 後端固定對外 IP | — |
| 3 | 接收 webhook 的 HTTPS URL | webhook secret（只給一次） |
| 4 | 需要的權限（3.4） | 各環境 base URL |

```bash
# 選 A 時
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:3072 -out dq-private.pem
openssl pkey -in dq-private.pem -pubout -out dq-public.pem   # 把 dq-public.pem 給 Aegis
```

- 建議選 A，私鑰不經過 Aegis。選 B 時 Aegis 不保存私鑰，遺失只能重新核發。
- 憑證需要 Aegis 兩位主管（建立與核准）才生效；有到期日（最長一年）；同時最多 3 組。換金鑰時先切換到新的，再請 Aegis 撤銷舊的。

## 3. 請求認證

### 3.1 Header

| Header | 內容 |
|---|---|
| `X-Aegis-Api-Key` | API key（`ak_` 開頭） |
| `X-Aegis-Api-Timestamp` | Unix 秒，跟伺服器相差不超過 5 分鐘 |
| `X-Aegis-Api-Nonce` | 每次不同，16–64 字元，英數字、`-`、`_` |
| `X-Aegis-Api-Signature` | 簽章 |
| `Content-Type` | 有 body 時 `application/json` |

### 3.2 簽章

1. 串接：`apiKey + timestamp + nonce + pathWithQuery + method + body`（`pathWithQuery` 從 `/v1` 開始、**含 query string**；`method` 大寫；`body` 是送出的原文，GET 為空字串）。
2. RSA-PSS（SHA-256、MGF1 SHA-256、salt length 32）。
3. 標準 Base64。

**跟熱錢包唯一的差異：path 要包含 query string。**

### 3.3 範例（Node.js）

```ts
import crypto from 'crypto'

const API_KEY = process.env.AEGIS_VAULT_API_KEY!
const PRIVATE_KEY = process.env.AEGIS_VAULT_PRIVATE_KEY_PEM!
const BASE_URL = process.env.AEGIS_VAULT_API_BASE_URL!

export async function callVault(method: 'GET' | 'POST', pathWithQuery: string, payload?: unknown) {
  const body = payload === undefined ? '' : JSON.stringify(payload) // 簽的字串就是送出的字串
  const timestamp = Math.floor(Date.now() / 1000).toString()
  const nonce = crypto.randomUUID()
  const message = API_KEY + timestamp + nonce + pathWithQuery + method + body
  const signature = crypto
    .sign('sha256', Buffer.from(message, 'utf8'), {
      key: PRIVATE_KEY,
      padding: crypto.constants.RSA_PKCS1_PSS_PADDING,
      saltLength: 32,
    })
    .toString('base64')

  const res = await fetch(BASE_URL + pathWithQuery, {
    method,
    headers: {
      'X-Aegis-Api-Key': API_KEY,
      'X-Aegis-Api-Timestamp': timestamp,
      'X-Aegis-Api-Nonce': nonce,
      'X-Aegis-Api-Signature': signature,
      ...(body ? { 'Content-Type': 'application/json' } : {}),
    },
    body: body || undefined,
  })
  return { status: res.status, json: await res.json() }
}
```

- 重試時換新的 nonce 和 timestamp，`referenceId` 不變。

### 3.4 權限（scope）

| scope | 可以呼叫 |
|---|---|
| `READ` | 所有查詢 |
| `CLIENT_WRITE` | 申請客戶 |
| `WALLET_WRITE` | 建錢包、登記外部地址、加入提領白名單 |
| `CMTAT_WRITE` | 條款、部署、合約操作 |
| `WITHDRAWAL_WRITE` | 提領 |
| `WEBHOOK_MANAGE` | webhook 補查、重送 |

### 3.5 認證失敗

| HTTP | code | 原因 |
|---|---|---|
| 401 | `UNAUTHORIZED` | API key、憑證狀態、IP、時間差、簽章、nonce 重複（不說明是哪一項） |
| 403 | `SCOPE_NOT_ALLOWED` | 沒有權限 |
| 404 | `NOT_FOUND` | 客戶沒有連結到你們、已停用，或資料不屬於該客戶 |
| 429 | `RATE_LIMITED` | 每分鐘超過 120 次，依 `Retry-After` 等待 |

- 客戶被 Aegis 停用後，對該客戶的操作回 404；但已送出的請求仍可用 `/v1/requests` 查詢，webhook 照送。

## 4. 回應格式

```json
{ "success": true, "data": { } }
{ "success": true, "data": { "items": [ ], "nextCursor": "eyJ…" } }
{ "success": false, "data": { "code": "CMTAT_TARGET_NOT_ALLOWLISTED", "message": "target is not allowlisted", "referenceId": "dq-usdsn27-mint-0001", "retryable": false } }
```

- **`retryable`**：`true` 表示可以用**同一個** `referenceId` 稍後重試；`false` 表示不要原樣重試（修正後用新的 referenceId，或聯絡 Aegis）。
- 列表用 cursor（`limit` 1–100）。每個回應都有 `X-Request-Id`。
- 數量一律是字串、代幣單位、大於 0。
- `cmtatTokenId` 是十進位數字字串（例如 `"100000"`），跟合約上的 `tokenId`（ISIN 等）不同。

## 5. 寫入請求的狀態

```
PENDING_AEGIS_REVIEW ──► PROCESSING ──► COMPLETED
        │                    ├────────► FAILED
        │                    └────────► TIMEOUT
        └──────────────────────────────► REJECTED
```

| status | 意義 |
|---|---|
| `PENDING_AEGIS_REVIEW` | 等 Aegis 審核；`review.aegisStage` 顯示卡在哪一關（關卡推進時查詢結果會更新，但不會另外發 webhook） |
| `PROCESSING` | 已核准，簽章、上鏈、記帳中。CMTAT 操作要等冷錢包的帳也記好才會變 `COMPLETED` |
| `TIMEOUT` | 審核或簽章逾時。**不是最終狀態**：之後仍可能變成 `COMPLETED` 或 `FAILED`（`isFinal` 為 false） |
| `COMPLETED` | 完成，`result` 有結果；客戶申請完成代表可以開始建錢包 |
| `REJECTED`、`FAILED` | 被拒、失敗 |

- 最終狀態只有 `COMPLETED`、`REJECTED`、`FAILED`，彼此互斥。狀態只會往前推進：一筆請求變成 `COMPLETED` 之後不會再變回 `TIMEOUT` 或其他狀態。
- **收到 `TIMEOUT` 不要自動用新的 referenceId 重送同一筆業務**（例如 mint），交易仍可能完成；請持續查詢或聯絡 Aegis。

RequestView（`202`、查詢、webhook 共用）：

```json
{
  "referenceId": "dq-usdsn27-mint-0001",
  "requestId": "8d1f0a8e-…",
  "clientId": "3c1e…",
  "clientExternalId": "issuer-C001",
  "operationType": "CMTAT_CALL",
  "cmtatAction": "BATCH_MINT",
  "status": "PENDING_AEGIS_REVIEW",
  "isFinal": false,
  "createdAt": "2026-11-22T08:00:00.000Z",
  "updatedAt": "2026-11-22T08:00:00.000Z",
  "review": { "aegisStage": "SUPERVISOR" },
  "rejection": null,
  "failure": null,
  "result": null
}
```

| operationType | `result` |
|---|---|
| `WALLET_CREATE` | `{ "type": "WALLET_CREATE", "walletId": "…", "address": "0x…" }` |
| `WALLET_UPDATE` | `{ "type": "WALLET_UPDATE", "walletId": "…" }` |
| `WHITELISTED_WALLET_CREATE` | `{ "type": "WHITELISTED_WALLET_CREATE", "whitelistedWalletId": "…" }` |
| `CMTAT_DEPLOY` | `{ "type": "CMTAT_DEPLOY", "cmtatTokenId": "100000", "contractAddress": "0x…", "txHash": "0x…" }` |
| `CMTAT_CALL`、`CRYPTO_WITHDRAWAL` | `{ "type": "…", "txHash": "0x…" }`（`txHash` 偶爾暫時為 null，可再查詢） |

客戶申請回 ClientRequestView：

```json
{ "referenceId": "dq-investor-0001", "clientRequestId": "…", "kind": "INVESTOR", "externalId": "investor-0001",
  "status": "PENDING_AEGIS_REVIEW", "isFinal": false, "createdAt": "…", "updatedAt": "…", "rejection": null, "result": null }
```

完成後 `status: "COMPLETED"`、`result: { "clientId": "…" }`。

## 6. referenceId 與重送

- 會建立審核單或申請的 POST 都要帶 `referenceId`（上傳條款不用）。
- **整個 DigiQuick 內唯一**，1–64 字元，英數字、`-`、`_`，建議綁業務單號。
- 同一個 `referenceId` 只會建立一次。

| 情況 | 回應 |
|---|---|
| 已完成、內容相同 | `409 DUPLICATE_REQUEST`，`data.result` 是第一次的回應 |
| 還在處理 | `409 REQUEST_IN_PROGRESS`（retryable） |
| 內容不同 | `409 REFERENCE_ID_CONFLICT` |
| 結果不明（極少見） | `409 REQUEST_STATE_UNKNOWN`。Aegis 會收到告警並處理，**這段期間不要換 referenceId 重送**。處理完會是下列其中一種，且每筆請求只會有一個最終狀態：(1) 連結到實際的審核單，之後照常查詢（審核單若因技術問題卡住，Aegis 會拒絕它，你們會收到 `REJECTED` 且 `rejection.code = TECHNICAL_RETRY`）；(2) 標記失敗（`status = FAILED`、`failure.code = REQUEST_ABANDONED`，也會送 webhook）。**只有** `rejection.code = TECHNICAL_RETRY` 或 `failure.code = REQUEST_ABANDONED` 時，才用**新的** referenceId 重送；一般的審核拒絕（`rejection.code` 為 null）不要自動重送 |

- `400`、`422` 的 `retryable` 為 false；`422` 會佔用 referenceId。
- `502` 看 `retryable`：`true` 的（例如 KYT 沒有評分或服務錯誤、讀鏈、估 gas 時的網路問題）可以用同一個 referenceId 重試；`false` 的 `KV_UNAVAILABLE` 表示結果可能不明，請先查詢，查到 `REQUEST_STATE_UNKNOWN` 時照上表處理。
- 一個 referenceId 一旦結果不明，就**不會再被重新使用**；即使 Aegis 標記失敗，同一個 referenceId 重送也只會拿到那份失敗結果（`409 DUPLICATE_REQUEST`）。

## 7. 完整流程

### 7.1 錢包規劃

| 客戶 | 錢包 | 用途 | 需要 CFX |
|---|---|---|---|
| 發行方 | admin | 部署、指派角色、改合約資訊 | 要 |
| 發行方 | maker | allowlist、發行、銷毀 | 要 |
| 發行方 | custodian | Aegis 用來暫停、凍結 | 要 |
| 發行方 | 發行持有（選用） | 收還沒分配的代幣 | — |
| 投資人 | 託管錢包 | 持有代幣 | 提領時要 |

### 7.2 準備

```
① POST /v1/client-requests（kind=ISSUER）              申請發行方，Aegis 核准後 webhook 帶回 clientId
② POST /v1/client-requests（kind=INVESTOR）× N         每位投資人
③ POST /v1/clients/{issuerId}/wallets × 4               admin、maker、custodian、發行持有
④ POST /v1/clients/{investorId}/wallets                 投資人託管錢包
⑤ POST /v1/clients/{investorId}/whitelisted-wallets     要提領或持有在外部地址時：登記投資人自己的地址
⑥ POST /v1/clients/{investorId}/wallets/{walletId}/whitelisted-wallets  加進託管錢包的提領白名單
```

```json
{ "referenceId": "dq-issuer-C001", "externalId": "issuer-C001", "kind": "ISSUER",
  "companyName": "Example Issuer Ltd", "incorporationNo": "12345678", "contactEmail": "ops@example.com" }
```

```json
{ "referenceId": "dq-investor-0001", "externalId": "investor-0001", "kind": "INVESTOR",
  "firstName": "Tai Man", "lastName": "Chan", "birthday": "1985-03-12", "countryOfCitizenship": "HK" }
```

- 只需要這些欄位。`externalId` 不能重複；申請被拒後可以用同一個 `externalId`、新的 `referenceId` 重新申請。

```json
{ "referenceId": "dq-c001-wallet-maker", "name": "USDSN27 maker", "network": "CONFLUX_MAINNET", "usage": "USDSN27 mint, burn and allowlist" }
```

```json
{ "referenceId": "dq-investor-0002-wl-01", "name": "Self-custody", "network": "CONFLUX_MAINNET", "address": "0x…" }
```

- ⑤ 只能登記投資人**自己的**地址，會做 KYT。正式環境 Conflux 的 KYT 接上前，⑤ 和提領會回 `422 KYT_NOT_AVAILABLE_FOR_NETWORK`。

### 7.3 發行（發行方客戶底下）

```
⑦ POST /v1/clients/{issuerId}/cmtat/terms-documents        條款 PDF（10 MB 以內，回 201）
⑧ POST /v1/clients/{issuerId}/cmtat/deployments
⑨ POST …/cmtat/tokens/{cmtatTokenId}/calls  GRANT_ROLE × 6  （同一檔代幣一次只能處理一筆角色指派，撞到時回 409 REQUEST_IN_PROGRESS，稍後用同一個 referenceId 重送）
⑩ GET  …/cmtat/tokens/{cmtatTokenId}                         確認角色都已在鏈上生效
⑪ POST …/calls  BATCH_SET_ALLOWLIST                          投資人託管錢包、投資人自己的外部地址、發行持有錢包
⑫ POST …/calls  BATCH_MINT                                   直接發到投資人託管錢包
```

`⑪`

```json
{
  "referenceId": "dq-usdsn27-allowlist-01",
  "action": "BATCH_SET_ALLOWLIST",
  "allowlistStatus": true,
  "targets": [
    { "walletId": "<投資人 0001 託管錢包>" },
    { "walletId": "<投資人 0002 託管錢包>" },
    { "whitelistedWalletId": "<投資人 0002 自己的外部地址（⑤ 建立的）>" }
  ]
}
```

`⑫`

```json
{
  "referenceId": "dq-usdsn27-mint-01",
  "action": "BATCH_MINT",
  "targets": [
    { "walletId": "<投資人 0001 託管錢包>", "amount": "1000000" },
    { "walletId": "<投資人 0002 託管錢包>", "amount": "500000" }
  ]
}
```

### 7.4 存續

| 需求 | 做法 |
|---|---|
| 從發行持有錢包分配給投資人、投資人之間轉讓、換錢包 | 發行方 `BURN_AND_MINT` |
| 投資人提領到自己的地址 | 地址先完成 ⑤⑥，並由發行方加進 allowlist（⑪），再 `POST /v1/clients/{investorId}/withdrawals` |
| 修改合約資訊 | `SET_TERMS`、`SET_INFORMATION`、`SET_NAME`、`SET_SYMBOL`、`SET_TOKEN_ID` |

```json
{
  "referenceId": "dq-investor-0002-withdraw-01",
  "walletId": "<投資人 0002 託管錢包>",
  "whitelistedWalletId": "<投資人 0002 自己的外部地址>",
  "cmtatTokenId": "100000",
  "amount": "200000",
  "description": "Withdraw to self-custody"
}
```

### 7.5 到期

- 贖回：`BURN` 或 `BATCH_BURN`，對象可以是投資人託管錢包，也可以是投資人自己的外部地址。
- 暫停、停用：請 Aegis 在後台操作。

### 7.6 同時送出時可能的結果

冷錢包不會在審核期間預先保留要銷毀的代幣，所以同一個錢包的代幣異動同時送出時：

| 情況 | 受理 | 上鏈結果 | 代幣與 gas |
|---|---|---|---|
| 同時兩筆銷毀 | 兩筆都可能受理 | 兩筆合計超過餘額時，後上鏈的那筆失敗（`FAILED`）；合計足夠時，其他執行條件（gas、凍結、allowlist、審核結果等）都成立的話兩筆都可以成功 | 失敗那筆的代幣不變；gas 依實際上鏈的交易由 maker 錢包支付 |
| 先銷毀、再提領 | 都可能受理 | 合計超過餘額時，後上鏈的那筆失敗；足夠時，其他執行條件成立的話都可以成功 | 提領失敗時，提領鎖住的數量會解除；提領的 gas 由投資人錢包付 |
| 先提領、再銷毀 | 提領會先鎖住數量；銷毀受理時檢查剩餘可用量 | 剩餘不足 → 銷毀受理時就被擋；剩餘足夠 → 其他執行條件成立的話兩筆都可以成功 | — |
| 受理後、送出前就失敗 | — | 沒有上鏈 | 鎖住的數量解除，不扣 gas |

建議同一個錢包的代幣異動等前一筆有結果再送下一筆；大量對象用 `BATCH_*`（每筆最多 20 個）。

- 同一檔代幣的角色指派（GRANT_ROLE／REVOKE_ROLE）一次只處理一筆，前一筆還在送出時，下一筆會回 `409 REQUEST_IN_PROGRESS`。

## 8. 合約操作（`POST /v1/clients/{issuerId}/cmtat/tokens/{cmtatTokenId}/calls`）

| action | 欄位 | 簽名錢包 |
|---|---|---|
| `SET_ALLOWLIST` | `target`、`allowlistStatus` | maker |
| `BATCH_SET_ALLOWLIST` | `targets`（1–20）、`allowlistStatus` | maker |
| `MINT` | `target`、`amount` | maker |
| `BATCH_MINT` | `targets`（1–20，每筆含 `amount`） | maker |
| `BURN` | `target`、`amount` | maker |
| `BATCH_BURN` | `targets`（1–20，每筆含 `amount`） | maker |
| `BURN_AND_MINT` | `from`、`burnAmount`、`to`、`mintAmount` | maker |
| `GRANT_ROLE`、`REVOKE_ROLE` | `role`、`walletId`（只能是發行方自己的託管錢包） | admin |
| `SET_NAME`、`SET_SYMBOL`、`SET_TOKEN_ID`、`SET_INFORMATION` | `value` | admin |
| `SET_TERMS` | `termsName`、`termsDocumentId` | admin |

對象（`target`、`from`、`to`、`targets[]`）二選一：

| 寫法 | 可以是 |
|---|---|
| `{ "walletId": "…" }` | 發行方自己的託管錢包；已連結投資人的託管錢包 |
| `{ "whitelistedWalletId": "…" }` | 發行方自己登記的外部地址；已連結投資人登記的**自己的**外部地址 |

## 9. 查詢

| API | 用途 |
|---|---|
| `GET /v1/me` | 憑證、可用網路、哪些網路要求 KYT、webhook 設定 |
| `GET /v1/clients?kind=&externalId=`、`/v1/clients/{clientId}` | 已連結的客戶 |
| `GET /v1/client-requests/{referenceId}` | 客戶申請進度 |
| `GET /v1/clients/{clientId}/wallets`、`…/{walletId}` | 錢包、提領白名單、資產（單筆帶鏈上餘額） |
| `GET /v1/clients/{clientId}/whitelisted-wallets`、`…/{id}` | 外部地址 |
| `GET /v1/clients/{issuerId}/cmtat/terms-documents/{id}` | 條款 |
| `GET /v1/clients/{issuerId}/cmtat/tokens`、`…/{cmtatTokenId}` | 合約、角色、鏈上狀態 |
| `GET …/cmtat/tokens/{cmtatTokenId}/holders?type=ISSUER_CUSTODY\|ISSUER_EXTERNAL\|INVESTOR_CUSTODY\|INVESTOR_EXTERNAL` | 持有人的 allowlist、凍結、餘額（每頁最多 25；要讀鏈，請不要同時大量翻頁） |
| `GET /v1/requests/{referenceId}`、`GET /v1/requests` | 請求進度 |
| `GET /v1/operations/{operationId}` | Aegis 對你們代幣做的操作 |

## 10. Webhook

### 10.1 語意

跟 Stripe 等服務的慣例相同：
- **至少送一次**：同一則可能重複送達，用 `X-Aegis-Callback-Id` 去重。
- **送達可能亂序**：冷錢包產生通知的順序是對的（同一筆請求的狀態只會往前推進，`seq` 遞增），但派送是併發的、有重試和重送，**晚的通知可能比早的先到**（例如先收到 `COMPLETED`、後收到 `PROCESSING`）。接收端請對每筆請求記錄已處理的 `seq`，**只有收到的 `seq` 比較大時才更新**。
- 中間狀態可能被跳過；**最終狀態一定會產生通知**。
- 送不到時會在 1 分鐘、5 分鐘、15 分鐘、1 小時、6 小時後重試，共 6 次，之後標為 `FAILED`，可以用 10.4 補查與重送。冷錢包無法保證你們的系統一定收到，請以查詢 API 為準做對帳。
- Aegis 對你們代幣的暫停、凍結、強制轉移、停用也會通知（`referenceId` 為 `null`）。

### 10.2 內容

`POST {webhook URL}`，header：`Content-Type: application/json`、`X-Aegis-Callback-Id`、`X-Aegis-Signature`（webhook secret 對 body 原文做 HMAC-SHA256，hex）。

```json
{ "callbackId": "…", "seq": "10234", "eventType": "OPERATION_STATUS_CHANGED", "occurredAt": "…", "request": { "…": "RequestView" } }
{ "callbackId": "…", "seq": "10235", "eventType": "CLIENT_REQUEST_STATUS_CHANGED", "occurredAt": "…", "clientRequest": { "…": "ClientRequestView" } }
```

### 10.3 接收端

```ts
import crypto from 'crypto'
import express from 'express'

const WEBHOOK_SECRET = process.env.AEGIS_VAULT_WEBHOOK_SECRET!
const app = express()

// 由你們實作：在「同一個 DB transaction」內完成下面兩件事後 commit
//   1. INSERT callbackId（唯一鍵）；已存在 → 視為重複送達，直接結束
//   2. 依 requestId（或 clientRequestId）找到業務單；只有 BigInt(event.seq) > 已記錄的 seq 才更新狀態與 seq
declare function persistAegisEvent(callbackId: string, event: any): Promise<void>

app.post('/aegis-vault/webhook', express.raw({ type: 'application/json' }), async (req, res) => {
  const expected = crypto.createHmac('sha256', WEBHOOK_SECRET).update(req.body).digest('hex')
  const received = req.header('X-Aegis-Signature') ?? ''
  if (expected.length !== received.length || !crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(received))) {
    return res.status(401).end()
  }
  const callbackId = req.header('X-Aegis-Callback-Id') ?? ''
  const event = JSON.parse(req.body.toString('utf8'))
  try {
    await persistAegisEvent(callbackId, event) // commit 成功才回 2xx
    res.status(200).end()
  } catch (err) {
    res.status(500).end() // 冷錢包會依退避重送
  }
})
```

- 用原始 body 驗簽；回 2xx 算成功；10 秒內要回應。

### 10.4 補查與重送（`WEBHOOK_MANAGE`）

| API | 用途 |
|---|---|
| `GET /v1/webhook-deliveries?status=PENDING\|DELIVERED\|FAILED` | 通知紀錄 |
| `POST /v1/webhook-deliveries/{callbackId}/redeliver` | 重送（同一則 1 分鐘一次） |

## 11. 錯誤碼

| HTTP | code | retryable | 建議 |
|---|---|---|---|
| 400 | `VALIDATION_ERROR` | false | 修正參數 |
| 401／403／404／429 | 見 3.5（partner 被停用時也是 401） | 429 為 true | — |
| 409 | `DUPLICATE_REQUEST`、`REFERENCE_ID_CONFLICT`、`REQUEST_STATE_UNKNOWN` | false | 見第 6 節 |
| 409 | `REQUEST_IN_PROGRESS`、`WALLET_LOCKED_BY_PENDING_UPDATE` | true | 稍後用同一個 referenceId 重送 |
| 413 | `PAYLOAD_TOO_LARGE` | false | — |
| 422 | `ACTION_NOT_ALLOWED` | false | 這個 action 請 Aegis 在後台操作 |
| 422 | `NETWORK_NOT_SUPPORTED`、`KYT_NOT_AVAILABLE_FOR_NETWORK` | false | 看 `/v1/me` |
| 422 | `KYT_FAILED` | false | 確認地址 |
| 422 | `CLIENT_EXTERNAL_ID_EXISTS` | false | 確認 externalId |
| 422 | `MASTER_KEY_NOT_PROVISIONED` | false | 環境還沒準備好，聯絡 Aegis |
| 422 | `WHITELISTED_WALLET_INVALID`、`WALLET_UNCHANGED`、`DESTINATION_NOT_IN_WALLET_WHITELIST` | false | 確認外部地址與提領白名單 |
| 422 | `REQUESTER_NOT_ELIGIBLE`、`EXCEEDS_DAILY_LIMIT`、`WALLET_LOCKED` | false | 聯絡 Aegis |
| 422 | `INSUFFICIENT_GAS_FEE`、`INSUFFICIENT_BALANCE` | false | 補 CFX、確認餘額 |
| 422 | `CMTAT_TRANSFER_NOT_ALLOWED` | false | 確認 allowlist、凍結、暫停 |
| 422 | `CMTAT_ROLE_NOT_ASSIGNED`、`CMTAT_SIGNER_ROLE_MISSING`、`CMTAT_ROLE_OPERATION_IN_PROGRESS` | false | 角色相關 |
| 422 | `CMTAT_TARGET_NOT_ALLOWLISTED`、`CMTAT_INVALID_TARGET`、`CMTAT_STATE_UNCHANGED`、`CMTAT_VALIDATION_FAILED` | false | 對象與狀態 |
| 422 | `SYMBOL_ALREADY_EXISTS`、`TERMS_DOCUMENT_NOT_FOUND`、`FEE_ESTIMATE_REJECTED` | false | — |
| 502 | `FEE_ESTIMATE_UNAVAILABLE`、`KYT_UNAVAILABLE`（KYT 服務錯誤或沒有評分）、`CHAIN_READ_FAILED`、`FILE_STORAGE_UNAVAILABLE` | true | 稍後用同一個 referenceId 重試 |
| 502 | `KV_UNAVAILABLE` | false | 先查詢；結果不明時聯絡 Aegis |
| 500 | `INTERNAL_ERROR` | false | 先查詢；附 `X-Request-Id` 聯絡 Aegis |

## 12. API 一覽

| Method | Path | scope |
|---|---|---|
| GET | `/v1/me` | READ |
| GET | `/v1/clients`、`/v1/clients/{clientId}` | READ |
| POST | `/v1/client-requests` | CLIENT_WRITE |
| GET | `/v1/client-requests/{referenceId}` | READ |
| GET／POST | `/v1/clients/{clientId}/wallets`（GET 另有 `…/{walletId}`） | READ／WALLET_WRITE |
| POST | `/v1/clients/{clientId}/wallets/{walletId}/whitelisted-wallets` | WALLET_WRITE |
| GET／POST | `/v1/clients/{clientId}/whitelisted-wallets`（GET 另有 `…/{id}`） | READ／WALLET_WRITE |
| POST／GET | `/v1/clients/{issuerId}/cmtat/terms-documents`（GET `…/{id}`） | CMTAT_WRITE／READ |
| POST | `/v1/clients/{issuerId}/cmtat/deployments` | CMTAT_WRITE |
| GET | `/v1/clients/{issuerId}/cmtat/tokens`、`…/{cmtatTokenId}`、`…/{cmtatTokenId}/holders` | READ |
| POST | `/v1/clients/{issuerId}/cmtat/tokens/{cmtatTokenId}/calls` | CMTAT_WRITE |
| POST | `/v1/clients/{clientId}/withdrawals` | WITHDRAWAL_WRITE |
| GET | `/v1/requests/{referenceId}`、`/v1/requests`、`/v1/operations/{operationId}` | READ |
| GET／POST | `/v1/webhook-deliveries`、`…/{callbackId}/redeliver` | WEBHOOK_MANAGE |

## 13. 環境

| 環境 | 網路 | base URL |
|---|---|---|
| dev | Conflux eSpace testnet | 實作完成後提供 |
| stg | Conflux eSpace testnet | 實作完成後提供 |
| 正式 | Conflux eSpace 主網 | 上線前提供（要先接上 Conflux 的 KYT） |

- 每個環境的憑證、webhook secret、客戶各自獨立。
