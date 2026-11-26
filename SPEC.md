# 冷錢包對外 API（Open API）開發規格書

- 版本：v0.7（草稿）
- 日期：2026-11-25
- Jira：Epic AEG-42（規劃 AEG-40；前置 Bug AEG-43；相關 Bug AEG-41 在 AEG-3）
- 對應決策：`~/brains/aegis/decisions/cold-wallet-open-api.md`
- 對應的串接文件：[`digiquick-integration-guide-v0.7.md`](digiquick-integration-guide-v0.7.md)（兩份的 API 定義要一致）
- 依據的程式版本：aegis-vault-api `origin/develop` `8cef68d7`。開工前先 `git log 8cef68d7..origin/develop`。

> 讀者：實作的工程師。本文定義邊界、契約與流程，不提供完整程式碼。本文沒寫到的情況照既有程式的做法；沒有先例就停下來問。

## 修訂紀錄

| 版本 | 日期 | 內容 |
|---|---|---|
| v0.1 | 2026-11-20 | 初版（單一 client） |
| v0.2 | 2026-11-20 | partner 多 client、API_ONLY、跨 client CMTAT；處理 Codex 第一輪 16 項（附錄 A） |
| v0.3 | 2026-11-22 | （同日補：D1 預設 Aegis 核准、partner 可設自動核准；D2 KYT 用 Beosin；D3 照正式環境 `SYSTEM` 模式、master key 由 Aegis 事先建立，拿掉 PER_CLIENT 流程）依 Allen 決策：發行方與投資人 client 都由 API 開、partner 憑證可「產生」或「綁定」、partner 綁 HK、發行方可引用投資人的 SELF 地址、防超額採 (b)、webhook 至少一次；處理 Codex 第二輪 H1–H10、M1–M6（附錄 B）；CMTAT 完成狀態依賴 AEG-43 |
| v0.4 | 2026-11-22 | 處理 Codex 第三輪 R1–R17（附錄 C）：KV txId 改用 targetId 並可向 kv-api 查詢、client 申請單一 transaction、API 路徑的嚴格 KYT、HD 設定的 readiness 檢查、AUTO 核准由 main 輪詢、提領的跨 client 判斷、後台單在建單交易內登記追蹤、webhook 改「落庫單調、送達可亂序」與 TIMEOUT 規則、角色指派的持續佔用、AEG-43 的完成條件；依 Allen 決策：現有 bug 不修（只在 API 路徑繞開）、CMTAT 對帳單呈現交給 DQ |
| v0.5 | 2026-11-22 | 處理 Codex 第四輪（附錄 D）：**結果不明的請求改成營運人員在後台處理（A26）**，拿掉自動釋放與自動補 KV；對外狀態改成「從紀錄推導候選狀態＋讀已提交的公開狀態」；角色檢查納入 TIMEOUT 與未結算的指派；KYT provider 大寫與環境政策矩陣；加入白名單改成先鎖再重讀；client repo 傳 accessMode 與系統 actor；AUTO worker 退避；月結不動 UI 客戶；RPC 併發落在每次呼叫；加密金鑰提前部署 |
| v0.6 | 2026-11-23 | （同日補，依 Codex 輕量確認：角色佔用納入審核中的指派、同 rank 進度改用審核版本號、沒有 operation 時依 endpoint 選 KV 查詢、技術性拒絕的代碼）處理 Codex 第五輪（附錄 E）：KV key 只有 av-api 持有（A27）並禁止 open-api 呼叫核准 API；作廢與晚到的建單用同一筆請求列的鎖互斥；卡住的 operation 只拒絕、不再作廢（避免兩個最終狀態）；異常只看 `isKvOp`；正式環境的 KYT 設定錯誤一律不 ready；同 rank 也更新審核進度；角色佔用依證據解除；取鎖後的提早返回要 rollback；AUTO worker 用 savepoint；異常列表分頁 |
| v0.7 | 2026-11-25 | 新增機構投資人申請、相容預設值、核准建戶與權限驗收。補充 CMTAT 3.0.0 部署精度及角色錢包分工。 |

---

## 0. 背景與設計決定

### 0.1 背景

冷錢包的 CMTAT 操作目前只能在前台、後台用 GraphQL 發起。DigiQuick（DQ）要能用程式完成一檔 CMTAT 的生命週期。冷錢包的 client 是託管客戶本身（KYC、custody entity、帳務、對帳單），不能像熱錢包那樣把所有主體放在 DQ 一個 client 底下。

### 0.2 已定的設計決定

| # | 決定 | 來源 |
|---|---|---|
| A1 | 每個發行方、每位個人或機構投資人各自建立獨立 client。每個 clientId 只有一個業務角色（kind）。發行方為 COMPANY，投資人可為 INDIVIDUAL 或 COMPANY | Allen 11/25 |
| A2 | DQ 是 partner；partner 憑證代表已連結的 client；路徑 `/v1/clients/{clientId}/…`（在簽章範圍內） | — |
| A3 | client 存取模式 `UI`／`API_ONLY`；只有 API_ONLY client 能連結 partner；API_ONLY client 沒有真人使用者 | — |
| A4 | API_ONLY client 不做 client 端真人審核，只走 Aegis 審核（固定 policy 範本：API 帳號是唯一 requester、沒有 approver group） | Allen 11/20（方向 a） |
| A5 | **發行方與投資人 client 都由 DQ 用 API 開**（申請 → Aegis 核准，見 D1），只帶最少欄位 | Allen 11/22 |
| A6 | partner 連結有 `kind`：`ISSUER`／`INVESTOR`。**跨 client 只限 issuer → investor**（同 partner、雙方 ACTIVE）；issuer 之間、investor 之間不能互相引用 | Allen 11/22 |
| A7 | 發行方的 CMTAT 對象可以是：自己的託管錢包、自己的白名單錢包、**同 partner 投資人的託管錢包**、**同 partner 投資人的 SELF 白名單錢包**（所有權仍在投資人名下） | Allen 11/22（H3 選 A） |
| A8 | 開放修改錢包（加入 `whitelistedIds`） | Allen 11/20 |
| A9 | 上線前提：Conflux 有 KYT；白名單錢包建立與提領都套用 | Allen 11/20 |
| A10 | **後台建立 partner、核發憑證：可以「產生」金鑰（私鑰只顯示一次）或「綁定」DQ 提供的公鑰** | Allen 11/22 |
| A11 | **partner 綁單一 custody entity，v1 固定 HK**；所有 partner 管理動作與其 client 都照這個邊界 | Allen 11/22 |
| A12 | **防超額採 (b)**：不在審核期間保留代幣；兩筆同時送出時可能都建立成功，第二筆上鏈失敗、不動帳 | Allen 11/22 |
| A13 | **webhook 至少送一次**；最終狀態可補查；6 次失敗進 FAILED、可人工重送；不承諾接收端一定收到 | Allen 11/22 |
| A14 | 同 repo、不同 process、不同 container（`apps/open-api`）；API 只是轉接，審核、檢查、帳務邏輯不重寫 | — |
| A15 | 簽章沿用熱錢包（RSA-PSS SHA-256），path 含 query、nonce 防重放、timestamp 雙向檢查、沒有 admin key | — |
| A16 | 建立 operation／申請的 POST 非同步、帶 partner 內唯一的 `referenceId` | — |
| A17 | Aegis 端 CMTAT action 不開放給 API；發生在已連結發行方的代幣上時用 webhook 通知 | 照 `cmtat-role-wallets` |
| A18 | 白名單錢包在 API 只開放 `owner = SELF`，不開放上傳證明文件 | — |
| A19 | **CMTAT 部署與合約操作的 `COMPLETED` 要等帳務套用完成**（依賴 AEG-43） | Allen 11/22（H7 在本需求另開卡先修） |
| A20 | **API 開的 client 預設要 Aegis 核准**；保留彈性：partner 可以設成自動核准（`client_request_approval = AUTO`），流程與結果相同，只是不等人 | Allen 11/22（D1） |
| A21 | **KYT 用 Beosin**；Conflux eSpace 的 chain id 之後由 Beosin 提供，拿到後補進 `BeosinBlockchainMapper`（不在本案） | Allen 11/22（D2） |
| A22 | **`HD_WALLET_MODE` 跟正式環境一樣（`SYSTEM`，全系統一把 master key），master key 由 Aegis 事先建立**；建 client 時不處理 master key。正式環境已設 `HD_WALLET_SUPPORTED_CURVES`（含 secp256k1，Allen 11/22 確認） | Allen 11/22（D3） |
| A23 | **webhook：落庫單調、送達可亂序**（同 Stripe 等服務的慣例）。接收端用事件 ID 去重、用 `seq` 判斷新舊、必要時呼叫查詢 API | 11/22（Allen 問業界做法後採用） |
| A24 | **既有系統的 bug 不在本案修**（Beosin 分數 0 被當成沒有分數、TIMEOUT 無條件覆蓋狀態、月結寫檔未 await 與每個 client 明細只取 500、通知信收件人混寄）。本案只在 API 路徑上繞開或定義行為，不改前台行為 | Allen 11/22 |
| A25 | **CMTAT 在月結對帳單的呈現不在本案**（由 DQ 端自行處理）；本案只保證月結不會因 API_ONLY client 而失敗 | Allen 11/22 |
| A27 | **KV 的 API key 只有 av-api 持有**（main 與 open-api 都是 av-api 的 process，共用同一把）；open-api 不得呼叫 KV 的核准 API（§4.3.3） | Allen 11/23 |
| A26 | **結果不明（UNKNOWN）的請求由 Aegis 營運人員在後台處理，不做自動恢復**：referenceId 一旦 UNKNOWN 就永不釋放；後台提供自動診斷與「連結」「作廢」兩個動作；作廢後 DQ 用新的 referenceId 重送；全程不需要工程師進正式環境的 DB 或 KV | Allen 11/22 |

### 0.3 待確認

| # | 問題 | 預設（先照這個寫） |
|---|---|---|
| D1–D3 | 已定（見 A20–A22） | — |
| D4 | 投資人錢包的原生幣（gas）由誰補 | 不在 API 範圍；前期在「投資人轉出改用發行方 `BURN_AND_MINT`」與「熱錢包自動補幣」之間選，留待討論。dev 測試時投資人提領的案例，事先替投資人錢包補 CFX |
| D5 | 客戶審核需收哪些文件、由何管道提交 | 本版先依 §3.5.4 開申請端點，審核收件另定 |

---

## 1. 核心目標與非目標

### 1.1 目標

1. `apps/open-api` process：REST + JSON，前綴 `/v1`，只開 §3.5 的端點。
2. partner 模型：partner（HK）、憑證（產生或綁定）、client 連結（ISSUER／INVESTOR）、API_ONLY client、API 帳號、固定 policy。
3. client 申請（發行方、投資人）→ Aegis 核准 → 建立 client（沒有真人使用者）。
4. CMTAT 生命週期：

   | 階段 | API |
   |---|---|
   | 準備 | 申請發行方與投資人 client → 建錢包 → 投資人登記自己的外部地址（SELF）→ 加進投資人錢包的 `whitelistedIds` |
   | 發行 | 上傳條款 → 部署 → GRANT_ROLE × 6 → allowlist（含投資人的託管錢包與 SELF 地址）→ mint 給投資人 |
   | 存續 | `BURN_AND_MINT`（分配、轉讓、換錢包）→ 投資人提領到自己的地址 → `SET_*` |
   | 到期 | `BURN`／`BATCH_BURN`（含投資人的外部地址）→ 暫停、停用由 Aegis 後台操作 |
   | 全程 | 查詢、webhook、補查與重送 |

5. 後台：partner、憑證（產生／綁定、核准、撤銷）、client 申請核准、webhook、狀態不明的請求、對跨 client 持有人執行凍結等 Aegis 操作。
6. 既有系統的必要修正：月結對帳單支援 API_ONLY client（§4.12）、建 client 的 service 拆兩段、角色與 policy 的防護、白名單錢包同 client 檢查。

### 1.2 非目標

- 任何核准／拒絕 API；Aegis 端 CMTAT action 的 API。
- policy 建立與修改 API；查使用者 API；檔案上傳 API；白名單錢包 `owner ≠ SELF`。
- 非 CMTAT 資產的提領、staking、OTC、銀行。任意地址、原始 calldata。
- 投資人之間直接互轉（託管錢包 A → 託管錢包 B 的提領）：用發行方的 `BURN_AND_MINT`。
- 審核期間保留代幣防超額（A12）。
- `HYBRID`、既有 UI client 轉 API_ONLY、Ethereum 主網、入金通知 webhook。
- AEG-41（金額 0 略過 client 審核）、AEG-43（CMTAT 帳務冪等）的實作本身（各自的卡）。
- A24 列出的既有 bug；A25 的對帳單 CMTAT 呈現。
- 結果不明請求的自動恢復（自動釋放 referenceId、自動補 KV 關聯）（A26）。

### 1.3 絕對不要改動的範圍

| 範圍 | 說明 |
|---|---|
| 審核流程定義 | `*AdminReviewFlow`、`ReviewOperationService`、`Policy.getClientReviewFlow`、`WalletUtils.computeTriggeredCondition`（AEG-41 處理） |
| CMTAT 業務規則 | `validate-cmtat-call.ts` 的各 action 規則、`tx-params-builder.ts`、`CMTAT_ROLES_REQUIRED_BY_ACTION`、`CMTAT_CLIENT_ACTIONS`／`CMTAT_ADMIN_ACTIONS`、`resolveCmtatSigner`。**例外**只有 §4.10 的「對象 client 判斷」 |
| 簽名錢包 | 永遠是發行方自己的錢包；`resolveSignerWallet` 與 `cmtat-call-operation.ts` 第 250、296 行的檢查不動 |
| 既有 GraphQL 的 schema | 不改任何既有 input／output 型別（包括 `UserRoleDto` 的變數型別）；只新增 |
| KV、合約 | 全部不動 |
| 排程與初始化 | `core/job`、`EVALUATE_BEGINNING` 只在 main |
| Migration | 不改既有；不用 `migration:generate`；只有 main 跑 |
| 全域資產可見性 | `isAssetTypeVisible`（GraphQL 用）不改；REST 自己在 DB 判斷（§4.10） |

---

## 2. 受影響檔案清單

路徑相對於 aegis-vault-api，另有標明除外。

### 2.1 新的 process

| 檔案 | 職責 |
|---|---|
| `apps/open-api/src/main.ts` | `NestFactory.create(OpenApiAppModule, { bodyParser: false })`；依路徑掛 `express.json({ limit, verify })`（條款 15 MB、其他 1 MB）；trust proxy；**readiness**：啟動時確認 `migrations` 表已包含程式內所有 migration，否則不 listen、log 並重試（main 負責跑 migration）；不發 `EVALUATE_BEGINNING`；route allowlist 檢查 |
| `apps/open-api/src/open-api-app.module.ts` | §4.8 |
| `apps/open-api/src/typeorm.ts` | 沿用 `apps/main/src/typeorm.ts` 已在用的共用 entity 陣列（`database/entities`），`migrationsRun: false` |
| `apps/open-api/src/configs.ts`、`helpers.ts` | 只提供 open-api 實際 DI 用到的 config 與 helper（不整包帶入 main 的 `helpers`，例如不需要 Google Drive）；用精簡的 env 冷啟驗證 |
| `apps/open-api/tsconfig.app.json`、`apps/open-api/test/jest-e2e.json` | — |
| `nest-cli.json`、`package.json` | project、`build:open-api`、`start:open-api:*`、`test:e2e:open-api`；直接 import `express` 時列為直接依賴 |
| `open-api.dockerfile`、`entrypoint/open-api.sh` | — |
| `.github/workflows/ci.yml` | open-api image job；**另一個含 Postgres 13 service 的 e2e job**（現有 reusable build 的 prebuild 有 skip／copy 條件，不適合跑 e2e） |

### 2.2 `core/open-api/`

| 檔案 | 職責 | process |
|---|---|---|
| `partner/*` | partner、憑證（產生／綁定）、連結、custody entity 檢查、後台 GraphQL | main（repo 兩邊） |
| `client-request/*` | client 申請：建立（open-api）、核准／拒絕（main）、AUTO 申請的輪詢處理（main，§4.7） | 兩邊 |
| `provisioning/api-client-provisioning.service.ts` | 核准（人工或 AUTO）時建立 client（company 或 individual）、API 帳號、固定 policy、連結 | main |
| `auth/*` | `OpenApiAuthGuard`、`ClientScopeGuard`、簽章、nonce、限流、scope | open-api |
| `idempotency/*` | `open_api_requests` repo、`IdempotentWriteService`、清理程序 | 兩邊／open-api |
| `locks/open-api-lock.repo.ts` | `open_api_locks` 表（§4.4，只用於 CMTAT 角色指派），不用專用連線 | open-api |
| `kyt/kyt-gate.service.ts` | §4.6 的網路分類與啟動檢查（嚴格判斷在 domain 的 `kytPolicy: 'STRICT'`） | open-api |
| `holders/allowed-holder.service.ts` | §4.10 的 `isAllowedHolderClient`、`assertTargetAllowed`；domain、估費 query、後台、REST 共用 | 兩邊 |
| `view/*` | 輸出轉換、狀態與 rank | 兩邊 |
| `errors/*` | 錯誤格式、domain 錯誤對照（含 `retryable`） | open-api |
| `controllers/*` | §3.5 | open-api |
| `webhook/tracking.repo.ts` | `open_api_tracked_targets`（§4.5）；提供 `registerInTransaction(op, queryRunner)` 給建單交易使用 | 兩邊 |
| `kv/kv-client-guard.ts` | open-api process 的 KV HTTP client 只允許建任務與查詢的路徑；呼叫 `approvalTasks/*/records`（含 `supervisorApproval`）一律丟例外（A27）；單元測試覆蓋 | open-api |
| `anomaly/anomaly-diagnostics.service.ts` | 後台「Open API 異常請求」的自動診斷（§4.3.3，**只在點開單筆時查**）：查 av-api 的 operation 與本地 `kv_tasks`、呼叫 kv-api（交易任務：`GET /blockchain/task/:txId/status`；建錢包：`GET /wallet/:walletId/info`、`GET /wallet/:walletId/tasks`），結果分 `FOUND`／`NOT_FOUND`／`ERROR` | main |
| `anomaly/anomaly-alert.ts` | UNKNOWN 產生時寫結構化 error log（供 Grafana 告警）並寄信給 Aegis 營運信箱（`OPEN_API_ALERT_EMAIL`） | open-api |
| `hd/hd-readiness.ts` | §4.9 的 HD 設定檢查，兩個 process 啟動時執行 | 兩邊 |
| `webhook/outbox.listener.ts` | §4.5 | 兩邊 |
| `webhook/reconciler.ts`、`webhook/dispatcher.ts`、`webhook/url-guard.ts`、`webhook/secret-crypto.ts` | §4.5 | open-api（crypto 兩邊） |
| `open-api-admin.module.ts`、`open-api-public.module.ts` | 兩個 process 的組合 | — |

### 2.3 資料庫

| 檔案 | 內容 |
|---|---|
| `database/entities/open-api-*.ts`（11 個） | §3.1 |
| `database/migrations/<ts>-AddUserApiRole.ts` | 單獨一支（`ALTER TYPE … ADD VALUE` 不能跟使用它的語句同 transaction） |
| `database/migrations/<ts>-AddClientAccessMode.ts` | `clients.access_mode` |
| `database/migrations/<ts>-CreateOpenApiTables.ts` | §3.1 |
| `database/migrations.ts`、`database/types/*` | 註冊與 enum |

### 2.4 既有檔案的最小修改

| 檔案 | 修改 |
|---|---|
| `core/_interfaces/account/models/role.ts` | 加 `Role.USER_API`；`RoleActions` 見 §3.3.4；不加進人工建帳用的角色陣列 |
| `core/_interfaces/account/models/action.ts` | `VIEW_OPEN_API`、`MANAGE_OPEN_API`、`APPROVE_OPEN_API`、`REVIEW_OPEN_API_CLIENT_REQUEST` |
| 建使用者的 service（`commandUserCreate`、後台 `createUser`、`create-company-client.ts`、`create-individual-client.ts` 呼叫的 service） | **GraphQL input 型別不變**；在 service 層：角色必須屬於人工可建立的集合（`CompanyUserRoles`／`IndividualUserRoles`），否則 `RoleNotAllowedError`；client 是 API_ONLY 時拒絕（`ApiOnlyClientCannotHaveUsersError`）。補一個 schema 相容測試（用目前前台、後台的 GraphQL 操作文件驗證 schema 沒有破壞性變更） |
| `CreateCompanyClientService`、`CreateIndividualClientService` | 拆成「寫入 client 資料」與「建立真人使用者＋寄信」兩段；前台路徑照舊兩段都做；provisioning 只用第一段 |
| `IndividualMutateRepo.insert`、`CompanyMutateRepo.insert` 的 input 型別 | `email`、`mobile` 允許 `null`（individuals 表本來就可空；companies.email 是 NOT NULL，company 申請必須帶 email） |
| sign-in | `USER_API` 一律一般登入失敗 |
| `validate-policy-input-service.ts` | approver／viewer 不能是 `USER_API` |
| `wallet-create-command.ts`、`wallet-update` service | 白名單錢包要存在、同鏈、**同 client**；wallet update 另外確認來源錢包屬於呼叫的 client |
| `WalletUpdateCommandService` | 新增 `createAddWhitelisted(walletId, whitelistedWalletIds, requesterId, options)`：在**同一個 QueryRunner** 內依序 ① 用 `INSERT … ON CONFLICT DO NOTHING RETURNING` 取得 object lock（拿不到回 `ObjectHasBeenLockedError`，不 catch 23505 後續用已失敗的交易）② 重新讀錢包 ③ 以**現行** policy 與現行 `whitelistedIds` 算聯集 ④ insert operation。**取得鎖之後，任何沒有建立 operation 的返回（驗證失敗、全部已存在等）都要 `setRollback()`**（`common/misc/transaction.ts` 的 `runTransaction` 對一般 return 會 commit），鎖不能殘留。既有 `create` 也改用同一個 QueryRunner 取鎖 |
| `wallet-transaction-operation-service-base.ts` | `create(wallet, input, requesterId, options?: { operationId?: string; kvTxId?: string; kytPolicy?: 'DEFAULT' \| 'STRICT' })`：`kvTxId` 有傳就用（open-api 一律傳 targetId），沒傳照舊；`kytPolicy: 'STRICT'` 見 §4.6；fee 估值：`input.estimatedGasFee`／`maxEstimatedGasFee` 為 null 時用內部 `feeEstimate()` 結果填入，**寫入前轉成非空 Decimal**；`FeeEstimateError` 保留結構化原因（模擬執行 revert vs RPC 失敗） |
| 建 operation 的 domain service（base `create`、`wallet-create-command`、`wallet-update`、`whitelisted-wallet-create-command`） | 加 `options.insertGuard?: (queryRunner) => Promise<void>`：在**寫入 operation 的同一個 transaction 內、INSERT 之前**呼叫；它丟出例外時整個 transaction rollback（不寫 operation、不鎖資產）。open-api 一律傳入 §4.3.3 的作廢檢查 |
| `CmtatCallOperationService.create`、deploy、withdrawal service | 把 `options` 傳給 `super.create`；三者目前各自產生 txId（withdrawal 約第 157 行、cmtat-call 約第 419 行、deploy 約第 148 行），改成優先用 `options.kvTxId` |
| `cmtat-call-operation.ts` `createByAdmin` | 在建 operation 的同一個 transaction 內呼叫 `TrackingRepo.registerInTransaction`（token 屬於 ACTIVE ISSUER 連結時才登記），§4.5 |
| `whitelisted-wallet-create-command.ts` | 加 `options.kytPolicy`（同 base） |
| `UserMutateRepo.insert` | 接受 `Role.USER_API` 與 `mobile: null`（只給 provisioning 用） |
| `ClientMutateRepo.insert`、`CompanyMutateRepo.insert`、`IndividualMutateRepo.insert` | input 加 `accessMode: ClientAccessMode`（必填，不靠 DB 預設）與 `requesterId: string \| null`（AUTO 為 null）；寫入 `clients.access_mode`、`clients.createdBy`。既有 UI 入口明確傳 `accessMode: UI` 與真人 requesterId；`companies.email` 仍必填 |
| `cmtat-token-role.repo.ts` `findPendingRoleOperations` | 加 opt-in 參數 `includeUnsettled`（只有 open-api 路徑傳 true，前台行為不變）：對同一 token 的 GRANT／REVOKE，用 §4.5 的 `deriveCandidate` 判斷——候選狀態是 `PENDING_CLIENT_REVIEW`、`PENDING_AEGIS_REVIEW`、`PROCESSING`（含已完成但 AEG-43 未結算）、沒有結論的 `TIMEOUT` 的都視為進行中；候選狀態是 `REJECTED`、`FAILED`、已結算的 `COMPLETED` 的不算（即使資料庫 state 被改成 TIMEOUT） |
| 通知信的收件人解析（例如 `crypto-withdrawal-email.ts`、`wallet-email.ts`、CMTAT 的 email service） | 在組收件人清單的地方排除 `USER_API` 帳號；其他收件人與寄送邏輯不變 |
| `alchemy.ts` 約第 72 行的 fee data 例外、`ethereum-estimate-fee.ts` | provider adapter 保留錯誤分類（模擬 revert vs RPC／`getFeeData` 失敗），往上傳到 `FeeEstimateError` 的結構化原因 |
| `wallet-create-command.ts`、`wallet-update`、`whitelisted-wallet-create-command.ts` 與它們的 mutate repo | 接受 `options.operationId`，一路傳到最終 INSERT（目前 repo 內部 `v4()`）；`wallet-create-command.ts` 另接受 `options.walletId`（open-api 一律傳 targetId），新錢包與送給 KV 的 walletId 都用它，供後台診斷查詢 |
| `mutation.request-cmtat-call.ts`、deploy、withdrawal 的 resolver | 主體抽成共用 request service（行為與錯誤訊息不變） |
| `cmtat-call-operation.ts` `_loadWallets`、`_loadWhitelistedWallets` | client 條件改呼叫 `AllowedHolderService`（§4.10）；**GRANT_ROLE／REVOKE_ROLE 另加明確檢查：target 必須是發行方自己的託管錢包**（不依賴 loader） |
| `query.cmtat-call-fee-estimate.ts`（約第 105 行的同 client 檢查） | 改呼叫同一個 `AllowedHolderService` |
| 後台 CMTAT token 的 resolver | 新增 field `linkedHolders`（同 partner 投資人的託管錢包與 SELF 白名單錢包，分頁），供後台凍結等操作選擇對象 |
| `core/misc/statement/services/create-statement.ts` | §4.12（只做 API_ONLY 相容所需的最小修改） |
| `cmtat-asset-type-registry.ts` | 不新增方法；REST 在載入錢包前呼叫既有的 `update()`（§4.10） |
| `apps/main/src/app.module.ts` | import `OpenApiAdminModule` |
| `api.graphql` | 重新產生（只新增） |

### 2.5 其他 repo

| repo | 內容 |
|---|---|
| aegis-vault-admin | 「Open API」頁：partner、憑證（綁定公鑰或產生金鑰並下載私鑰）、client 申請（核准、拒絕）、連結列表與停用、webhook、UNKNOWN 請求；client 詳情顯示存取模式與 DQ issuer／investor；CMTAT 管控視窗用 `linkedHolders` 顯示投資人持有人並可凍結；operation 頁顯示跨 client 對象「名稱（client displayId）」；`USER_API` 角色名稱 |
| aegis-vault-app | `USER_API` 角色名稱；policy 選單排除 API 帳號 |
| aegis-deploy | open-api Deployment／Service／Ingress（獨立 host、body 16 MB）、image-updater 規則；rollout 驗收 |
| 正式環境 | compose、Caddy host、SG（正式部署另外取得同意；KYT 未解決不上線） |
| architecture-docs | `services/aegis-vault-api.md`、`cross-repo/protocols-and-auth.md` |

---

## 3. 資料結構與介面契約

### 3.1 DB Schema

```sql
CREATE TYPE client_access_mode AS ENUM ('UI', 'API_ONLY');
ALTER TABLE clients ADD COLUMN access_mode client_access_mode NOT NULL DEFAULT 'UI';

CREATE TYPE open_api_partner_state AS ENUM ('ACTIVE', 'DISABLED');
CREATE TYPE open_api_client_request_approval AS ENUM ('MANUAL', 'AUTO');
CREATE TABLE open_api_partners (
  id              uuid PRIMARY KEY,
  name            varchar(100) NOT NULL UNIQUE,
  custody_entity  "<CustodyEntityEnumName>" NOT NULL,   -- v1 只允許 HK
  client_request_approval open_api_client_request_approval NOT NULL DEFAULT 'MANUAL',  -- A20
  state           open_api_partner_state NOT NULL,
  created_by      uuid NOT NULL REFERENCES accounts(id),
  created_at      timestamptz NOT NULL DEFAULT now()
);

CREATE TYPE open_api_credential_state AS ENUM ('PENDING_APPROVAL', 'ACTIVE', 'REVOKED');
CREATE TYPE open_api_key_source AS ENUM ('BOUND', 'GENERATED');
CREATE TABLE open_api_credentials (
  id                     uuid PRIMARY KEY,
  partner_id             uuid NOT NULL REFERENCES open_api_partners(id),
  name                   varchar(100) NOT NULL,
  api_key                varchar(64) NOT NULL UNIQUE,     -- 'ak_' + base64url(24 bytes)
  key_source             open_api_key_source NOT NULL,
  public_key_pem         text NOT NULL,                   -- RSA 2048/3072/4096 SPKI；GENERATED 時也只存公鑰
  public_key_fingerprint char(64) NOT NULL,
  ip_allowlist           text[] NOT NULL,
  scopes                 text[] NOT NULL,
  state                  open_api_credential_state NOT NULL,
  expires_at             timestamptz NOT NULL,
  created_by             uuid NOT NULL REFERENCES accounts(id),
  created_at             timestamptz NOT NULL DEFAULT now(),
  approved_by            uuid NULL REFERENCES accounts(id),
  approved_at            timestamptz NULL,
  revoked_by             uuid NULL REFERENCES accounts(id),
  revoked_at             timestamptz NULL,
  revoke_reason          varchar(500) NULL,
  last_used_at           timestamptz NULL
);

-- externalId 的唯一佔用（申請與連結共用，避免同一個 externalId 同時被兩邊佔用）
CREATE TABLE open_api_external_ids (
  partner_id   uuid NOT NULL REFERENCES open_api_partners(id),
  external_id  varchar(100) NOT NULL,
  client_request_id uuid NOT NULL,
  created_at   timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (partner_id, external_id)
);

CREATE TYPE open_api_client_kind AS ENUM ('ISSUER', 'INVESTOR');
CREATE TYPE open_api_link_state AS ENUM ('ACTIVE', 'DISABLED');
CREATE TABLE open_api_partner_clients (
  client_id          uuid PRIMARY KEY REFERENCES clients(id),
  partner_id         uuid NOT NULL REFERENCES open_api_partners(id),
  kind               open_api_client_kind NOT NULL,
  external_id        varchar(100) NOT NULL,
  api_account_id     uuid NOT NULL REFERENCES accounts(id),
  default_policy_id  uuid NOT NULL,
  state              open_api_link_state NOT NULL,
  created_at         timestamptz NOT NULL DEFAULT now(),
  disabled_by        uuid NULL REFERENCES accounts(id),
  disabled_at        timestamptz NULL,
  UNIQUE (partner_id, external_id),
  FOREIGN KEY (partner_id, external_id) REFERENCES open_api_external_ids(partner_id, external_id)
);
CREATE INDEX ON open_api_partner_clients (partner_id, kind, state);

CREATE TYPE open_api_client_request_state AS ENUM ('PENDING', 'APPROVED', 'REJECTED');
CREATE TABLE open_api_client_requests (
  id                      uuid PRIMARY KEY,
  partner_id              uuid NOT NULL REFERENCES open_api_partners(id),
  kind                    open_api_client_kind NOT NULL,
  external_id             varchar(100) NOT NULL,
  payload                 jsonb NOT NULL,                 -- §3.5.4
  state                   open_api_client_request_state NOT NULL,
  version                 int NOT NULL DEFAULT 0,         -- 條件式更新用
  client_id               uuid NULL REFERENCES clients(id),
  approval_mode           open_api_client_request_approval NOT NULL,  -- 送出當下的 partner 設定（之後切換不影響已送出的申請）
  auto_attempts           int NOT NULL DEFAULT 0,
  auto_last_error         varchar(500) NULL,
  auto_next_attempt_at    timestamptz NOT NULL DEFAULT now(),
  reviewed_by             uuid NULL REFERENCES accounts(id),  -- AUTO 時為 null
  auto_approved           boolean NOT NULL DEFAULT false,
  reviewed_at             timestamptz NULL,
  reject_reason           varchar(500) NULL,
  created_at              timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE open_api_nonces (
  credential_id uuid NOT NULL REFERENCES open_api_credentials(id),
  nonce         varchar(64) NOT NULL,
  created_at    timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (credential_id, nonce)
);
-- 清理：刪除 created_at < now() − 15 分鐘（> timestamp 容許窗 ±300 秒的兩倍）

CREATE TYPE open_api_request_state AS ENUM ('PROCESSING', 'DONE', 'UNKNOWN', 'ABANDONED');
CREATE TYPE open_api_request_target AS ENUM ('OPERATION', 'CLIENT_REQUEST');
CREATE TABLE open_api_requests (
  id              uuid PRIMARY KEY,
  partner_id      uuid NOT NULL REFERENCES open_api_partners(id),
  credential_id   uuid NOT NULL REFERENCES open_api_credentials(id),
  client_id       uuid NULL REFERENCES clients(id),
  reference_id    varchar(64) NOT NULL,
  endpoint        varchar(120) NOT NULL,
  request_hash    char(64) NOT NULL,
  request_body    jsonb NULL,                 -- OPERATION 時存原始請求內容（含 path 參數），供人工判斷；CLIENT_REQUEST 不存（申請表已有 payload）
  target_type     open_api_request_target NOT NULL,
  target_id       uuid NOT NULL,              -- 預先產生，傳到最終 INSERT；OPERATION 時也作為 KV 的 txId；WALLET_CREATE 時也作為 walletId
  state           open_api_request_state NOT NULL,
  version         int NOT NULL DEFAULT 0,     -- 條件式更新：handler、清理程序、人工修復都帶舊 version
  http_status     int NULL,
  response_body   jsonb NULL,
  unknown_reason  varchar(500) NULL,
  resolution      varchar(20) NULL,           -- LINKED | ABANDONED（§4.3.3）
  resolved_by     uuid NULL REFERENCES accounts(id),
  resolved_at     timestamptz NULL,
  resolution_note varchar(500) NULL,
  diagnostics     jsonb NULL,                 -- 處理當下的診斷結果快照（稽核用）
  created_at      timestamptz NOT NULL DEFAULT now(),
  completed_at    timestamptz NULL,
  UNIQUE (partner_id, reference_id),
  UNIQUE (target_type, target_id)
);

-- CMTAT 角色指派的佔用（不佔 DB 連線）
CREATE TABLE open_api_locks (
  key          varchar(200) PRIMARY KEY,   -- 'cmtat-roles:<cmtatTokenId>'
  request_id   uuid NOT NULL REFERENCES open_api_requests(id) ON DELETE CASCADE,
  acquired_at  timestamptz NOT NULL DEFAULT now()
);
-- 只有在持有者的 open_api_requests 已是 DONE 或 ABANDONED、或已被刪除時才能被接手；PROCESSING／UNKNOWN 期間一律保留

CREATE TABLE open_api_webhook_endpoints (
  partner_id        uuid PRIMARY KEY REFERENCES open_api_partners(id),
  url               varchar(500) NOT NULL,
  secret_ciphertext text NOT NULL,          -- AES-256-GCM
  enabled           boolean NOT NULL DEFAULT false,   -- 建立時停用，secret 交付後才啟用
  updated_by        uuid NOT NULL REFERENCES accounts(id),
  updated_at        timestamptz NOT NULL DEFAULT now()
);

-- 需要通知的對象：建立時登記，直到最終事件落庫
CREATE TABLE open_api_tracked_targets (
  target_type          open_api_request_target NOT NULL,
  target_id            uuid NOT NULL,
  partner_id           uuid NOT NULL REFERENCES open_api_partners(id),
  last_rank            int NOT NULL DEFAULT 0,     -- §3.7.1 的 rank（只增不減）
  last_status          varchar(40) NULL,           -- 已提交的對外 status
  last_view            jsonb NULL,                 -- 與 last_status 一起提交的完整 view（含 result、failure），GET 直接回它
  last_view_version    int NOT NULL DEFAULT 0,     -- 同一 rank 內的進度版本（§4.5），用來判斷 view 新舊
  final_event_stored   boolean NOT NULL DEFAULT false,  -- rank = 5 時為 true
  timeout_since        timestamptz NULL,           -- 進入 TIMEOUT 的時間（reconciler 前 30 天每 5 分鐘、之後每天一次）
  issuer_client_id     uuid NULL,                  -- 後台單：建單當下的發行方 client（之後連結停用也保留歸屬）
  created_at           timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (target_type, target_id)
);
CREATE INDEX ON open_api_tracked_targets (final_event_stored) WHERE final_event_stored = false;

CREATE TYPE open_api_webhook_event_state AS ENUM ('PENDING', 'DELIVERED', 'FAILED');
CREATE TABLE open_api_webhook_events (
  id               uuid PRIMARY KEY,
  seq              bigserial NOT NULL UNIQUE,
  partner_id       uuid NOT NULL REFERENCES open_api_partners(id),
  target_type      open_api_request_target NOT NULL,
  target_id        uuid NOT NULL,
  status           varchar(40) NOT NULL,
  rank             int NOT NULL,
  payload          jsonb NOT NULL,
  state            open_api_webhook_event_state NOT NULL DEFAULT 'PENDING',
  attempts         int NOT NULL DEFAULT 0,
  next_attempt_at  timestamptz NOT NULL DEFAULT now(),
  lease_token      uuid NULL,
  lease_until      timestamptz NULL,
  last_http_status int NULL,
  last_error       varchar(500) NULL,
  created_at       timestamptz NOT NULL DEFAULT now(),
  delivered_at     timestamptz NULL,
  UNIQUE (target_type, target_id, status)
);
CREATE INDEX ON open_api_webhook_events (state, next_attempt_at);
CREATE UNIQUE INDEX ON open_api_webhook_events (target_type, target_id) WHERE rank = 5;   -- 每個對象只能有一筆最終事件
```

- 「使用者的角色」存在 `accounts.role`（`users` 表沒有 role 欄位），本文所有「client 有沒有真人使用者」的判斷都是 `users JOIN accounts ON users.id = accounts.id WHERE users.clientId = $1 AND accounts.role <> 'USER_API'`。

### 3.2 Domain 型別

```ts
export enum ClientAccessMode { UI = 'UI', API_ONLY = 'API_ONLY' }
export enum OpenApiClientKind { ISSUER = 'ISSUER', INVESTOR = 'INVESTOR' }
export enum OpenApiScope {
  READ = 'READ', CLIENT_WRITE = 'CLIENT_WRITE', WALLET_WRITE = 'WALLET_WRITE',
  CMTAT_WRITE = 'CMTAT_WRITE', WITHDRAWAL_WRITE = 'WITHDRAWAL_WRITE', WEBHOOK_MANAGE = 'WEBHOOK_MANAGE',
}
export interface OpenApiPrincipal { partnerId: string; credentialId: string; custodyEntity: CustodyEntity; scopes: ReadonlySet<OpenApiScope> }
export interface OpenApiClientContext { clientId: string; kind: OpenApiClientKind; externalId: string; apiAccountId: string; defaultPolicyId: string }
```

### 3.3 認證契約

#### 3.3.1–3.3.2 Header 與簽章

| Header | 格式 |
|---|---|
| `X-Aegis-Api-Key` | `ak_…` |
| `X-Aegis-Api-Timestamp` | Unix 秒 |
| `X-Aegis-Api-Nonce` | 16–64，`[A-Za-z0-9_-]` |
| `X-Aegis-Api-Signature` | Base64 |

```
message = apiKey + timestamp + nonce + pathWithQuery + method + body
```

- `pathWithQuery` 含原樣 query；RSA-PSS／SHA-256／MGF1 SHA-256；簽章端 salt 32，驗證端 `RSA_PSS_SALTLEN_AUTO`。
- 測試用跨語言驗簽向量（Node 簽 Go 驗、Go 簽 Node 驗），不比對簽章 bytes。

#### 3.3.3 失敗回應

| 情況 | 回應 |
|---|---|
| 認證失敗（含 partner DISABLED） | `401 UNAUTHORIZED` |
| scope 不足 | `403 SCOPE_NOT_ALLOWED` |
| 路徑 clientId 未連結本 partner；連結 DISABLED 時的寫入與 client-scoped 查詢 | `404 NOT_FOUND` |
| 限流 | `429 RATE_LIMITED`＋`Retry-After` |

- 連結 DISABLED 後：client-scoped 端點都回 404；`GET /v1/requests…`、`GET /v1/operations/{id}` 仍可查該 client 的歷史；既有 operation 的 webhook 照送。

#### 3.3.4 `USER_API` 權限

`REQUEST_WALLET_CREATE`、`REQUEST_WALLET_UPDATE`、`REQUEST_WHITELISTED_WALLET_CREATE`、`REQUEST_WALLET_WITHDRAWAL`、`REQUEST_CMTAT`。

### 3.4 共用格式

```ts
type Ok<T> = { success: true; data: T };
type Page<T> = { items: T[]; nextCursor: string | null };
interface OpenApiErrorBody {
  success: false;
  data: {
    code: OpenApiErrorCode; message: string; referenceId?: string;
    retryable: boolean;                       // true：可以用同一個 referenceId 重試；false：不要原樣重試
    details?: Record<string, unknown>;
    result?: unknown;                         // 只有 409 DUPLICATE_REQUEST
  };
}
type CmtatTokenId = string;   // cmtat_tokens.id 十進位字串，也是 assetTypeId
type TokenAmountInput = string;   // 請求用：> 0、代幣單位、小數 ≤ decimals、無科學記號；乘上 10^decimals 後的整數 ≤ uint256 上限
type TokenAmount = string;        // 回應用：≥ 0（餘額、lockedAmount、totalSupply 可以是 0）
type OpenApiNetwork = 'CONFLUX_MAINNET' | 'CONFLUX_TESTNET' | 'ETHEREUM_SEPOLIA';
type HolderRef = { walletId: string } | { whitelistedWalletId: string };
```

#### 3.4.1 RequestView

```ts
interface OpenApiRequestView {
  referenceId: string | null; requestId: string; clientId: string; clientExternalId: string;
  operationType: 'WALLET_CREATE' | 'WALLET_UPDATE' | 'WHITELISTED_WALLET_CREATE' | 'CMTAT_DEPLOY' | 'CMTAT_CALL' | 'CRYPTO_WITHDRAWAL';
  cmtatAction: string | null;
  status: OpenApiRequestStatus; isFinal: boolean;
  createdAt: string; updatedAt: string;
  review: { aegisStage: 'SUPERVISOR' | 'AUTH_OFFICER' | 'AUTH_MANAGER' | 'AUTH_WALLET' | 'AUTH_ACCOUNT' | null };
  rejection: { by: 'CLIENT' | 'AEGIS'; code: 'TECHNICAL_RETRY' | null; reason: string | null } | null;
  // code = 'TECHNICAL_RETRY'：Aegis 因技術問題拒絕（§4.3.3），DQ 可以用新的 referenceId 重送；null：一般審核拒絕，不要自動重送
  failure: { code: string; message: string } | null;
  result: OpenApiRequestResult | null;
}
type OpenApiRequestResult =
  | { type: 'WALLET_CREATE'; walletId: string; address: string | null }
  | { type: 'WALLET_UPDATE'; walletId: string }
  | { type: 'WHITELISTED_WALLET_CREATE'; whitelistedWalletId: string }
  | { type: 'CMTAT_DEPLOY'; cmtatTokenId: CmtatTokenId; contractAddress: string; txHash: string | null }
  | { type: 'CMTAT_CALL'; txHash: string | null }
  | { type: 'CRYPTO_WITHDRAWAL'; txHash: string | null };
```

- `walletId`、`whitelistedWalletId` 來自 `operations.relations.createdItemId`；`updatedAt` = max(createdAt, 最後 review record 時間, execution record 時間)。
- **GET 一律回 `open_api_tracked_targets.last_view`（已提交的公開狀態）**。狀態只會往 rank 高的方向變（§3.7.1）；同一個非最終狀態內的進度（`review.aegisStage`、`updatedAt`）也會更新 `last_view`，但不另發 webhook。候選狀態怎麼推導見 §4.5。
- `isFinal` 只有 rank 5（COMPLETED、FAILED、REJECTED）為 true；`TIMEOUT` 的 `isFinal` 是 false。

#### 3.4.2 ClientRequestView

```ts
interface ClientRequestView {
  referenceId: string; clientRequestId: string; kind: 'ISSUER' | 'INVESTOR'; externalId: string;
  status: 'PENDING_AEGIS_REVIEW' | 'COMPLETED' | 'REJECTED';
  isFinal: boolean; createdAt: string; updatedAt: string;
  rejection: { reason: string | null } | null;
  result: { clientId: string } | null;   // COMPLETED 才有；COMPLETED 代表可以開始建錢包
}
```

### 3.5 端點

| # | Method | Path | Scope | 限 kind |
|---|---|---|---|---|
| 1 | GET | `/v1/me` | READ | — |
| 2 | GET | `/v1/clients`、`/v1/clients/{clientId}` | READ | — |
| 3 | POST | `/v1/client-requests` | CLIENT_WRITE | — |
| 4 | GET | `/v1/client-requests/{referenceId}` | READ | — |
| 5 | GET | `/v1/clients/{clientId}/wallets`、`…/{walletId}` | READ | — |
| 6 | POST | `/v1/clients/{clientId}/wallets` | WALLET_WRITE | — |
| 7 | POST | `/v1/clients/{clientId}/wallets/{walletId}/whitelisted-wallets` | WALLET_WRITE | — |
| 8 | GET | `/v1/clients/{clientId}/whitelisted-wallets`、`…/{id}` | READ | — |
| 9 | POST | `/v1/clients/{clientId}/whitelisted-wallets` | WALLET_WRITE | — |
| 10 | POST／GET | `/v1/clients/{clientId}/cmtat/terms-documents`、`…/{id}` | CMTAT_WRITE／READ | ISSUER |
| 11 | POST | `/v1/clients/{clientId}/cmtat/deployments` | CMTAT_WRITE | ISSUER |
| 12 | GET | `/v1/clients/{clientId}/cmtat/tokens`、`…/{cmtatTokenId}`、`…/{cmtatTokenId}/holders` | READ | ISSUER |
| 13 | POST | `/v1/clients/{clientId}/cmtat/tokens/{cmtatTokenId}/calls` | CMTAT_WRITE | ISSUER |
| 14 | POST | `/v1/clients/{clientId}/withdrawals` | WITHDRAWAL_WRITE | — |
| 15 | GET | `/v1/requests/{referenceId}`、`/v1/requests` | READ | — |
| 16 | GET | `/v1/operations/{operationId}` | READ | — |
| 17 | GET | `/v1/webhook-deliveries` | WEBHOOK_MANAGE | — |
| 18 | POST | `/v1/webhook-deliveries/{callbackId}/redeliver` | WEBHOOK_MANAGE | — |
| 19 | GET | `/healthz` | 不驗簽 | — |

- ISSUER 端點：路徑 client 不是 ISSUER → 404；`cmtatTokenId` 不屬於路徑 client → 404。
- `/v1/operations/{id}`：只回 `open_api_tracked_targets` 裡屬於本 partner 的 operation。

#### 3.5.1 `/v1/me`

```ts
interface MeResponse { partnerId: string; partnerName: string;
  credential: { apiKey: string; name: string; scopes: OpenApiScope[]; expiresAt: string };
  networks: OpenApiNetwork[]; kytRequiredNetworks: OpenApiNetwork[]; webhook: { url: string; enabled: boolean } | null }
```

#### 3.5.2 client

```ts
interface ClientView { clientId: string; kind: 'ISSUER' | 'INVESTOR'; externalId: string;
  clientType: 'COMPANY' | 'INDIVIDUAL'; displayId: string; name: string; linkState: 'ACTIVE' | 'DISABLED'; createdAt: string }
// GET /v1/clients?kind=&externalId=&limit=&cursor=
```

#### 3.5.3 錢包

```ts
interface CreateWalletRequest { referenceId: string; name: string; network: OpenApiNetwork; usage: string; whitelistedWalletIds?: string[] }
interface AddWhitelistedWalletsRequest { referenceId: string; whitelistedWalletIds: string[] }  // 1–20
interface WalletView { walletId: string; clientId: string; name: string; network: OpenApiNetwork; address: string;
  state: 'ACTIVE' | 'LOCKED'; /* WalletState.ACTIVATED → 'ACTIVE'、WalletState.LOCKED → 'LOCKED' */ usage: string; whitelistedWalletIds: string[];
  assets: { assetTypeId: string; symbol: string; decimals: number; cmtatTokenId: CmtatTokenId | null;
            onChainAmount: TokenAmount | null; confirmedAmount: TokenAmount; lockedAmount: TokenAmount }[] }
```

- 建錢包一律用 default policy。
- 加入白名單：來源錢包要屬於路徑 client；每個 id 要同 client、同鏈；呼叫 `WalletUpdateCommandService.createAddWhitelisted`（先取鎖、再重讀錢包、用**現行** policy 與 `whitelistedIds` 算聯集，§2.4）；全部已存在 → 422 `WALLET_UNCHANGED`；錢包有審核中的修改（domain 回 `ObjectHasBeenLockedError` 字串）→ 409 `WALLET_LOCKED_BY_PENDING_UPDATE`（`retryable: true`，並刪除本次的 PROCESSING 紀錄，不佔用 referenceId）。

#### 3.5.4 client 申請

```ts
type CompanyFields = { companyName: string; incorporationNo: string; contactEmail: string };
type IndividualFields = { firstName: string; lastName: string; birthday: string; countryOfCitizenship: string };
type CreateClientRequest = { referenceId: string; externalId: string } & (
  | ({ kind: 'ISSUER'; clientType?: 'COMPANY' } & CompanyFields)
  | ({ kind: 'INVESTOR'; clientType?: 'INDIVIDUAL' } & IndividualFields)
  | ({ kind: 'INVESTOR'; clientType: 'COMPANY' } & CompanyFields)
);
// 202 ClientRequestView；externalId 1–100、[A-Za-z0-9_.-]；只用來對應，不會用在 displayId 或任何檔案路徑
```

省略 `clientType` 時，`ISSUER` 預設為 `COMPANY`，`INVESTOR` 預設為 `INDIVIDUAL`。機構投資人須明傳 `INVESTOR + COMPANY`。依組合驗證必填欄位。未知值、明傳 null、`ISSUER + INDIVIDUAL`、缺欄位或混填另一種類型欄位，均在建單前回 `400 VALIDATION_ERROR`。本端點只收建戶資料。審核所需文件及收件方式另定。

申請通過欄位驗證後，將有效的 `clientType` 寫入 `open_api_client_requests.payload`。後台審核列表與明細顯示角色及客戶類型（公司發行方、個人投資人、機構投資人）。讀取 v0.6 申請時也按上述預設值解讀。簽章與 §4.3 的 requestHash 均使用送出的 body，重送時保留原本的欄位寫法。`externalId` 的唯一範圍仍為整個 partner，涵蓋三種組合。

#### 3.5.5 白名單錢包

```ts
interface CreateWhitelistedWalletRequest { referenceId: string; name: string; network: OpenApiNetwork; address: string }
// owner = SELF；KYT gate（§4.6）
interface WhitelistedWalletView { whitelistedWalletId: string; clientId: string; name: string; network: OpenApiNetwork; address: string; owner: 'SELF' }
```

#### 3.5.6 條款、部署

```ts
interface UploadTermsDocumentRequest { fileName: string; contentBase64: string; isPublic: boolean }  // 解碼後 ≤ 10,000,000 bytes、%PDF；201
interface DeployCmtatRequest { referenceId: string; walletId: string; name: string; symbol: string; decimals: number;
  tokenId: string; termsName: string; termsDocumentId: string; information: string }
```

CMTAT v3.0.0 Allowlist 部署使用發行方的 admin 錢包。九個部署欄位均必填。`termsDocumentId` 填同一發行方上傳條款 PDF 後取得的文件 ID。`information` 會公開上鏈。`decimals` 為 0–18 的整數，部署後固定。USDSN27 為 0，發行 100 單位的 `amount` 填 `"100"`。

#### 3.5.7 合約與持有人

```ts
interface CmtatTokenView { cmtatTokenId: CmtatTokenId; network: OpenApiNetwork; contractAddress: string;
  name: string; symbol: string; decimals: number; tokenId: string; information: string;
  terms: { name: string; uri: string; documentHash: string; termsDocumentId: string | null };
  deployerWalletId: string; deployTxHash: string;
  roles: { role: string; walletId: string | null; address: string | null; onChainGranted: boolean | null }[];
  onChain: { totalSupply: TokenAmount; paused: boolean; deactivated: boolean; allowlistEnabled: boolean } | null;
  createdAt: string }

interface CmtatHolderView { holder: HolderRef; holderClientId: string; holderName: string; address: string;
  isAllowlisted: boolean; isFrozen: boolean; balance: TokenAmount; frozenTokens: TokenAmount;
  availableBalance: TokenAmount; confirmedAmount: TokenAmount | null }
// GET …/holders?type=ISSUER_CUSTODY|ISSUER_EXTERNAL|INVESTOR_CUSTODY|INVESTOR_EXTERNAL&walletIds=&limit=1..25&cursor=
//   type 必填；DB 依 cursor 分頁後才讀鏈；每個持有人約 4 次 RPC，一頁最多約 100 次
//   重用 CmtatContractReader.holderState 與既有的代幣單位轉換
//   RPC 併發：在「每一次 RPC 呼叫」外層套整個 process 共用的 semaphore（OPEN_API_RPC_CONCURRENCY，預設 8）；
//   holderState 內部的 4 個呼叫各算一次，不是每個 holder 算一次
//   單頁延遲預算 10 秒：逾時時取消還在排隊、尚未發出的呼叫；已發出的呼叫等它真的結束才釋放名額，避免重試疊加
//   超過預算整頁回 502 CHAIN_READ_FAILED（retryable）
```

#### 3.5.8 合約操作

```ts
type CmtatCallRequest = { referenceId: string } & (
  | { action: 'SET_ALLOWLIST'; target: HolderRef; allowlistStatus: boolean }
  | { action: 'BATCH_SET_ALLOWLIST'; targets: HolderRef[]; allowlistStatus: boolean }
  | { action: 'MINT'; target: HolderRef; amount: TokenAmountInput }
  | { action: 'BATCH_MINT'; targets: (HolderRef & { amount: TokenAmountInput })[] }
  | { action: 'BURN'; target: HolderRef; amount: TokenAmountInput }
  | { action: 'BATCH_BURN'; targets: (HolderRef & { amount: TokenAmountInput })[] }
  | { action: 'BURN_AND_MINT'; from: HolderRef; burnAmount: TokenAmountInput; to: HolderRef; mintAmount: TokenAmountInput }
  | { action: 'GRANT_ROLE' | 'REVOKE_ROLE'; role: 'MINTER' | 'BURNER' | 'ALLOWLIST' | 'PAUSER' | 'ENFORCER' | 'ERC20ENFORCER'; walletId: string }
  | { action: 'SET_NAME' | 'SET_SYMBOL' | 'SET_TOKEN_ID' | 'SET_INFORMATION'; value: string }
  | { action: 'SET_TERMS'; termsName: string; termsDocumentId: string }
);
```

`GRANT_ROLE` 由發行方的 admin 簽署，每筆指派一個角色。`walletId` 為發行方名下接受授權的託管錢包。maker 取得 `MINTER`、`BURNER`、`ALLOWLIST`，另一個 custodian 錢包取得 `PAUSER`、`ENFORCER`、`ERC20ENFORCER`。每筆使用自己的 `referenceId`。

對照 `CmtatCallParamsInput`：`target`／`from` → `targetWalletId`／`targetWhitelistedWalletId`；`to` → `toWalletId`／`toWhitelistedWalletId`；MINT `amount` → `mintAmount`、BURN `amount` → `burnAmount`；`targets`（BATCH_SET_ALLOWLIST、BATCH_MINT）→ `batchTargets`；`targets`（BATCH_BURN）→ `burnTargets`；GRANT／REVOKE 的 `walletId` → `targetWalletId`；`value` → `textValue`；其餘同名；沒列到的設 null。對象規則見 §4.10 矩陣。

#### 3.5.9 提領

```ts
interface CreateWithdrawalRequest { referenceId: string; walletId: string; whitelistedWalletId: string;
  cmtatTokenId: CmtatTokenId; amount: TokenAmountInput; description: string }
```

- `walletId` 屬於路徑 client；`whitelistedWalletId` 屬於路徑 client、owner SELF、在 `walletId` 的 `whitelistedIds`（否則 422 `DESTINATION_NOT_IN_WALLET_WHITELIST`）。
- `cmtatTokenId` 的判斷改用 §4.10：`AllowedHolderService.isAllowedHolderClient(token.clientId, sourceWallet.clientId)` 為 true 才可以（token 屬於路徑 client 本身，或路徑 client 是同 partner、ACTIVE 的 INVESTOR 而 token 屬於 ACTIVE 的 ISSUER）；同 partner 的另一個 ISSUER、已停用的 investor、不同 partner → 404。
- 同時檢查：`walletId`、`whitelistedWalletId` 都屬於路徑 client，且 token、來源錢包、目的地三者同鏈。以上都在估 gas 與建 KV 任務之前完成。
- KYT gate（§4.6）；鏈上限制由 domain 的 `canTransfer`（`CmtatTransferNotAllowedError`）判斷。目的地要先由發行方 `SET_ALLOWLIST` 加進合約 allowlist（A7）。

#### 3.5.10 查詢與 webhook 補查

- `GET /v1/requests/{referenceId}`：依 target_type 回 RequestView 或 ClientRequestView；PROCESSING → 409 `REQUEST_IN_PROGRESS`（retryable）；UNKNOWN → 409 `REQUEST_STATE_UNKNOWN`（not retryable）；ABANDONED → RequestView（`status = FAILED`、`failure.code = REQUEST_ABANDONED`）；DONE(422) → 原錯誤 body。
- `GET /v1/requests?status=&operationType=&clientId=&createdAfter=&createdBefore=`：只列 DONE 且 202。
- `GET /v1/webhook-deliveries?status=`、`POST …/{callbackId}/redeliver`（同一則 1 分鐘一次；寫 activity log）。

### 3.6 後台 GraphQL（main）

```graphql
enum OpenApiScope { READ CLIENT_WRITE WALLET_WRITE CMTAT_WRITE WITHDRAWAL_WRITE WEBHOOK_MANAGE }
enum OpenApiKeySource { BOUND GENERATED }
enum OpenApiClientRequestApproval { MANUAL AUTO }
enum OpenApiLookupResult { FOUND NOT_FOUND ERROR }
enum OpenApiAnomalyKind { UNKNOWN KV_LINK_MISSING }
enum OpenApiRecommendedAction { LINK REJECT ABANDON }

type OpenApiAnomaly {                   # 列表只回 DB 摘要，不查 KV
  requestId: ID! partner: OpenApiPartner! referenceId: String! endpoint: String! targetType: String! targetId: ID!
  kind: OpenApiAnomalyKind!
  createdAt: DateTime!
}
type OpenApiAnomalyPage { items: [OpenApiAnomaly!]! nextCursor: String }
type OpenApiAnomalyDetail {
  anomaly: OpenApiAnomaly!
  requestBodyText: String               # 原始請求內容（JSON 字串；CLIENT_REQUEST 為 null）
  operationExists: Boolean!
  operationState: String
  isKvOp: Boolean
  localKvTaskExists: Boolean!
  kvTask: OpenApiLookupResult           # 要查 KV 時才有值（交易：GET /blockchain/task/:txId/status；建錢包：GET /wallet/:walletId/tasks）；查詢逾時 5 秒 → ERROR
                                        # 是否查、查哪一個，依請求保存的 endpoint 決定（operation 不存在、isKvOp 為 null 時也能診斷孤兒任務）：
                                        #   建錢包 → 錢包 API；提領、CMTAT 部署、CMTAT 合約操作 → 交易任務 API；加入白名單、建白名單錢包 → 不查
  kvTaskApprovalStage: String           # KV 回傳的任務內容中可讀到的審核／執行階段（只供參考）
  kvWallet: OpenApiLookupResult         # 只有 WALLET_CREATE：GET /wallet/:walletId/info
  recommendedAction: OpenApiRecommendedAction!
  checkedAt: DateTime!
}

input CreateOpenApiCredentialInput {
  partnerId: ID!
  name: String!                 # 1–100
  keySource: OpenApiKeySource!
  publicKeyPem: String          # BOUND 必填；GENERATED 不可帶
  ipAllowlist: [String!]!       # ≥ 1；IP 或 CIDR；不接受 0.0.0.0/0、::/0
  scopes: [OpenApiScope!]!      # ≥ 1
  expiresAt: DateTime!          # 一年以內
}
type CreateOpenApiCredentialResult {
  credential: OpenApiCredential!
  privateKeyPem: String         # 只有 GENERATED 才有，且只在這個 response 出現一次；server 不保存、不寫 log
}
type OpenApiWebhookSetResult { endpoint: OpenApiWebhookEndpoint! secret: String }  # secret 只在第一次建立或 rotate 時有值

type Query {
  openApiPartners: [OpenApiPartner!]!
  openApiCredentials(partnerId: ID!): [OpenApiCredential!]!
  openApiClientRequests(partnerId: ID!, state: String, kind: String): [OpenApiClientRequest!]!
  openApiClientLinks(partnerId: ID!, kind: String): [OpenApiClientLink!]!
  openApiAnomalies(partnerId: ID!, limit: Int, cursor: String): OpenApiAnomalyPage!   # VIEW_OPEN_API；DB 分頁摘要
  openApiAnomaly(requestId: ID!): OpenApiAnomalyDetail!                                # VIEW_OPEN_API；點開單筆才查 KV
  openApiWebhookEndpoint(partnerId: ID!): OpenApiWebhookEndpoint
}
type Mutation {
  createOpenApiPartner(name: String!): OpenApiPartner!                                   # MANAGE；custodyEntity 固定 HK；client_request_approval 預設 MANUAL
  setOpenApiPartnerClientRequestApproval(partnerId: ID!, mode: OpenApiClientRequestApproval!): OpenApiPartner! # APPROVE；寫 activity log；只影響之後送出的申請
  disableOpenApiPartner(partnerId: ID!, reason: String!): OpenApiPartner!                # APPROVE；條件 state = ACTIVE；之後該 partner 所有憑證的下一個請求即 401；寫 activity log
  enableOpenApiPartner(partnerId: ID!): OpenApiPartner!                                  # APPROVE；條件 state = DISABLED
  createOpenApiCredential(input: CreateOpenApiCredentialInput!): CreateOpenApiCredentialResult!  # MANAGE
  approveOpenApiCredential(id: ID!): OpenApiCredential!                                  # APPROVE；≠ createdBy
  revokeOpenApiCredential(id: ID!, reason: String!): OpenApiCredential!                  # MANAGE
  approveOpenApiClientRequest(id: ID!): OpenApiClientRequest!                            # REVIEW_OPEN_API_CLIENT_REQUEST
  rejectOpenApiClientRequest(id: ID!, reason: String!): OpenApiClientRequest!            # REVIEW_OPEN_API_CLIENT_REQUEST
  disableOpenApiClientLink(clientId: ID!, reason: String!): OpenApiClientLink!           # MANAGE
  linkOpenApiRequest(requestId: ID!, note: String!): Boolean!       # APPROVE；server 重查 partner、請求狀態，operation 存在才允許（§4.3.3）
  abandonOpenApiRequest(requestId: ID!, note: String!): Boolean!    # APPROVE；server 在鎖內重查，只有請求是 UNKNOWN 且 operation 不存在時允許（§4.3.3）
  rejectOpenApiStuckOperation(requestId: ID!, note: String!): Boolean!  # APPROVE；只限 KV_LINK_MISSING；走既有 ReviewOperationService 的拒絕（操作者需有當前審核群組的權限）
  setOpenApiWebhookEndpoint(partnerId: ID!, url: String!): OpenApiWebhookSetResult!      # MANAGE；第一次建立時 enabled=false 並回 secret
  rotateOpenApiWebhookSecret(partnerId: ID!): OpenApiWebhookSetResult!                   # MANAGE
  setOpenApiWebhookEnabled(partnerId: ID!, enabled: Boolean!): OpenApiWebhookEndpoint!   # MANAGE
}
```

| Action | 指派 |
|---|---|
| `VIEW_OPEN_API`、`MANAGE_OPEN_API` | `ADMIN_SUPERVISOR`、`ADMIN_ACCOUNT_MANAGER` |
| `APPROVE_OPEN_API` | `ADMIN_SUPERVISOR` |
| `REVIEW_OPEN_API_CLIENT_REQUEST` | `ADMIN_SUPERVISOR`、`ADMIN_COMPLIANCE` |

- **custody entity**：所有 query／mutation 都要求 admin 的 `custodyEntities` 包含該 partner 的 `custody_entity`（v1 = HK），包括 partner、憑證、申請（含個資）、連結、webhook、異常請求處理。
- **GENERATED 金鑰**：server 用 `crypto.generateKeyPairSync('rsa', { modulusLength: 3072 })`，只存公鑰；私鑰只在 mutation response 回傳一次；後台 UI 提供下載 `.pem` 並警示「關閉後無法再取得」；activity log 只記 fingerprint。建議優先用 BOUND。
- **併發**：狀態轉移都是條件式 UPDATE；核准憑證時 `SELECT … FROM open_api_partners WHERE id = $1 FOR UPDATE` 串行化 ACTIVE 上限 3；撤銷 `WHERE state <> 'REVOKED'`。

### 3.7 狀態與 webhook

#### 3.7.1 狀態、rank

| status | 來源 | rank |
|---|---|---|
| `PENDING_CLIENT_REVIEW` | `REVIEWING` + `USER`（API_ONLY 不會出現） | 1 |
| `PENDING_AEGIS_REVIEW` | `REVIEWING` + `ADMIN_*`；client 申請 PENDING | 2 |
| `PROCESSING` | `APPROVED`／`EXECUTING`；**CMTAT_DEPLOY／CMTAT_CALL 的 operation 已 COMPLETED，但 AEG-43 的 `isOperationSettled(operationId)` 尚未為 true** | 3 |
| `TIMEOUT` | `TIMEOUT`（KV 任務逾時）。**不是最終狀態**：之後可能變成 COMPLETED／FAILED | 4 |
| `COMPLETED`／`REJECTED`／`FAILED` | 最終 | 5 |

- 對外狀態只往 rank 高的方向變（已提交的 `last_status`／`last_view`）。候選狀態的推導見 §4.5：**有完成或失敗的證據（execution record、AEG-43 已結算）時，以證據為準**，即使資料庫的 state 被既有行為改成 TIMEOUT（A24，不修）也一樣推導為 COMPLETED／FAILED；TIMEOUT 之後晚到的結果會正常推進。
- `COMPLETED`、`REJECTED`、`FAILED` 同為 rank 5、彼此互斥；同一個對象只會有一個。

#### 3.7.2 Webhook

```
POST {url}
Content-Type: application/json
X-Aegis-Callback-Id: <events.id>
X-Aegis-Signature: hex(HMAC-SHA256(secret, rawBody))
```

```ts
type WebhookPayload =
  | { callbackId: string; seq: string; eventType: 'OPERATION_STATUS_CHANGED'; occurredAt: string; request: OpenApiRequestView }
  | { callbackId: string; seq: string; eventType: 'CLIENT_REQUEST_STATUS_CHANGED'; occurredAt: string; clientRequest: ClientRequestView };
```

- 語意（A23）：**落庫單調、送達可亂序**。同一個對象落庫的事件 rank 與 `seq` 都只增不減；但派送是併發的、有重試與人工重送，**送達順序不保證**（例如 PROCESSING 可能比 COMPLETED 晚到）。接收端：用 callbackId 去重；每個對象只在收到的 `seq` 大於已記錄的 `seq` 時才更新；不確定時呼叫查詢 API。最終狀態一定會落庫並嘗試派送；6 次失敗 FAILED，可以補查與重送。至少送一次，不保證接收端一定收到。

### 3.8 錯誤碼

| HTTP | code | retryable | 來源 |
|---|---|---|---|
| 400 | `VALIDATION_ERROR` | false | 驗證 |
| 401／403／404／429 | 見 §3.3.3 | 429 為 true | guard |
| 409 | `DUPLICATE_REQUEST` | false | §4.3 |
| 409 | `REFERENCE_ID_CONFLICT` | false | §4.3 |
| 409 | `REQUEST_IN_PROGRESS` | true | §4.3、§4.4 |
| 409 | `REQUEST_STATE_UNKNOWN` | false | §4.3 |
| 409 | `WALLET_LOCKED_BY_PENDING_UPDATE` | true | object lock |
| 413 | `PAYLOAD_TOO_LARGE` | false | parser |
| 422 | `ACTION_NOT_ALLOWED`、`NETWORK_NOT_SUPPORTED`、`KYT_NOT_AVAILABLE_FOR_NETWORK`、`KYT_FAILED`、`CLIENT_EXTERNAL_ID_EXISTS`、`MASTER_KEY_NOT_PROVISIONED`、`WHITELISTED_WALLET_INVALID`、`WALLET_UNCHANGED`、`DESTINATION_NOT_IN_WALLET_WHITELIST`、`REQUESTER_NOT_ELIGIBLE`、`EXCEEDS_DAILY_LIMIT`、`INSUFFICIENT_GAS_FEE`、`INSUFFICIENT_BALANCE`、`WALLET_LOCKED`、`CMTAT_TRANSFER_NOT_ALLOWED`、`CMTAT_ROLE_NOT_ASSIGNED`、`CMTAT_SIGNER_ROLE_MISSING`、`CMTAT_ROLE_OPERATION_IN_PROGRESS`、`CMTAT_TARGET_NOT_ALLOWLISTED`、`CMTAT_INVALID_TARGET`、`CMTAT_STATE_UNCHANGED`、`CMTAT_VALIDATION_FAILED`、`SYMBOL_ALREADY_EXISTS`、`TERMS_DOCUMENT_NOT_FOUND`、`FEE_ESTIMATE_REJECTED` | false | domain（佔用 referenceId） |
| 502 | `FEE_ESTIMATE_UNAVAILABLE`（估 gas 時 RPC 或 `getFeeData` 失敗）、`KYT_UNAVAILABLE`（KYT 服務錯誤、沒有評分、`UNDETERMINED`）、`CHAIN_READ_FAILED`、`FILE_STORAGE_UNAVAILABLE` | true | 副作用前的暫時性錯誤（不佔用） |
| 502 | `KV_UNAVAILABLE` | false | 結果可能不明 → UNKNOWN |
| 500 | `INTERNAL_ERROR` | false | 結果不明時為 UNKNOWN |

- `domain-error-map.ts` exhaustive（新增 domain 錯誤沒加對照會讓測試失敗）。
- `REQUEST_ABANDONED` 不是 HTTP 錯誤碼，而是作廢請求的 `failure.code`（§4.3.3）。`KytFailedErrorDto` 不含分數，`details` 只放供應商與 log id。

---

## 4. 核心業務邏輯與流程

### 4.1 Raw body

條款端點 15 MB、其他 1 MB；`verify` 存 `req.rawBody`；ingress 16 MB。

### 4.2 驗證管線

```
OpenApiAuthGuard
 1. header 齊全
 2. credential（每次查 DB，不快取）＋ partner
 3. credential ACTIVE、未過期、partner ACTIVE
 4. req.ip ∈ ip_allowlist（net.BlockList）
 5. |now − ts| ≤ 300
 6. nonce 格式
 7. RSA-PSS 驗 message（req.originalUrl）
 8. INSERT nonce（衝突 → 401）
 9. rate limit（每 credential，記憶體）
10. scope
11. principal（含 partner.custody_entity）；last_used_at 每分鐘最多寫一次
ClientScopeGuard
12. link（client_id, partner_id）存在且 ACTIVE；ISSUER 端點檢查 kind
13. req.openApiClient
```

### 4.3 冪等與結果不明的處理

**requestHash** = `sha256hex(method + "\n" + 實際路徑（含 clientId、cmtatTokenId 等 path 參數，不含 query） + "\n" + canonicalJson(body 去掉 referenceId))`。canonicalJson：key 依字母排序、無空白。不含任何 header（nonce、簽章、timestamp）。同一個 body 送到不同路徑視為不同內容。

#### 4.3.1 CLIENT_REQUEST（純 DB，不會有 UNKNOWN）

```
單一 transaction：
  INSERT open_api_requests(state = 'DONE', target_type = 'CLIENT_REQUEST', target_id = 新 uuid, …)
    ON CONFLICT (partner_id, reference_id) DO NOTHING
  沒插入 → 依既有紀錄回 409（hash 不同 → REFERENCE_ID_CONFLICT；相同 → DUPLICATE_REQUEST）
  INSERT open_api_external_ids … ON CONFLICT DO NOTHING
  沒插入 → 同一個 transaction 內把 open_api_requests 記成 http_status 422（CLIENT_EXTERNAL_ID_EXISTS），COMMIT 後回 422
  否則 INSERT open_api_client_requests(id = target_id, approval_mode = partner 當下設定, state = PENDING)、
       open_api_tracked_targets、第一筆 webhook 事件；open_api_requests 記 202 與 response_body
COMMIT
```

#### 4.3.2 OPERATION

```
1. 驗證（400 不建紀錄）
2. targetId = uuidv4()（同時是 operation id 與 KV txId）
3. INSERT open_api_requests(state = PROCESSING, version = 0) ON CONFLICT (partner_id, reference_id) DO NOTHING
   已存在 → 依 hash／state 回 409
4. 需要角色佔用時（§4.4）：取得佔用；取不到 → DELETE 自己這筆（WHERE id AND version = 0）並回 409 REQUEST_IN_PROGRESS（retryable）
5. 呼叫 domain：options = { operationId: targetId, kvTxId: targetId, walletId: targetId（WALLET_CREATE）, kytPolicy（§4.6 矩陣）,
     insertGuard: 在寫 operation 的 transaction 內 `SELECT state FROM open_api_requests WHERE id = 本次請求 FOR UPDATE`，state 不是 PROCESSING／UNKNOWN 就丟 AbandonedRequestError（§4.3.3） }
6. 結果（皆以 UPDATE … WHERE id = $1 AND version = $v，成功才 version += 1）：
   a. 成功 → DONE(202)；同一個 transaction 寫 open_api_tracked_targets；釋放角色佔用
   b. preSideEffectErrors 中的業務錯誤 → DONE(422)；釋放佔用
      preSideEffectErrors 中的暫時性錯誤 → DELETE（502，retryable）；釋放佔用
   c. 其他（例外、KV 錯誤、逾時）→ 查 operations WHERE id = targetId
        存在 → DONE(202)＋同 transaction 寫 tracked target；釋放佔用（若 operation 缺 KV 關聯，會出現在異常列表，§4.3.3）
        不存在 → UNKNOWN（502 KV_UNAVAILABLE 或 500，retryable = false）；**保留角色佔用**；觸發告警
```

- `preSideEffectErrors`：每個 request wrapper 明列，並用單元測試證明它們都在第一個外部副作用之前回傳。`WalletTransactionOperationServiceBase.create` 的 `validate`、`kytCheck`（含 STRICT）、`feeEstimate`、`canInitWalletTransaction` 錯誤屬於；`initKvTask`（先呼叫遠端、帶 txId，再寫本地 `kv_tasks`）的錯誤與之後的錯誤都不屬於。`WalletCreateCommandService.create` 在 commit operation 之前的錯誤屬於；之後一律走 6c。
- **清理程序**（open-api 每 5 分鐘）：PROCESSING 超過 10 分鐘 → 依 6c 判斷（version 條件更新，不覆蓋 handler 晚到的結果）；轉成 UNKNOWN 時觸發告警。

#### 4.3.3 異常請求的人工處理（A26、A27）

**前提**分成兩部分：
- **程式規則（已查證）**：KV 的交易與建錢包任務都要求 `Supervisor` 角色核准（keyvault-api-service `pkg/approvaltask/type.go:486–496`）；核准紀錄湊齊必要角色後任務才進入待執行（`service.go:513`、`:318`），離線 batch 也只撿已核准的任務；停在待審的任務有每分鐘的逾時程序（`main.go:236`）。KV 的核准 API 本身不檢查 av-api 有沒有對應的 operation。
- **部署條件（Allen 11/23 確認，A27）**：KV 的 API key 只有 av-api 持有。av-api 內只有 main 的 `supervisorKvTaskReview`（在 Supervisor 核准 operation 之後）會送 Supervisor 核准；open-api 的 KV client 用 `kv-client-guard` 擋掉所有核准 API。Step 5 部署時把「正式環境持有 KV key 的服務＝av-api 的 main 與 open-api」記進部署紀錄。

兩者成立時，「KV 有任務、av-api 沒有 operation」的孤兒任務拿不到 Supervisor 核准，不會被簽章或廣播，最後逾時，不需要處理。

**作廢與晚到建單的互斥**（DB 層，不靠人看警示）：
- open-api 建 operation 時，domain 在寫 operation 的 transaction 內呼叫 `insertGuard`：鎖住本次的 `open_api_requests` 列並重查 state；已是 ABANDONED → rollback，不寫 operation、不鎖資產（KV 若已有任務，就是無害的孤兒）。
- `abandonOpenApiRequest` 在一個 transaction 內：鎖住同一列 → 重查 state 必須是 UNKNOWN → 重查 operation（id = targetId）必須不存在 → 改成 ABANDONED。
- 兩邊鎖同一列，先 commit 的贏：作廢先 commit，舊的建單一定失敗；建單先 commit，作廢時會看到 operation 已存在而被拒（改用 `linkOpenApiRequest`）。最多只會有一張可執行的 operation。

**異常的兩種類型**（後台「Open API 異常請求」頁）：

| kind | 條件 |
|---|---|
| `UNKNOWN` | `open_api_requests.state = UNKNOWN` |
| `KV_LINK_MISSING` | `target_type = OPERATION`、請求是 DONE、operation 存在且 **`isKvOp = true`**、仍是 `REVIEWING`，但建立超過 10 分鐘仍沒有本地 `kv_tasks`。`WALLET_UPDATE`、`WHITELISTED_WALLET_CREATE` 這類 `isKvOp = false` 的 operation 本來就沒有 KV 關聯，**不會列入** |

- 列表只回 DB 分頁摘要；點開單筆（`openApiAnomaly`）才即時查 KV（逾時 5 秒，結果 `FOUND`／`NOT_FOUND`／`ERROR`）。要不要查、查哪個 KV API，依請求保存的 endpoint 決定（見 §3.6 註解），不依賴 operation 是否存在。按下處理動作時，server 會再重查一次並把當下的診斷存進 `diagnostics`。

**營運人員的處理**（`APPROVE_OPEN_API`，HK；每個動作寫 activity log）：

| 診斷 | 動作 | 對外結果 |
|---|---|---|
| operation 存在（不論有沒有 KV 關聯、`isKvOp` 為何） | `linkOpenApiRequest` | 請求變 DONE（202），補登 tracked target 並 project；DQ 看到 operation 的真實狀態 |
| operation 存在、缺 KV 關聯（`KV_LINK_MISSING`） | 在異常頁按「技術性拒絕」（`rejectOpenApiStuckOperation`：以操作者身分走既有的 `ReviewOperationService` 拒絕，權限與審核規則相同，並記 `resolution = TECHNICAL_REJECT`） | operation REJECTED → DQ 收到 `REJECTED`，`rejection.code = 'TECHNICAL_RETRY'`（唯一的最終狀態），用新的 referenceId 重送 |
| operation 不存在（不論 KV 有沒有任務） | `abandonOpenApiRequest` | 請求作廢 → DQ 收到 `FAILED`（`failure.code = REQUEST_ABANDONED`） |

- 「拒絕」與「作廢」不會用在同一筆請求：作廢只在 operation 不存在時允許，所以每個對象只會有一個最終狀態（另有 DB 的部分唯一索引保證每個對象只有一筆 rank 5 事件）。
- `KV_LINK_MISSING` 的請求在拒絕前就已經是 DONE、有 tracked target，拒絕後由正常的 project 路徑發出 `REJECTED`；營運人員中途離開也不會遺失（它仍在 REVIEWING，會留在待審清單與異常列表）。

**作廢（ABANDONED）的效果**：
- `open_api_requests.state = ABANDONED`、`resolution = ABANDONED`；釋放角色佔用。
- 對外：`GET /v1/requests/{referenceId}` 回 RequestView，`status = FAILED`、`failure.code = 'REQUEST_ABANDONED'`、`result = null`；登記 tracked target、直接提交這個公開狀態並送 webhook。
- 同一個 referenceId 重送 → `409 DUPLICATE_REQUEST`，`data.result` 是這份 FAILED 的回應（retryable = false）。**referenceId 永不釋放**，DQ 用新的 referenceId 重送。

**告警**：請求變成 UNKNOWN 時寫結構化 error log（`open_api.request_unknown`，含 partner、referenceId、endpoint）供 Grafana 告警，並寄信給 `OPEN_API_ALERT_EMAIL`。

### 4.4 角色指派的佔用與防超額的範圍（A12）

- **只有 `GRANT_ROLE`／`REVOKE_ROLE` 需要佔用**（避免同時指派讓 maker／custodian 分組失效；domain 的分組檢查只看得到已寫入的 pending operation，而 operation 要等 KV 回應後才寫入）：
  - key `cmtat-roles:<cmtatTokenId>`；`INSERT … ON CONFLICT (key) DO NOTHING`。
  - 已被佔用時：若持有者的 `open_api_requests` 是 DONE 或已不存在 → 用條件式 UPDATE 接手；否則 409 `REQUEST_IN_PROGRESS`（retryable，不佔用 referenceId）。
  - 釋放時機見 §4.3.2（operation 寫入後才釋放；UNKNOWN 期間保留，直到 `linkOpenApiRequest` 或 `abandonOpenApiRequest`）。不設時間到期。
  - **有效角色檢查**：佔用只保護「建單」這段。operation 寫入後，domain 的分組檢查靠 `findPendingRoleOperations`，它只看 REVIEWING／APPROVED／EXECUTING。open-api 路徑傳 `includeUnsettled: true`，改用 `deriveCandidate`（§4.5）判斷同一 token 的 GRANT／REVOKE：候選是 `PENDING_CLIENT_REVIEW`、`PENDING_AEGIS_REVIEW`、`PROCESSING`（含已完成但角色還沒寫回）或沒有結論的 `TIMEOUT` → 視為進行中；候選是 `REJECTED`、`FAILED`、已結算的 `COMPLETED` → 不算，即使資料庫 state 被既有行為改成 TIMEOUT（A24）。六個相容的角色指派仍可同時排隊審核。
- **加入白名單**：不另外加鎖，靠既有 object lock，且「先取鎖、再重讀錢包、再算聯集」（`createAddWhitelisted`，§2.4）；這樣第一筆核准釋鎖後，第二筆才拿到鎖並讀到最新的 `whitelistedIds` 與 policy，兩筆新增的地址都會保留。拿不到鎖 → 409 `WALLET_LOCKED_BY_PENDING_UPDATE`（retryable，刪除本次 PROCESSING 紀錄）。
- **不保證**以下情況在建立時就擋下（A12），結果如下（寫進 DQ 文件）：

  | 情況 | 受理 | 上鏈 | 帳務 |
  |---|---|---|---|
  | 同一個持有人錢包同時兩筆 BURN | 兩筆都可能受理（CMTAT call 不鎖代幣數量） | 兩筆合計**超過**鏈上可用量時，後上鏈的那筆 revert → FAILED；合計足額時，**其他執行條件（gas、凍結、allowlist、審核結果等）都成立的話**兩筆都可以成功 | 失敗那筆的代幣本金不動；gas 依實際廣播與 receipt 的 fee 由簽名錢包（maker）支付 |
  | 先 BURN 再提領（同一錢包） | 都可能受理 | 合計超過可用量時，後上鏈的那筆失敗；足額時，其他執行條件成立的話都可以成功 | 提領失敗時，提領鎖住的數量解除；提領的 gas 依 receipt 由投資人錢包支付 |
  | 先提領再 BURN | 提領鎖住數量；BURN 受理時檢查「剩餘可用量」 | 剩餘不足 → BURN 受理即被擋；剩餘足額 → 其他執行條件成立的話兩筆都可以成功 | — |
  | 已受理但未廣播就失敗 | — | 沒有上鏈 | 鎖住的數量解除，不扣 gas |

- CMTAT 的 USD 金額是 0，client 每日上限不影響 CMTAT 操作與 CMTAT 提領。

### 4.5 對外狀態與 Webhook

**兩層分開**：
- **候選狀態**：每次需要時從資料庫的紀錄推導（`deriveCandidate(op)`）。
- **公開狀態**：已提交在 `open_api_tracked_targets` 的 `last_status`／`last_view`。GET 與 webhook 只讀公開狀態。

```
deriveCandidate(op) → { status, view }      // 依序判斷，先命中先用
  1. 有拒絕紀錄（client 或 admin reject）                         → REJECTED
  2. CMTAT_DEPLOY／CMTAT_CALL 且 AEG-43 isOperationSettled = true   → COMPLETED（result 依結算後資料組）
     其他 type 且 execution record 成功（或 state = COMPLETED）      → COMPLETED
  3. execution record 失敗或 state = FAILED                        → FAILED
  4. CMTAT 的 operation state = COMPLETED 但尚未結算                 → PROCESSING
  5. state = TIMEOUT                                                → TIMEOUT
  6. state = APPROVED／EXECUTING                                    → PROCESSING
  7. state = REVIEWING                                              → PENDING_CLIENT_REVIEW／PENDING_AEGIS_REVIEW
  → 第 2、3 步以證據為準，所以資料庫 state 被既有行為改成 TIMEOUT（A24）也能推導出真正結果

project(target)：                                   // 唯一會改公開狀態的地方
  BEGIN
    c = deriveCandidate(op)；r = rank(c.status)
    UPDATE open_api_tracked_targets
       SET last_rank = r, last_status = c.status, last_view = c.view, final_event_stored = (r = 5),
           timeout_since = CASE WHEN r = 4 THEN coalesce(timeout_since, now()) ELSE timeout_since END
     WHERE target_type = $1 AND target_id = $2 AND last_rank < r
    影響 1 列 → INSERT open_api_webhook_events(status = c.status, rank = r, payload) ON CONFLICT DO NOTHING
    影響 0 列 → UPDATE … SET last_view = c.view, last_view_version = c.version
                 WHERE … AND last_rank = r AND r < 5 AND last_status = c.status AND last_view_version < c.version
      → 只更新 last_view（例如 Supervisor → Auth Wallet 的審核進度），不發 webhook
    c.version（同一 rank 內單調遞增）= 該 operation 的 review record 筆數 × 2 ＋（有 execution record ? 1 : 0）；
      不使用時間戳比較，避免同一毫秒的兩次更新分不出先後；rank、status 與版本在同一句條件更新裡判斷
  COMMIT
```

- **CLIENT_REQUEST 的候選狀態**：`PENDING` → `PENDING_AEGIS_REVIEW`（rank 2）、`APPROVED` → `COMPLETED`（rank 5）、`REJECTED` → `REJECTED`（rank 5）；payload 用 `{ eventType: 'CLIENT_REQUEST_STATUS_CHANGED', clientRequest: ClientRequestView }`。

- **登記時就提交第一個狀態**：tracked target 一律在建單或確認的同一個 transaction 內登記（API 的 6a／6c、`linkOpenApiRequest`、client 申請、`createByAdmin`），並在同一個 transaction 裡執行一次 `project`，寫入第一個有效狀態與第一筆 outbox 事件。所以 202 回應之後，GET 就有值，不會出現「已登記但沒有狀態」。
- **作廢的請求**：`abandonOpenApiRequest` 直接寫公開狀態 FAILED（`failure.code = REQUEST_ABANDONED`、rank 5）與事件；作廢後不可能再建出 operation（§4.3.3 的互斥）。
- **OutboxListener**（兩個 process）：收到 CREATED、CLIENT_APPROVED、ADMIN_REVIEW_GROUP_STARTED、CLIENT_REVIEW_GROUP_STARTED、CLIENT_REJECTED、ADMIN_REJECTED、ADMIN_APPROVED、FAILED、COMPLETED、TIMEOUT，以及 AEG-43 的「已結算」事件（只用來提早喚醒）→ tracked 存在就 `project`。
- **Reconciler**（open-api）：
  - 每 5 分鐘：`final_event_stored = false` 且（不是 TIMEOUT，或 TIMEOUT 未滿 30 天）的 target，逐筆 `project`。
  - 每天一次：TIMEOUT 已滿 30 天、仍未最終的 target，分頁（每批 200）逐筆 `project`，不設期限。
  - 涵蓋：listener 漏寫（commit 後、emit 前中斷）、舊待審單、零事件、後台單、TIMEOUT 之後的晚到結果、第 31 天以後才完成且通知遺失。
- **Dispatcher**（open-api 每 5 秒）：短 transaction 領取（lease_token、lease_until = now() + 60s、FOR UPDATE SKIP LOCKED、只取 enabled 的 partner、LIMIT 20）→ transaction 外 POST（併發 5、timeout 10 秒、redirects 0、DNS 綁定 IP）→ 用 `WHERE id AND lease_token` 條件更新結果；2xx → DELIVERED；否則退避 [1m,5m,15m,1h,6h]，第 6 次 FAILED。
- 送達可亂序（A23）；「對方收到但派送器沒回寫」會重送，DQ 用 callbackId 去重、用 `seq` 條件更新。
- URL、secret、停用、換 URL：建立時停用、secret 只在第一次建立或 rotate 回傳一次；只接受 https；DNS 解析一次並綁定 IP；redirects 0。

### 4.6 KYT（A9、A21、A24）

- provider 的值一律用共用 enum `KytProvider`（實際值是大寫 `'BEOSIN'`、`'ELLIPTIC'`，`kyt-provider.ts`）；不要寫字串常數。
- **環境政策矩陣**：

  | `OPEN_API_REQUIRE_KYT` | 網路在 Beosin 對照表 | open-api 的處理 | domain 的 `kytPolicy` |
  |---|---|---|---|
  | true（正式） | 有 | 呼叫 domain | `STRICT` |
  | true（正式） | 沒有（目前的 Conflux） | 422 `KYT_NOT_AVAILABLE_FOR_NETWORK`，不呼叫 domain | — |
  | false（dev／stg） | 有或沒有 | 呼叫 domain | `DEFAULT`（既有行為：沒評分時放行） |

- **啟動檢查（依部署環境）**：
  - 正式環境（`NODE_ENV=production`）：`OPEN_API_REQUIRE_KYT` 必須是字串 `true`，且 `KYT_PROVIDER` 等於 `KytProvider.BEOSIN`；漏設、`false`、其他字串、provider 不符 → open-api 不 ready。
  - 非正式環境：`OPEN_API_REQUIRE_KYT` 必須明確設成 `true` 或 `false`（漏設或非法字串 → 不 ready）；`false` 時使用 DEFAULT 政策。
- **STRICT 判斷**（domain 的 opt-in，前台不受影響）：套用在白名單錢包建立（約第 103 行）與 base 的 `kytCheck`（約第 121 行）。在那一次 KYT 查詢之後、建 operation 與 KV 之前：

  | KYT 結果 | STRICT 的處理 |
  |---|---|
  | 有 logId、`PASSED` | 放行 |
  | `FAILED` | 422 `KYT_FAILED` |
  | 沒有分數、`UNDETERMINED`、供應商錯誤、查詢回 null | 502 `KYT_UNAVAILABLE`（retryable） |

- **已知限制（A24，不修）**：Beosin service 用真假值判斷分數，分數剛好是 0 時會被當成沒有分數；在 STRICT 下得到 502，不會被放行。
- **驗收分開做**：
  - 正式 gate：用 `OPEN_API_REQUIRE_KYT=true`＋測試替身的 Beosin，測未支援網路 422、0 分／null／UNDETERMINED／服務故障 502、FAILED 422、有 logId 的 PASSED 放行；都不建 operation（除了 PASSED）。
  - dev 成功路徑：`OPEN_API_REQUIRE_KYT=false`、`DEFAULT` 政策（Conflux testnet 沒評分也放行）。
  - 正式啟用：以真實 Beosin 對 Conflux eSpace 的評分與 logId 驗收（拿到 chain id 後）。

### 4.7 client 建立（A5、A20、A22）

```
open-api：POST /v1/client-requests → §4.3.1（單一 transaction；approval_mode = partner 當下設定）

ClientRequestApprovalService.approve(requestId, actor)   // main；人工與 AUTO 共用
  BEGIN
    req = SELECT … FOR UPDATE；state ≠ PENDING → 結束（另一方已處理）
    建 client 資料（拆出來的第一段，沿用 CompanyMutateRepo／IndividualMutateRepo，同一個 QueryRunner）：
      clientType = COMPANY（ISSUER 或 INVESTOR）→ companies(name, incorporationNo, email = contactEmail) + clients(type COMPANY)
      clientType = INDIVIDUAL（INVESTOR）→ individuals(firstName, lastName, birthday, countryOfCitizenship, email = null, mobile = null) + clients(type INDIVIDUAL)
      共同：custodyEntity = partner.custody_entity（HK）、groupUsage = EXTERNAL、displayId = 'API-' + 10 碼 base32 隨機（衝突重產）、
            accessMode = API_ONLY、requesterId = actor.adminId（AUTO 為 null）——兩者都明確傳給 mutate repo，不靠 DB 預設（§2.4）
    API 帳號（UserMutateRepo，同一個 QueryRunner）：
      accounts(role = USER_API, email = 'api+<clientId>@open-api.invalid', firstName = 'API', lastName = displayId,
               mobile = null, hashedPassword = argon2(random 32 bytes), hasSetPassword = false, tfaSecret = null,
               createdBy = actor.adminId 或 null) + users(id = account.id, clientId)
    固定 policy（PolicyMutateRepo.insert，同一個 QueryRunner）：
      requesters = [{ userId: apiAccount.id, dailyLimit: 'FULL' }]、approverGroups = []、viewerIds = []
    INSERT open_api_partner_clients(kind = req.kind, external_id, api_account_id, default_policy_id, state = ACTIVE)
    UPDATE req SET state = APPROVED, client_id, reviewed_by = actor.adminId, auto_approved = (actor 是系統)（條件式，version）
    webhook 事件 COMPLETED（tracked target 條件更新）
  COMMIT

AutoApprovalWorker（main，setInterval 每 30 秒；不使用 ScheduleModule）
  每一輪最多處理 20 筆，每筆一個 transaction：
    BEGIN
      SELECT … FROM open_api_client_requests
       WHERE state = 'PENDING' AND approval_mode = 'AUTO' AND auto_next_attempt_at <= now() AND auto_attempts < 5
       ORDER BY auto_next_attempt_at, created_at LIMIT 1 FOR UPDATE SKIP LOCKED   // 領取並持有這一列的鎖直到 COMMIT
      SAVEPOINT provision
      approve 的內容（同上）
      成功 → COMMIT
      失敗 → ROLLBACK TO SAVEPOINT provision；在同一個 transaction 內：
             auto_attempts = auto_attempts + 1、auto_last_error = $err、
             auto_attempts < 5 → auto_next_attempt_at = now() + [1m, 5m, 30m, 2h][auto_attempts − 1]
             auto_attempts = 5 → auto_next_attempt_at 保持原值（欄位維持合法），worker 依 auto_attempts < 5 不再領取
             COMMIT
  → 失敗紀錄與鎖在同一個 transaction，另一個 worker 不會在退避落庫前重新領到；一直失敗的申請被往後排，不擋後面的申請
  auto_attempts = 5 → 後台標示「自動核准失敗，請人工處理」，人工可以核准或拒絕；不會有第 6 次自動執行

rejectOpenApiClientRequest（人工；AUTO 的申請在自動核准前也可以拒絕）：
  條件式 state = PENDING → REJECTED；DELETE open_api_external_ids；webhook 事件 REJECTED
```

- process 在申請 commit 之後重啟：AUTO 申請由 worker 下一輪處理，不會遺失。人工與自動同時處理同一筆：`FOR UPDATE` 加狀態檢查，只會建出一個 client。
- 切換 partner 的模式只影響之後送出的申請（`approval_mode` 是送出當下的快照）。
- master key（A22）：建 client 不處理 master key；見 §4.9 的 readiness 檢查。
- 通知（A24、§2.4）：收件人層排除 `USER_API`；Aegis 審核人的待審通知照舊（AUTO 申請不寄待審通知）。

### 4.8 open-api process 的組成

- imports：GlobalModule（精簡 configs／helpers、open-api 的 dataSource、pubSub）、EventEmitterModule、CacheModule、HttpModule（`maxRedirects: 0` 只給 webhook client）、`OpenApiPublicModule`、各 request service 的「只有 exports、沒有 controller」module（新增，補齊目前沒 export 的 service）、`CmtatTermsCoreModule`、CMTAT registry／reader／token repo module、上述 operation 的事件處理 module。
- route allowlist（只允許 `/healthz` 與 §3.5）；e2e 檢查沒有 cron。
- open-api 會發出 `CREATED`、`CLIENT_REVIEW_GROUP_STARTED`／`ADMIN_REVIEW_GROUP_STARTED`（domain 建單時發的），不會發其他 operation 事件；不要手動 emit。
- subscription（in-memory PubSub）不受影響：狀態變化的發布都在 main。

### 4.9 環境變數

| 變數 | process |
|---|---|
| DB、KV、RPC、S3（條款）、KYT、Mailgun、`HD_WALLET_MODE` 等 open-api 實際用到的 | open-api（不帶前台登入的 JWT、cookie、reCAPTCHA） |
| `OPEN_API_PORT`、`OPEN_API_TRUST_PROXY_HOPS`、`OPEN_API_RATE_LIMIT_PER_MIN`、`OPEN_API_REQUIRE_KYT`、`OPEN_API_ALLOW_HTTP_WEBHOOK`、`OPEN_API_RPC_CONCURRENCY`（整個 process 共用，預設 8） | open-api |
| `HD_WALLET_MODE`、`HD_WALLET_SUPPORTED_CURVES` | main、open-api（必須相同） |
| `OPEN_API_WEBHOOK_SECRET_ENC_KEY`（32 bytes base64，兩邊相同） | main、open-api |
| `OPEN_API_ALERT_EMAIL` | open-api（UNKNOWN 告警的收件信箱） |

- DB 連線池（目前 max 10）：open-api 不使用專用連線；e2e 做一次併發飽和測試（50 個同時請求不出現連線逾時）。
- **加密金鑰的部署順序**：`OPEN_API_WEBHOOK_SECRET_ENC_KEY` 要在**第一個包含 `OpenApiAdminModule` 的 main image** 部署之前，就放進各環境的 env（dev／ag-dev 是 `av-api-env`，正式環境是 compose 的 env 檔）。main 啟動時若沒有這個 key 或長度不對：**只停用 webhook secret 相關的 mutation 並記 error log，不讓 main 啟動失敗**（避免影響前台、後台）；open-api 則不 ready。這一步放在 Step 1。
- **HD readiness（A22）**：main 與 open-api 啟動時檢查：`HD_WALLET_MODE = SYSTEM`、`HD_WALLET_SUPPORTED_CURVES` 包含 `secp256k1`（沒設時程式預設是空陣列，建錢包會改成獨立金鑰而不是 HD，`configs.ts:387`）、DB 有 secp256k1、`clientId = null` 的系統 master key。任一不符 → open-api 不 ready；main 記 error log（main 不因此停止，避免影響既有前台）。建錢包時 key 缺失仍走 422 `MASTER_KEY_NOT_PROVISIONED`。正式環境已設 curves（Allen 11/22 確認）。
- **RPC 預算**：正式環境 Conflux 用 Confura 官方 API key（11/12 起，tx-listener 同一把）；open-api 的鏈上讀取與 listener 共用配額，上線前依 holders 與錢包詳情的預估量確認方案額度。

### 4.10 跨 client 對象（A6、A7）

```
isAllowedHolderClient(issuerClientId, holderClientId):
  holderClientId == issuerClientId → true
  issuer = link(issuerClientId)、holder = link(holderClientId)
  return issuer.state = ACTIVE && issuer.kind = ISSUER && holder.state = ACTIVE && holder.kind = INVESTOR
      && issuer.partner_id = holder.partner_id
```

| 入口 | action | 簽名錢包 | 託管錢包對象 | 白名單錢包對象 |
|---|---|---|---|---|
| API（ISSUER client） | SET_ALLOWLIST、BATCH_SET_ALLOWLIST、MINT、BATCH_MINT、BURN、BATCH_BURN、BURN_AND_MINT（from、to） | 發行方 maker | 發行方自己的；同 partner ACTIVE 投資人的 | 發行方自己的；同 partner ACTIVE 投資人的 **SELF** 白名單錢包 |
| API（ISSUER client） | GRANT_ROLE、REVOKE_ROLE | 發行方 admin | **只能是發行方自己的**（明確檢查） | 不接受 |
| API（ISSUER client） | SET_* | 發行方 admin | — | — |
| 後台（Aegis） | PAUSE、UNPAUSE、DEACTIVATE | custodian／admin | — | — |
| 後台（Aegis） | FREEZE／UNFREEZE（含 BATCH）、FREEZE／UNFREEZE_PARTIAL、FORCED_TRANSFER | custodian／admin | 同 API 的對象規則 | 同 API 的對象規則 |
| 前台（UI client） | 既有 | 既有 | 只有同 client（UI client 沒有連結，`isAllowedHolderClient` 自然只回同 client） | 只有同 client |

- domain（`_loadWallets`、`_loadWhitelistedWallets`）、估費 query、後台 `linkedHolders` 都用同一個 `AllowedHolderService`。投資人的白名單錢包另外要求 `owner = SELF`。
- `_isCustodyWalletAddress`（白名單地址不能是任何託管錢包）不變。
- 帳務：`getCmtatBalanceChanges` 以 walletId 記帳，投資人託管錢包直接入帳／扣帳；白名單錢包對象不動帳（既有）。完成入帳的冪等由 AEG-43 保證。
- **asset type 準備（REST 所有會載入錢包或 token 的路徑）**：寫成一個共用的 `prepareCmtatAssetTypes(walletIds, cmtatTokenIds)`：先查這些錢包持有的 `assetTypeIds`，加上請求指定的 `cmtatTokenId`，批次載入 ≥ 100000 的 `cmtat_tokens`，逐一 `CmtatAssetTypeRegistry.update()`，**之後**才用 `WalletQueryRepo` 載入（`WalletFactory` hydration 時就讀 `AssetTypeById`，提領的 resolver 也直接索引）。套用在：錢包列表、錢包詳情、提領、合約操作、部署完成後的第一個請求。REST 自己依「token 屬於路徑 client，或路徑 client 的錢包持有它」決定是否顯示，不改 `isAssetTypeVisible`。

### 4.11 生命週期

```
準備
  POST /v1/client-requests（kind=ISSUER）→ Aegis 核准 → webhook COMPLETED（clientId）
  POST /v1/client-requests（kind=INVESTOR）× N → 同上
  POST /v1/clients/{issuer}/wallets × 4（admin、maker、custodian、發行持有）
  POST /v1/clients/{investor}/wallets
  POST /v1/clients/{investor}/whitelisted-wallets（SELF，KYT）
  POST /v1/clients/{investor}/wallets/{w}/whitelisted-wallets
發行
  terms-documents → deployments → calls GRANT_ROLE × 6 → GET token 確認 roles
  calls BATCH_SET_ALLOWLIST（發行持有、投資人託管錢包、投資人 SELF 地址）
  calls BATCH_MINT（投資人託管錢包）
存續
  calls BURN_AND_MINT（分配、轉讓、換錢包）
  POST /v1/clients/{investor}/withdrawals（到自己的 SELF 地址；該地址已在合約 allowlist）
  calls SET_*
到期
  calls BATCH_BURN（投資人託管錢包、投資人 SELF 地址）
```

### 4.12 月結對帳單（只做 API_ONLY 相容的最小修改，A24、A25）

- 範圍：**只保證月結不會因為 API_ONLY client 而失敗或中斷**。CMTAT 在對帳單上的呈現不在本案（A25）；既有的「PDF／S3 寫入未 await」「每個 client 的明細只取 500 筆」不在本案修（A24）。
- `create-statement.ts`：
  - 取客戶清單改成 cursor 分頁迴圈（不再只取前 500 個 client；這是 API_ONLY 投資人數量造成的新風險）；排序用穩定的唯一鍵（`createdAt, id`）。
  - 每個 client 獨立隔離：該 client 的查詢（await）與產生報表的錯誤都在自己的 try/catch 內，只記 log，不中斷後面的 client（現在用 `reduce` 串接）。
  - 客戶名稱與聯絡資訊：**UI client 維持現行做法**（`formatClientInfo` 用 primary user 的姓名與信箱，欄位結果不變）；**只有 API_ONLY client** 改用 company 的 `companies.name`、`companies.email`，或 individual 的 `firstName lastName`、email 為 null；沒有值的欄位 PDF 留空，不得拋錯。
  - 寄送：有 `USER_PRIMARY` 才寄信；API_ONLY client 只產生並保存對帳單，不寄信。
- 驗收：UI client、API_ONLY 發行方、API_ONLY 投資人混在同一批、client 總數超過 500，排程跑完不中斷，API_ONLY client 都產生對帳單；**UI 的個人與公司客戶，PDF 欄位與寄信行為跟修改前完全相同**。另外跑一次其他三個排程（入金偵測、對帳、每日資產），確認不受 API_ONLY client 影響。

---

## 5. 邊界條件與錯誤處理

| # | 情境 | 處理 |
|---|---|---|
| E-1 | header 缺少、timestamp 非整數、時間偏移 > 5 分鐘、簽章錯、nonce 重複 | 401 |
| E-2 | 憑證過期／撤銷／partner 停用（`disableOpenApiPartner`） | 下一個請求即 401；既有 operation 照常、webhook 照送 |
| E-3 | 撤銷與請求同時 | 每次查 DB，撤銷 commit 後下一個請求即 401 |
| E-4 | 撤銷與核准同時 | 條件式 UPDATE，撤銷不被覆蓋 |
| E-5 | 已有 2 組 ACTIVE、兩筆核准同時 | 只成功一筆；已有 3 組時兩筆都失敗 |
| E-6 | 後台 admin 的 custody entity 不含 HK | 所有 partner 相關 query／mutation 拒絕 |
| E-7 | GENERATED 私鑰遺失 | 無法取回；建新憑證、撤銷舊的 |
| E-8 | 路徑 clientId 不屬本 partner／連結 DISABLED | 404；歷史查詢與 webhook 照常 |
| E-9 | 同 referenceId 併發、不同內容 | unique constraint；409 |
| E-10 | 建單後回應失敗 | 6c：operation 存在 → DONE |
| E-11 | KV 遠端已建任務、本地未寫、operation 未寫 | UNKNOWN＋告警；營運人員在後台看診斷後作廢請求；KV 的孤兒任務拿不到 Supervisor 核准、不會執行（§4.3.3） |
| E-12 | handler 晚於清理程序寫回、或晚於人工作廢 | version 條件更新保護請求列；建 operation 時 `insertGuard` 鎖同一列重查，已作廢就 rollback，不會出現第二張可執行的單（§4.3.3） |
| E-12a | 建錢包 commit 後 KV 呼叫失敗（KV 可能沒建、只建了錢包、或錢包和任務都建了） | DONE(202)；超過 10 分鐘沒有本地 `kv_tasks` → 列入 `KV_LINK_MISSING`；營運人員在審核頁拒絕 → DQ 收到 REJECTED，用新的 referenceId 重建（KV 殘留的錢包或任務不會被核准） |
| E-12d | `WALLET_UPDATE`、`WHITELISTED_WALLET_CREATE` 待審超過 10 分鐘 | `isKvOp = false`，不列入異常 |
| E-12e | 拒絕後服務重啟、listener 與人工處置順序互換 | 卡住的單只拒絕不作廢；每個對象只有一個最終狀態（部分唯一索引） |
| E-12f | open-api 的程式誤呼叫 KV 核准 API | `kv-client-guard` 丟例外（A27） |
| E-12b | 取角色佔用失敗 | 刪除自己的 PROCESSING 紀錄、409 retryable，可以用同一個 referenceId 重試 |
| E-12c | 同一個 body 送到不同路徑（不同 client 或 token） | hash 含路徑，視為不同內容 → 409 REFERENCE_ID_CONFLICT |
| E-13 | UNKNOWN 的 referenceId 重送 | 409 REQUEST_STATE_UNKNOWN（not retryable）；作廢後重送 → 409 DUPLICATE_REQUEST（附 FAILED／REQUEST_ABANDONED），DQ 改用新的 referenceId |
| E-14 | externalId 同時被兩筆申請佔用 | `open_api_external_ids` PK，只有一筆成功 |
| E-15 | 申請被拒後重送 | 用同一個 externalId、新的 referenceId |
| E-16 | 同一筆申請兩位 admin 同時核准 | `FOR UPDATE`＋條件式 UPDATE |
| E-17 | AUTO 自動核准失敗、申請後 process 重啟、人工與自動同時處理、兩個 worker | 退避重試（最多 5 次後轉人工）、失敗不擋後面的申請；在核准交易內 `FOR UPDATE SKIP LOCKED` 領取，只建一個 client |
| E-17a | HD 設定不符（不是 SYSTEM、curves 沒有 secp256k1、沒有系統 master key） | open-api 不 ready（§4.9）；建錢包時 key 缺失 422 MASTER_KEY_NOT_PROVISIONED |
| E-18 | 同時 GRANT 同一 token 的兩個角色、第一筆等 KV 超過 60 秒、兩個 process | 佔用持續到 operation 寫入（不會逾時被接手）；第二筆 409 retryable |
| E-18a | 同角色已有審核中、TIMEOUT（沒有結論）的指派，或已完成但角色還沒寫回 | `includeUnsettled` 視為進行中 → 422 CMTAT_ROLE_OPERATION_IN_PROGRESS |
| E-19 | 同時兩筆 BURN、BURN 與提領交錯 | §4.4 表格：不保證建立時擋下；後上鏈的 FAILED，本金不動、gas 照扣簽名錢包 |
| E-20 | GRANT_ROLE 對象是投資人錢包 | 422 CMTAT_INVALID_TARGET |
| E-21 | 對象是別的 partner、DISABLED、issuer → issuer、投資人的非 SELF 白名單 | 422 CMTAT_INVALID_TARGET |
| E-22 | 提領目的地不在 whitelistedIds | 422 |
| E-23 | 提領目的地不在合約 allowlist | 422 CMTAT_TRANSFER_NOT_ALLOWED（domain） |
| E-24 | 提領別的 partner 的代幣、同 partner 另一個 issuer 的代幣、已停用 investor 的提領 | 404（§3.5.9） |
| E-25 | 加入白名單時已有審核中的修改；第一筆核准釋鎖後第二筆才取鎖 | 409 retryable；第二筆在鎖內重讀錢包，兩筆新增都保留、policy 是最新的 |
| E-26 | 白名單錢包屬於別的 client | 422 WHITELISTED_WALLET_INVALID |
| E-27 | Conflux 沒有 KYT（正式） | 白名單建立、提領 422 KYT_NOT_AVAILABLE_FOR_NETWORK |
| E-27a | KYT 沒有分數、UNDETERMINED、分數剛好 0（既有解析問題） | STRICT 下 502 KYT_UNAVAILABLE，不建 operation |
| E-28 | KYT 服務故障 | 502 retryable |
| E-29 | 估 gas：模擬 revert vs RPC 故障 | 422 FEE_ESTIMATE_REJECTED vs 502 FEE_ESTIMATE_UNAVAILABLE |
| E-30 | CMTAT 上鏈完成但帳務、角色、SET_*、部署建檔尚未結算（AEG-43） | 對外 PROCESSING；`isOperationSettled` 為 true 才 COMPLETED；事件遺失時 reconciler 依查詢補 |
| E-30a | 已完成的 operation 被晚到的逾時改成 TIMEOUT（既有行為），包括完成還沒被投影就被覆蓋 | 候選狀態以 execution record／AEG-43 結算證據為準 → 仍推導為 COMPLETED（§4.5） |
| E-30b | TIMEOUT 之後晚到 COMPLETED／FAILED（含第 31 天以後、通知遺失） | 對外推進到最終狀態；reconciler 前 30 天每 5 分鐘、之後每天分頁核對 |
| E-31 | 剛部署或改名的 token | 載入錢包前 `update()` |
| E-32 | 一個投資人持有多個發行方的代幣 | 每個 token 都依 §4.10 顯示與操作 |
| E-33 | 有人在前台用 USER_API 角色建使用者、替 API_ONLY client 建使用者、把 API 帳號放進 policy | service 層擋下 |
| E-34 | 用 API 帳號登入 | 一般登入失敗 |
| E-35 | 月結遇到 API_ONLY client 或單一 client 出錯 | 不中斷整批（§4.12） |
| E-36 | webhook 舊快照晚到（落庫時） | rank 條件更新，不寫入 |
| E-36a | 舊事件因重試、人工重送而晚於新事件送達 | 允許（A23）；DQ 用 seq 條件更新 |
| E-37 | listener 漏寫、超過 30 天的待審單、零事件、後台單在 commit 後 emit 前中斷 | tracked target 在建單交易內登記並提交第一個狀態；reconciler 不限時間補 |
| E-38 | 派送中租約被接手 | lease_token 條件更新 |
| E-39 | 對方收到但未回寫 | 重送，DQ 去重 |
| E-40 | webhook URL 指向內網、DNS 檢查後改變、redirect | 擋下、綁 IP、redirects 0 |
| E-41 | open-api 先於 migration 啟動 | readiness 等待 |
| E-42 | open-api 誤載入帶 controller 的 module 或 JobModule | 啟動失敗／CI 失敗 |
| E-43 | 持有人很多、多頁同時查詢 | DB 分頁（每頁 ≤ 25）、每一次 RPC 呼叫都計入整個 process 共用的併發上限、逾時取消排隊中的呼叫 |
| E-48 | 部署新 main 時環境沒有 `OPEN_API_WEBHOOK_SECRET_ENC_KEY` | main 照常啟動、webhook secret 功能停用並記 error log；open-api 不 ready |
| E-48a | 正式環境 `OPEN_API_REQUIRE_KYT` 漏設、`false` 或非法字串 | open-api 不 ready（§4.6） |
| E-48b | 白名單取到 object lock 後驗證失敗 | `setRollback()`，鎖不殘留；立刻重送合法請求可以成功 |
| E-48c | 異常列表積很多筆 | 列表只回 DB 分頁；點開單筆才查 KV（逾時 5 秒） |
| E-49 | KYT provider 設定的大小寫 | 一律比對 `KytProvider` enum（值是 `BEOSIN`） |
| E-44 | `amount` 格式錯誤 | 400 |
| E-45 | 投資人錢包沒有 gas | 422 INSUFFICIENT_GAS_FEE（D4；dev 測試時事先補 CFX） |
| E-46 | 大量投資人 | client 申請可以設 AUTO（A20），但**每個錢包仍要 Aegis 核准**（Supervisor＋Auth Wallet）；寫進 DQ 文件 |
| E-47 | API 帳號出現在通知收件人 | 收件人層排除；同一封信的其他收件人照舊 |

---

## 6. 分階段執行清單

每步獨立驗證、獨立開 PR（merge 前逐一詢問 Allen）。**AEG-43 要在 Step 4 之前完成**，並提供 §3.7.1 用到的 `isOperationSettled(operationId)`（跨重啟有效，涵蓋沒有金額分錄的操作、角色同步、SET_* 的 token metadata、部署的 token 建檔）。Step 1–3 可以並行開發，但 Step 2 與 Step 3 共用 schema 與 outbox，**整合驗收要一起簽收**。AEG-41 獨立。

`tsc` 的驗收一律是「新增與修改的檔案沒有新的診斷」，不用總錯誤數比較（develop 本來就有既有錯誤，總數比較可能掩蓋新錯）。

### Step 1：骨架、partner／憑證、驗證

- [ ] `apps/open-api`（readiness：migration、HD 設定、KYT provider、加密金鑰；精簡 configs／helpers、route allowlist）、CI（image＋含 Postgres 的 e2e job）
- [ ] 各環境先放 `OPEN_API_WEBHOOK_SECRET_ENC_KEY`，再部署第一個包含新 module 的 main image
- [ ] migration 三支
- [ ] `USER_API`、新 Action、sign-in、建使用者 service 的角色與 API_ONLY 防護（GraphQL schema 不變＋相容測試）、policy 防護
- [ ] partner（HK、停用／啟用）、憑證（BOUND／GENERATED、核准、撤銷）的後台 GraphQL；custody entity 檢查；條件式轉移；mode、resolution 等封閉集合用 GraphQL enum
- [ ] guard、簽章、nonce、限流、scope、錯誤格式；`/healthz`、`/v1/me`

**驗收標準**

1. 兩個 image 同時冷啟：只有 main 跑 migration；open-api 在 migration 完成前不 ready，完成後正常；精簡 env 下 DI 正常；`HD_WALLET_SUPPORTED_CURVES` 拿掉 secp256k1、或 `KYT_PROVIDER` 不等於 `KytProvider.BEOSIN`（`OPEN_API_REQUIRE_KYT=true`）時 open-api 不 ready。
1a. 升級情境：舊 main＋新 schema 能正常服務；新 main 在沒有加密金鑰時仍能啟動（webhook secret 功能停用、記 error log）；新 main＋預先放好金鑰時一切正常。
1b. KYT 設定：`NODE_ENV=production` 時 `OPEN_API_REQUIRE_KYT` 漏設、`false`、非法字串各自不 ready；非正式環境漏設或非法字串不 ready、明確 `false` 可以 ready。
2. 跨語言驗簽向量全過；格式錯誤的各種變體都失敗。
3. e2e：401 的 9 種情況、403、429、撤銷後立即 401、`disableOpenApiPartner` 後下一個請求即 401。
4. 憑證：BOUND 與 GENERATED 都能核准使用；GENERATED 的私鑰只出現在 response，DB 與 log 都查不到；建立者不能核准；E-4、E-5 的併發情況符合預期。
5. custody entity 不含 HK 的 admin 看不到、改不了。
6. 前台、後台既有 GraphQL 操作文件對新 schema 驗證無破壞性變更；傳 `USER_API` 建使用者、替 API_ONLY client 建使用者、把 API 帳號放進 policy 都被 service 擋下；API 帳號無法登入。
7. 故意 import 帶 controller 的 module → 啟動失敗；open-api 沒有 cron。
8. `yarn test` 全過；新增與修改的檔案沒有新的 `tsc` 診斷。

### Step 2：client 建立、冪等、錢包與白名單、月結修正

- [ ] client 申請（API，單一 transaction）、核准／拒絕（後台）、AUTO worker（main）、partner 的 MANUAL／AUTO 設定、provisioning（company／individual、API 帳號、policy、連結，同一個 QueryRunner）、建 client service 拆兩段、mutate repo 允許 null 與 USER_API
- [ ] `ClientScopeGuard`、`GET /v1/clients…`
- [ ] `IdempotentWriteService`（hash 定義、保存 `request_body`、targetId 同時作 operation id、KV txId、walletId、`insertGuard`、6a／6b／6c、version、清理程序）、UNKNOWN 告警、後台「Open API 異常請求」（分頁列表、單筆診斷、連結、作廢）、`kv-client-guard`
- [ ] `open_api_locks`（CMTAT 角色佔用；Step 4 才會用到，表與 repo 先在這裡建）
- [ ] 錢包查詢／建立／加入白名單（`createAddWhitelisted`：先鎖再重讀）、白名單錢包（SELF＋§4.6 政策矩陣）、domain 的同 client 檢查、client repo 傳 accessMode 與系統 actor、通知收件人排除 USER_API
- [ ] 月結修正（§4.12）
- [ ] `GET /v1/requests…`、`GET /v1/client-requests/{referenceId}`

**驗收標準**

1. 發行方與投資人申請 → 核准 → client（沒有真人使用者、沒寄信）、API_ONLY、API 帳號、policy（無 approver group）、連結 ACTIVE；displayId 不含 externalId；被拒後同 externalId 可以重新申請；兩筆同 externalId 併發只成功一筆。
2. partner 設 AUTO 時，申請送出後不需人工即變 COMPLETED，建出的 client 與 MANUAL 核准的完全相同；申請 commit 後立刻重啟 main，worker 仍會處理；人工與 worker 同時處理只建一個 client；切回 MANUAL 只影響之後的申請。
3. 故障恢復（測試替身）：
   - 提領在 KV 遠端呼叫後丟例外 → UNKNOWN、告警發出、沒有第二筆；後台診斷顯示 KV `FOUND`；作廢後 GET 為 FAILED／REQUEST_ABANDONED，同 referenceId 重送 409 DUPLICATE_REQUEST，新 referenceId 可以建立；KV 的孤兒任務停在待審、無法被核准。
   - 作廢與晚到建單的兩種提交順序：作廢先 commit → 舊的建單 rollback、沒有 operation、沒有鎖資產；建單先 commit → 作廢被拒、改用連結。兩種情況最多只有一張可執行的 operation。
   - 建錢包在 commit 後 KV 失敗 → DONE；10 分鐘後列入 `KV_LINK_MISSING`；診斷能分辨 KV 沒建、只有錢包、錢包和任務都有；拒絕後 DQ 收到唯一的 REJECTED，新 referenceId 重建只產生一個新錢包；拒絕後立刻重啟服務，GET 與 webhook 仍只有一個最終狀態。
   - `WALLET_UPDATE`、`WHITELISTED_WALLET_CREATE` 待審超過 10 分鐘 → 不在異常列表。
   - open-api 的 KV client 呼叫核准 API → 被 `kv-client-guard` 擋下（單元測試）。
   - handler 晚於清理程序回寫 → 不覆蓋。
   - client 申請在 transaction 中途斷線 → 沒有殘留的 PROCESSING 紀錄；同 body 送到不同路徑 → 409 REFERENCE_ID_CONFLICT。
   - 以上的處理動作都只用後台完成，不需要直接存取 DB 或 KV。
4. 冪等的 409 各情境；暫時性錯誤可用同一 referenceId 重試成功。
5. 同一錢包同時兩筆加入白名單 → 一筆 409 retryable，且沒有殘留的 PROCESSING 紀錄；第一筆核准釋鎖後第二筆（持有舊快照的請求）才送出 → 結果保留兩筆新增的地址、policy 是最新的；取鎖後驗證失敗（不同鏈、不同 client）→ 沒有殘留的 object lock，立刻重送合法請求成功。
5a. client：公司發行方、個人投資人、機構投資人各測 MANUAL 與 AUTO。涵蓋 v0.6 省略 clientType 的兩種申請、v0.7 明傳的三種合法組合及 §3.5.4 的 400 案例。從 DB 驗證 clients.type、相應公司或個人資料及連結的 kind 與申請一致，commit 後從 DB 重新讀取 `access_mode = API_ONLY`、`createdBy`（AUTO 為 null）；既有 UI 建戶流程的 `access_mode = UI`、`createdBy` 不變。
5b. AUTO worker：兩個 worker 同時跑不重複處理；前 20 筆持續失敗時，第 21 筆仍能完成；同一筆連續失敗 5 次 → 轉人工、欄位合法、不會有第 6 次自動執行；失敗時另一個 worker 不會在退避寫入前重新領到。
6. dev：建錢包（斷言送給 KV 的 `keyMode = hd`、`masterKeyId` 是系統 master key）、SELF 白名單、加入白名單都 COMPLETED（`OPEN_API_REQUIRE_KYT=false`、DEFAULT 政策）。KYT 正式 gate 另外驗收（§4.6）。
7. 通知：API_ONLY client 的提領或建錢包事件，Aegis 審核人仍收到信，API 帳號不在收件人；UI client 的收件人不變。
8. API_ONLY client 的 operation 直接進 Aegis 審核。
9. 月結：§4.12 驗收。

### Step 3：Webhook

- [ ] tracked targets（含建單交易內登記、`createByAdmin` 的登記）、listener（兩邊）、reconciler（含 TIMEOUT 30 天）、dispatcher（lease token、交易外 HTTP、redirects 0、DNS 綁定）、client 申請事件
- [ ] webhook 後台 GraphQL（建立時停用、secret 一次）
- [ ] 補查與重送（WEBHOOK_MANAGE、稽核）

**驗收標準**

1. 單元：HMAC、secret 加解密、退避表、URL 檢查（含 DNS 改變的情況）。
2. e2e：建錢包流程的事件 rank 遞增、同 status 不重複、簽章正確；client 申請事件正確。
3. 注入「舊快照晚到」→ 不寫入；listener 漏寫最終事件 → reconciler 補上；建立超過 30 天的待審單、從未有事件的單 → reconciler 補上；後台單在 commit 後、emit 前終止 process → 重啟後 `/v1/operations/{id}` 查得到、最終事件只落一筆；途中停用連結仍保留歸屬。
3a. 狀態：202 之後立刻 GET 就有狀態；Supervisor 核准後進到 Auth Wallet，GET 的 `review.aegisStage` 跟著更新但不發新的 webhook；operation TIMEOUT 後晚到 COMPLETED → 對外推進；完成還沒被投影就被改成 TIMEOUT → 依證據推導仍是 COMPLETED；第 31 天才完成且通知遺失 → 每日核對補上；GET 與 webhook 一致。
3b. 舊事件因重試晚於新事件送達 → 允許，payload 的 seq 正確。
4. 兩個 dispatcher 同時跑、租約過期被接手 → 不重複標記、不覆蓋；接收端 500 → 退避後 FAILED；redeliver 後再送、callbackId 不變；對方收到但派送器在回寫前中斷 → 重送。
5. 停用、換 URL、rotate 的行為正確；secret 不出現在 log。

### Step 4：CMTAT、跨 client、提領（前置：AEG-43 已完成）

- [ ] 共用 request service（resolver 行為不變）
- [ ] 條款、部署、token 查詢、持有人（四種 type、分頁、RPC 併發）、合約操作 14 個 action
- [ ] `AllowedHolderService`（domain、估費 query、後台共用）、GRANT／REVOKE 明確檢查、SELF 限制
- [ ] REST 資產顯示（先 update registry 再載入）
- [ ] 提領（投資人提領發行方代幣；§3.5.9 的跨 client 判斷先於估費與 KV）＋STRICT KYT
- [ ] CMTAT 狀態接上 AEG-43 的 `isOperationSettled`（事件只用來提早喚醒）
- [ ] 角色佔用（§4.4）、`prepareCmtatAssetTypes` 套用到所有載入錢包／token 的路徑、fee 估算錯誤分類
- [ ] 後台：`linkedHolders` 與管控視窗、跨 client 對象顯示
- [ ] `domain-error-map.ts` exhaustive

**驗收標準**

1. 單元：§3.5.8 對照表每個 action；§4.10 矩陣每一列的正向與負向（GRANT 給投資人、別的 partner、DISABLED、issuer → issuer、投資人非 SELF 白名單）；`CMTAT_ADMIN_ACTIONS` 走 API 都 422；提領的負例（同 partner 另一個 issuer 的代幣、不同鏈、已停用 investor）與投資人自提正例。
1a. 同 partner、雙方 ACTIVE 且已加入 allowlist，發行方可 mint 給機構投資人。機構投資人持有 COMPANY clientId 時，對該 clientId 呼叫部署或 mint 時依 §3.5 回 404。
2. 既有 CMTAT 測試全過；前台對既有 UI client 做一次 MINT 行為不變。
3. dev 完整生命週期（Conflux testnet，全部用 API；dev 測試政策：`OPEN_API_REQUIRE_KYT=false`、投資人 B 的託管錢包事先補 CFX 供提領的 gas、Aegis 審核照 dev 慣例由 e2e 帳號核准）：發行方＋個人投資人 A＋機構投資人 B → 部署 → GRANT × 6（同時送出，第二筆起若撞鎖就重試）→ BATCH_SET_ALLOWLIST（發行持有、兩個投資人託管錢包、投資人 B 的 SELF 地址）→ BATCH_MINT 給兩個投資人 → BURN_AND_MINT（A → B）→ 投資人 B 提領到 SELF 地址 → BATCH_BURN（投資人 A 託管錢包＋投資人 B 的 SELF 地址）→ SET_INFORMATION；每步 webhook 正確（CMTAT 的 COMPLETED 在帳務套用之後）、holders 與鏈上一致、各錢包帳正確；紀錄寫進 `testing-records/`。
4. 後台從管控視窗選投資人託管錢包與投資人 SELF 地址 → 估費 → 凍結成功；DQ 收到 webhook（referenceId null）並可查 `/v1/operations/{id}`。
5. 剛部署、剛改 symbol 的 token，不重啟 open-api、且之前沒有任何 GET，第一個請求直接 POST call 或提領也能成功；一個投資人同時持有兩個發行方的代幣顯示正確。
5a. 角色：第一筆 GRANT 讓 KV 回應超過 60 秒（測試替身）、第二筆由另一個 process 送出 → 第二筆 409，第一筆完成後第二筆重試成功；第一筆 GRANT 已提交、仍在審核中（建單佔用已釋放）時，衝突的指派被擋、相容的指派可以排隊；同角色已有「沒有結論的 TIMEOUT」指派、或已完成但角色未寫回時，同角色不同錢包與 maker／custodian 衝突都被擋，相容的角色仍可同時送；反過來，已被拒絕或已結算、只是 state 被改成 TIMEOUT 的指派不再擋。
5c. holders：多頁同時查詢時，實測同時在途的 RPC 不超過上限；持續 502 時重試不會讓在途數累加。
5b. AEG-43：帳務已套用但通知遺失 → reconciler 依 `isOperationSettled` 補送 COMPLETED，GET 與 webhook 一致。
6. §4.4 表格的情境（含足額與不足額、已上鏈 revert、未廣播）結果符合文件，包括 gas 由哪個錢包支付。

### Step 5：後台 UI、部署、文件

- [ ] aegis-vault-admin「Open API」頁與相關顯示；aegis-vault-app 小改
- [ ] aegis-deploy（dev／ag-dev、ingress 16 MB、image-updater、rollout）
- [ ] architecture-docs；DQ 文件依實作校正
- [ ] stg 部署；正式環境計畫（正式部署另外取得同意；Conflux KYT 未解決前不上線）

**驗收標準**

1. 後台：建 partner → 綁定公鑰或產生金鑰（下載私鑰）→ 另一位 Supervisor 核准；停用／啟用 partner；核准 client 申請、看到 AUTO 失敗的申請；webhook secret 只顯示一次；「Open API 異常請求」頁的分頁列表、單筆診斷、連結、作廢可操作。
1b. 部署紀錄：記下正式環境持有 KV API key 的服務只有 av-api 的 main 與 open-api（A27）。
1a. 處理手冊：把 §4.3.3 的表格整理成營運用的處理手冊（`docs/cmtat-cold-wallet-api/runbook-anomalies.md`），由 Aegis 營運人員照著在 stg 演練一次。
2. dev host 只開 `/healthz` 與 `/v1/*`。
3. 用 DQ 文件的 Node 範例從零接上 dev，跑完 §4.11 的前半段（到 BATCH_MINT）。
4. 兩個前端 `tsc`、`next lint`、`next build` 通過；rollout 後新 image 正常服務。

### 人天估算（以不熟悉的工程師估，含環境建置）

| Step | 人天 |
|---|---|
| 環境建置 | 1.5 |
| 1 骨架、partner／憑證、驗證 | 6.5 |
| 2 client 建立（含 AUTO worker）、冪等、異常請求後台、錢包與白名單、KYT 政策、月結修正 | 10.5 |
| 3 對外狀態推導與 Webhook | 6 |
| 4 CMTAT、跨 client、提領、後台管控 | 10 |
| 5 後台 UI、部署、文件、處理手冊 | 6 |
| **合計** | **40.5** |

不含：AEG-43、AEG-41、Beosin 的 Conflux chain id 補進對照表、正式環境部署。

---

## 附錄 A：Codex 第一輪審查（2026-11-20）

v0.2 已處理，對照表見 v0.2 版本；第二輪審查的評估摘要：#2、#5、#7 已解；#1、#9 在 v0.2 未解，v0.3 由 B-H5、B-H6 處理；其餘部分解，見附錄 B。

## 附錄 B：Codex 第二輪審查（2026-11-22）的處理

| # | 問題摘要 | v0.3 的處理 |
|---|---|---|
| H1 | 發行方無可用建立入口（建公司必建 USER_PRIMARY） | A5：發行方也由 API 申請；建 client service 拆兩段；mutate repo 允許 null；「有沒有真人」改 join `accounts.role`（§3.1、§4.7） |
| H2 | 月結只查 USER_PRIMARY、串接中斷、只取 500 | §4.12；displayId 改系統產生、不含 externalId；通知只排除 API 帳號（收件人層） |
| H3 | 投資人外部自提與從外部地址贖回走不通 | A7（H3 選 A）；§4.10 矩陣；§3.5.9 允許提領同 partner 發行方的代幣；§4.11 補 allowlist 步驟 |
| H4 | 放寬 loader 錯放角色對象；估費 query 與後台凍結走不通 | §4.10 入口×action 矩陣、`AllowedHolderService` 共用、GRANT／REVOKE 明確檢查、後台 `linkedHolders` |
| H5 | 固定 targetId 不足以 RELEASE；id 沒傳到 INSERT；client 申請恢復 | §4.3：KV 先遠端後本地 → RELEASE 需 kv-api 確認；options 傳到 super 與 repo；client 申請同 transaction；version 條件更新；`retryable` 區分 502 |
| H6 | 鎖錯資源、只保護建立幾秒、連線池 | A12 選 (b)：拿掉專用連線的 advisory lock，只保留角色與錢包修改的表鎖；§4.4 寫清三種交錯結果；`insertLock` 同 QueryRunner；連線飽和測試 |
| H7 | COMPLETED 早於入帳／建檔，listener 不冪等 | AEG-43 另卡先修；A19＋§3.7.1：CMTAT 的 COMPLETED 等帳務套用 |
| H8 | 補送範圍不完整、舊快照、租約覆蓋 | `open_api_tracked_targets`＋rank 條件更新、reconciler 不限時間且涵蓋後台單、lease_token；A13 至少一次 |
| H9 | partner 管理權未對應 custody entity | A11：partner.custody_entity = HK，所有管理動作檢查；連結停用後的行為定義（§3.3.3） |
| H10 | KYT gate 只擋白名單 | §4.6 白名單與提領都套用、結果分類、正式驗收以真實供應商為準 |
| M1 | 改 GraphQL input 型別違反 §1.3 | schema 不變，在 service 驗證＋相容測試 |
| M2 | FeeEstimateError 一律 422 | 拆 `FEE_ESTIMATE_REJECTED`（422）與 `FEE_ESTIMATE_UNAVAILABLE`（502）；nullable 只在入口 |
| M3 | entity 共用、helpers 過多、readiness、e2e job、PER_CLIENT | 沿用既有 entity 陣列；精簡 helpers；readiness；Postgres e2e job；A22 照正式環境 SYSTEM 模式，不處理 per-client master key |
| M4 | 不需改全域可見性；hydration 時機 | 不改 `isAssetTypeVisible`；先 update registry 再載入錢包 |
| M5 | webhook 初設與 secret 矛盾、DNS 綁定、redirect、nonce 窗 | 建立時停用＋secret 一次；DNS 綁 IP；redirects 0；nonce 保留 15 分鐘 |
| M6 | externalId 雙表佔用、作廢規則、wallet update 檢查、409 的佔用 | `open_api_external_ids` 統一佔用；被拒釋放；來源錢包與每個 id 的檢查；409 不佔用 |
| Low | 憑證併發驗收拆分；GRANT 簽名是 admin；open-api 也發 REVIEW_GROUP_STARTED；SDL；schema 快照 | Step 1 驗收 4、§4.10、§4.8、§3.6、Step 1 驗收 6 |

## 附錄 C：Codex 第三輪審查（2026-11-22）的處理

| # | 問題摘要 | v0.4 的處理 |
|---|---|---|
| R1 | UNKNOWN 沒有可核對的 KV 識別；建錢包 commit 後 KV 例外 | targetId 同時作 KV txId（三條路都改用 `options.kvTxId`）；後台用 kv-api `GET /blockchain/task/:txId/status` 查，存 FOUND／NOT_FOUND／ERROR 與時間；RELEASE 由 server 檢查（NOT_FOUND＋超過 15 分鐘）；`kv_repair_needed`＋`retryOpenApiKvInit`（§4.3.3） |
| R2 | client 申請與冪等紀錄不同交易；409 未釋放；hash 定義 | §4.3.1 單一 transaction（externalId 衝突在同交易記 422）；§4.3.2 第 4 步刪除自己的紀錄；hash 含 method 與實際路徑 |
| R3 | KYT 會放過沒有評分 | domain 的 opt-in `kytPolicy: 'STRICT'`（只有 open-api 用，前台不變）；啟動檢查 provider；分數 0 的既有解析問題不修，STRICT 下得到 502（§4.6、A24） |
| R4 | SYSTEM 不足以保證用系統 master key | §4.9 HD readiness（mode、curves 含 secp256k1、系統 master key）；Step 2 斷言 keyMode 與 masterKeyId |
| R5 | AUTO 沒有可恢復的執行者；API 帳號欄位 | main 的 AutoApprovalWorker（輪詢＋`FOR UPDATE SKIP LOCKED`）、`approval_mode` 快照、失敗紀錄可見；帳號欄位、mobile null、createdBy、同 QueryRunner（§4.7） |
| R6 | 提領可能跨 issuer 引用 | §3.5.9 改用 `isAllowedHolderClient`，三方所有權與同鏈檢查先於估費與 KV |
| R7 | 後台單可能沒被追蹤 | `createByAdmin` 在建單交易內登記 tracked target；6c、LINK_EXISTING 同交易補登（§4.5） |
| R8 | 送達順序與 TIMEOUT | A23「落庫單調、送達可亂序」；TIMEOUT rank 4、非最終，COMPLETED／FAILED／REJECTED rank 5；對外狀態取 tracked 的最高 rank，GET 與 webhook 一致（§3.7） |
| R9 | 60 秒租約不足 | 角色佔用改成持續到 operation 寫入、UNKNOWN 期間保留、不設逾時；錢包修改改用既有 object lock（§4.4） |
| R10 | 月結的完成條件與 CMTAT 呈現 | 依 A24、A25：只做 API_ONLY 相容（client 分頁、每 client 隔離、空值規則）；寫檔未 await 與明細 500 不修；CMTAT 呈現由 DQ 處理（§4.12） |
| R11 | AEG-43 的完成條件 | 要求 AEG-43 提供跨重啟有效的 `isOperationSettled(operationId)`（含零分錄、角色、SET_*、部署建檔）；事件只用來喚醒（§3.7.1、§6） |
| R12 | A12 範例誤導 | §4.4 表格列出本金、解除鎖定、gas 各由誰承擔；DQ 文件同步 |
| R13 | fee 分類與 asset type 準備的入口 | provider adapter 保留錯誤分類；`prepareCmtatAssetTypes` 套用到詳情、提領、合約操作 |
| R14 | 通知收件人、partner 停用入口 | 收件人層排除 USER_API；新增 `disableOpenApiPartner`／`enableOpenApiPartner` |
| R15 | RPC 預算 | 每頁 25、整個 process 共用的 RPC 併發上限、單頁 10 秒預算；正式環境 Conflux 已用 Confura API key（Codex 引用的 11/3 公用節點紀錄已過時），上線前確認額度 |
| R16 | 型別與 tsc 驗收 | `TokenAmountInput`（> 0）與 `TokenAmount`（≥ 0）分開、uint256 依 decimals 換算後檢查；WalletState 對照；GraphQL enum；tsc 改為「修改檔案無新診斷」 |
| R17 | 決策與測試前提一致 | A20–A25 寫入；D4 註明待定與 dev 測試補 CFX；KYT fixture；Step 2／3 整合驗收一起簽收；E-46 區分 AUTO 與錢包審核 |

## 附錄 D：Codex 第四輪審查（2026-11-22）的處理

| # | 問題摘要 | v0.5 的處理 |
|---|---|---|
| H1 | 建錢包的補單用錯 KV API；FOUND 後清旗標仍失去關聯；「錢包有、任務沒有」 | A26：不做自動補單。walletId = targetId；後台診斷改用 `/wallet/:walletId/info`、`/wallet/:walletId/tasks`；`KV_LINK_MISSING` 由營運人員拒絕＋作廢、DQ 用新 referenceId 重建（§4.3.3） |
| H2 | 15 分鐘＋查無不能證明可以重送；FOUND 但沒有 operation 無解 | A26：referenceId 永不釋放；已查證 KV 孤兒任務拿不到 Supervisor 核准、不會執行，所以一律作廢即可；晚到的 operation 由審核頁警示並拒絕；保存 `request_body` 供人工判斷 |
| H3 | 對外狀態循環依賴、TIMEOUT 補查斷點 | §4.5 分成「候選狀態（依證據推導）」與「公開狀態（已提交）」；登記時就提交第一個狀態；TIMEOUT 滿 30 天後改每日分頁核對 |
| H4 | 角色檢查漏掉 TIMEOUT 與未結算的指派 | `findPendingRoleOperations` 的 opt-in `includeUnsettled`（只有 open-api 用）；相容角色仍可並行（§4.4） |
| H5 | KYT provider 大小寫；dev 成功路徑矛盾 | 用 `KytProvider` enum；§4.6 環境政策矩陣（dev 用 DEFAULT）；正式 gate 獨立驗收 |
| H6 | 白名單聯集在取鎖前計算；409 佔住 referenceId | `createAddWhitelisted`：同一個 QueryRunner 先取鎖（ON CONFLICT DO NOTHING RETURNING）、重讀、用現行 policy 算聯集；409 刪除本次紀錄 |
| H7 | client repo 沒有 accessMode、不接受系統 actor | mutate repo 明確傳 `accessMode` 與 `requesterId: string \| null`；UI 入口明確傳 UI；Step 2 驗收 5a |
| M1 | AUTO worker 重複領取、失敗卡住隊伍 | 在核准交易內 `FOR UPDATE SKIP LOCKED LIMIT 1`；`auto_next_attempt_at` 退避、5 次轉人工 |
| M2 | 月結改動到 UI 客戶 | UI 維持現行 `formatClientInfo`；只有 API_ONLY 用新規則；穩定排序鍵、每 client 隔離 await 錯誤 |
| M3 | RPC 併發要落在每次呼叫 | semaphore 包在每一次 RPC 呼叫；逾時取消排隊、已發出的等完成才釋放；Step 4 驗收 5c |
| M4 | 加密金鑰在 main 啟動就需要 | 金鑰在 Step 1 先部署；main 缺金鑰時只停用相關功能、不啟動失敗；升級情境驗收 |
| L1 | §4.4 表格的「必失敗／都成功」 | 改成依「合計是否超過可用量」分列；gas 依實際 receipt |
| L2 | DQ 文件 §1.3 與 §10.3 | DQ v0.5 同步修正 |

## 附錄 E：Codex 第五輪審查（2026-11-22～23）的處理

| # | 問題摘要 | v0.6 的處理 |
|---|---|---|
| H1 | 「只有 av-api 能送核准」缺部署證據；KV 核准 API 不檢查 operation | 前提拆成程式規則與部署條件；Allen 確認只有 av-api 持有 KV key（A27）；open-api 用 `kv-client-guard` 擋掉核准 API；Step 5 記錄部署條件（§4.3.3） |
| H2 | 作廢與晚到建單沒有 DB 層互斥 | domain 的 `insertGuard`：建 operation 的 transaction 內鎖同一筆請求列並重查；作廢也鎖同一列並重查 operation 不存在；先 commit 的贏，最多一張可執行（§4.3.3、§2.4） |
| H3 | 非 KV 單被誤列 `KV_LINK_MISSING` | 條件加 `target_type = OPERATION`、`isKvOp = true`；非 KV 單可直接連結；mutation 由 server 重查 |
| H4 | 「拒絕再作廢」產生兩個最終狀態 | 卡住的 operation **只拒絕、不作廢**；作廢只用在 operation 不存在；webhook 事件加「每個對象只有一筆 rank 5」的部分唯一索引；拿掉 `LATE_OPERATION_AFTER_ABANDON`（互斥後不會發生） |
| H5 | 正式環境 KYT 單一設定錯誤就放行 | 依 `NODE_ENV` 判斷：正式環境只接受明確的 `true`＋`BEOSIN`，其他不 ready；非正式環境也要明確設定 |
| M1 | 同一狀態內的審核進度不更新；CLIENT_REQUEST 映射 | 同 rank 非最終狀態時只更新 `last_view`、不發 webhook；補 CLIENT_REQUEST 的狀態與 payload 映射 |
| M2 | 角色佔用可能永久保留 | `includeUnsettled` 改用 `deriveCandidate`：沒有結論的 TIMEOUT 與未結算才佔用，已拒絕／失敗／已結算的不佔用 |
| M3 | 白名單取鎖後提早返回會 commit 鎖 | 取鎖後沒建 operation 的返回一律 `setRollback()`；驗收無殘留鎖 |
| M4 | AUTO 第 5 次失敗沒有合法時間、失敗紀錄另開交易 | 外層交易持鎖、provisioning 用 savepoint、同交易記失敗；第 5 次保留原時間、不再領取 |
| M5 | 異常列表沒分頁、每筆即時查 KV、`JSON` scalar | 列表回 DB 分頁摘要；單筆才查 KV（逾時 5 秒）；原始內容用 `requestBodyText: String`；CLIENT_REQUEST 不另存 |
| L1 | 「足額就一定成功」 | §4.4 與 DQ §7.6 改成「其他執行條件成立時可以成功」 |
| L2 | DQ webhook 範例只有註解 | DQ v0.6 範例改成實際 `await` 持久化函式，成功 commit 才回 2xx、失敗回 500 |

