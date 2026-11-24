# DigiQuick bond 升級 CMTAT（經 Aegis 冷錢包 Open API）：交付範圍與工作量估算報告

- 日期：2026-11-24
- 分析與估算：香港數字債券發行專家（bond-expert）、項目經理（pm），兩人討論兩輪後定稿
- 統籌與記錄：主會話
- 依據：
  - `dacc_chainfusion_docs/digiquick-integration-guide-v0.6.md`（下稱「指南」，v0.6 草稿。**指南所列的 API 全部仍在規劃中，尚未實作**）
  - `dacc_chainfusion_docs/Digital_Asset_Risk_Assessment_Report.md`（下稱「OZ 報告」）
  - 對四個 repo 的唯讀核對
- **依據缺口**：指南沒有寫 CMTAT 的細節，這些一律列為「Aegis 待確認」，包括：⑧ deployments 的 body、GRANT_ROLE ×6 是哪 6 個角色、CMTAT 3.0.0 的審計證據、decimals。（CMTAT 已定採用 3.0.0 版。）
- 本輪只做討論與估算，沒有修改任何 repo 檔案。

---

## 一、總覽

人日只計 DigiQuick 的投入，不含等待第三方的時間。區間的意思是：前面是「已有平台可改造」，括號內是「主要模組從零建置」。

| 版本 | DigiQuick 人日 | 建議配置（平均 FTE） | 預計日曆週數 | 決定工期的外部依賴 |
|---|---|---|---|---|
| **A 技術驗證版** | **54–84**（71–106） | 約 3.3：BE 2、FE 0.3、QA 0.4、PM 0.3、BA 0.2、DevOps 0.1 | **6–8 週**（mock）；Aegis dev 可用後再加 2–3 週真環境驗證 | Aegis dev 環境 |
| **B 首筆正式發行版**（A 之後的增量） | **71–112**（85–132）；另有 B4b 3–5，在 hypercare 期間做 | 開發期約 5：PM 0.5、BA 0.5、BE 2、FE 1、QA 0.5、DevOps 0.2、OPS 0.3、合規 0.2。UAT 與發行期約 3.5：PM 0.5、BE 1、QA 1、OPS 1 | 從 A 結束起算 **14–19 週** | 律師文件定稿、Aegis stg 與主網、Aegis 主網審核吞吐量 |
| **A＋B 累計（首發）** | **125–196**（156–238），另加 3–5 | — | 全案 **20–27 週**（前提：Aegis 環境準時；A 的真環境驗證和 B 的開發前段重疊） | — |
| **C 可重複交付版**（增量） | **61–101**（76–118） | 約 4.5：PM 0.5、BA 0.2、BE 2、FE 1、QA 0.5、DevOps 0.2、OPS 0.3 | **6–10 週**，B 的 hypercare 結束後開始 | 券商介面規格（若要介接） |
| C 選項（另議） | 投資人自助前台 15–25（25–35）；外部地址與提領 5–8；由 DigiQuick 計算利息 8–12（浮息再加 3–5） | — | 外部地址不排日期，要等 Aegis 主網 KYT | — |

三個重點：

1. **首發日期取決於第三方，DigiQuick 的開發不在關鍵路徑上。** 依序要看：
   - 債券文件、交收與登記冊安排何時定稿（律師、發行方）。
   - Aegis 的 dev、stg、主網何時交付，以及何時說明 CMTAT 3.0.0 的細節（⑧ body、6 個角色、審計證據、decimals）。
   - Aegis 主網的人工審核吞吐量：每檔新 CMTAT 至少有 8–9 筆必須依序審核，6 個角色指派一次只能處理一筆（指南 §7.3⑨、§7.6）。
2. **盡量少改現有功能的做法：** bond 走一條平行的新發行路徑（`issuanceChannel = AEGIS_CMTAT`）。既有的 TokenFactory、listener 鏈上掃描、其他 8 種資產和已發行的 DACC29 都不動。共用程式碼只改兩處：`Asset.status` 的推導多一個分支，以及新增 GraphQL 欄位。
3. **每新增一筆發行**：B 模式（人工）約 5–10 人日營運工作；新發行方加上 20 位以上新投資人時約 8–12 人日。C 模式約 2–4 人日。Aegis 審核的日曆等待：首發 5–10 個工作天，之後每檔 3–7 個工作天。

---

## 二、依據核對：既有功能、待開發、Aegis 待交付

### 2.1 既有可重用或可改造（唯讀核對，路徑相對於各 repo）

| 功能 | 路徑 | 判定 |
|---|---|---|
| bond 資料欄位（35 個，含發行人、規模、票息、付息日、day count、到期日、受託人、登記處等） | `digiquick-api/database/entities/bond-asset.ts`、`core/asset/{dto,models,repositories,services,resolvers}/*bond*` | 小改：缺 ISIN、記錄日、decimals、條款文件 ID；`paymentDates`、`dayCount` 是自由文字，不能直接用來計息 |
| 發行方提交 bond | `digiquick-app/components/Bond/BondAddBtn.tsx`、`utils/hooks/mutations/useCreateBondAsset.ts` | 直接重用；首發不改 app |
| 資產審核、確認、簽核 | `core/asset/services/review-asset-service.ts`、`confirm-asset-service.ts`、`sign-asset-service.ts`；admin `components/Asset/AssetReviewBtn.tsx` | 直接重用 |
| 公司 KYC、DD 文件 | `core/company/services/kyc-service.ts`、`dd-document-service.ts`、admin `components/PendingKyc/*` | 小改：`CompanyDb` 缺 `incorporationNo`，而 Aegis ISSUER 申請需要（指南 §7.2①） |
| 權限框架 | `database/entities/permission*.ts`、`common/graphql/auth.ts`（`@WriteAuth(AdminAction.X)`） | 直接重用並擴充；目前**沒有** maker-checker |
| email 事件與模板 | `core/email/services/*`、`core/email/templates/*`、`common/misc/mailer.ts` | 直接重用 |
| 檔案上傳（S3 等） | `core/file-upload/controllers/file-upload.controller.ts` | 直接重用；可作為條款 PDF 轉送 Aegis 的來源。main app 已有 REST controller，webhook 可以加在 main |
| cron 骨架、DB transaction 工具 | `apps/listener/src/register-cron-jobs.ts`、`common/misc/transaction.ts` | 直接重用（用於輪詢對帳、webhook 去重） |
| 網路設定 | `common/providers/network.ts` | 小改：缺 Conflux eSpace 主網 |
| CI 與 Docker | `.github/workflows/push-*`、`main.Dockerfile`、`listener.Dockerfile` | 直接重用；沒有 lint、test、migration 步驟 |

### 2.2 不適用，維持原樣不改

- TokenFactory 鑄造路徑：`digiquick-admin/components/Asset/AssetMintBtn.tsx` 在瀏覽器用 MetaMask 簽。bond 走 CMTAT 後只要在 UI 隱藏這顆按鈕。
- 綁定單一 TokenFactory 地址的 listener 掃鏈與事件表：`core/contract-activity-task/tasks/transaction.ts`、`token-transfer-event.ts`。
- `asset-transaction.ts` 的「逾時就作廢重送」語意。這和指南 §5「TIMEOUT 不是最終狀態、不可換 referenceId 重送」衝突，所以 CMTAT 路徑要另建請求表。

### 2.3 待開發（DigiQuick 完全沒有）

- **投資人實體**：現有 `Customer`（`database/entities/customer.ts`）是發行方員工帳號，不是投資人。
- **付款、交收、登記冊、付息模組**：admin 的 `PaymentPending`、`Withdrawal`、`Overdue` 等頁面只有表頭，沒有資料來源。
- **Aegis 整合**：Aegis client、請求狀態表、webhook 接收、輪詢對帳、發行編排、持有人快照。
- **自動化測試**：api 唯一的 `apps/main/test/app.e2e-spec.ts` 引用不存在的 `./../src/main.module`，跑不起來。

### 2.4 Aegis 待交付能力（指南已規劃、尚未實作）

| 能力 | 指南章節 | 狀態與待確認 |
|---|---|---|
| 客戶申請（發行方、投資人） | §1.1、§7.2①② | 已規劃。待確認：Aegis 對投資人的審核標準 |
| 錢包建立（admin、maker、custodian、發行持有；投資人託管錢包） | §7.1、§7.2③④ | 已規劃。待確認：CFX gas 由誰出、誰補充 |
| 條款上傳、CMTAT 部署、角色、allowlist、mint、burn、burn and mint、修改合約資訊 | §7.3、§7.4、§7.5、§8 | 已規劃。待確認：⑧ body、6 個角色、CMTAT 3.0.0 的審計證據（指南沒有寫）。OZ 報告的 CertiK 審計只覆蓋 V1 TokenFactory |
| 外部地址登記與提領 | §7.2⑤⑥、§7.4 | 已規劃，但主網 Conflux KYT 接上前會回 `422 KYT_NOT_AVAILABLE_FOR_NETWORK`（§7.2、§13） |
| webhook、補查與重送、各種查詢 | §9、§10 | 已規劃 |
| 暫停、凍結、強制轉移、停用 | §1.3、§7.5 | 只能由 Aegis 後台操作，DigiQuick 只會收到 webhook |
| dev、stg、主網 base URL | §13 | dev、stg：「實作完成後提供」；主網：「上線前提供」，要先接上 KYT |
| **付息、DvP、現金帳** | —（指南沒有涵蓋） | **不在 Aegis 範圍**，CMTAT 也不處理現金，付款一律在鏈外 |
| 審核 SLA | —（指南沒有涵蓋） | Aegis 待確認；指南 §1.3 只寫「一筆可能要好幾個小時」 |

---

## 三、A 技術驗證版

目標：用 mock 和 Aegis dev（testnet）證明以下各項能正確落地：客戶與錢包建立、條款上傳、CMTAT 部署、6 個角色、allowlist、mint、burn，以及非同步結果（202、webhook、輪詢）和三方對帳。

| # | 工作包 | 具體交付物 | 負責方 | 角色 | 改造 PD | 從零 PD | 前置依賴 | 驗收條件 |
|---|---|---|---|---|---|---|---|---|
| A1 | 項目管理、釐清 Aegis 介面 | 範圍說明；問題清單（⑧ body、6 個角色、CMTAT 3.0.0 審計證據、decimals、審核 SLA、CFX、ISIN 能否後補、投資人審核標準）；週會 | DigiQuick（Aegis 回覆） | PM | 4–6 | 5–7 | — | 每個問題都有回覆，或已登記為風險 |
| A2 | 產品與流程設計 | 發行狀態機；bond 欄位到 CMTAT 參數的映射；referenceId 規則；錯誤決策表（retryable、TIMEOUT、UNKNOWN、TECHNICAL_RETRY、ABANDONED；一般 REJECTED 不自動重送） | DigiQuick | BA、BE | 4–6 | 5–8 | A1 | 決策表涵蓋指南 §5、§6、§11 的所有錯誤碼 |
| A3 | Aegis API client | RSA-PSS 簽章（path 含 query string）、nonce 與 timestamp、錯誤分類、429 依 `Retry-After` 等待、120 次／分鐘限流、重試換 nonce 但不換 referenceId | DigiQuick | BE | 4–6 | 4–6 | A2、A12 | mock 和 Aegis dev 驗簽都通過；簽章有單元測試 |
| A4 | 請求落地與 webhook | `AegisRequest` 表、`AegisCallback` 去重表、migration；`POST /aegis-vault/webhook`（raw body、HMAC、同一 transaction 內去重並比較 seq、10 秒內回應） | DigiQuick | BE | 5–8 | 6–9 | A2 | 重複、亂序、跳號都測過；狀態只會往前推進 |
| A5 | 輪詢對帳 cron | listener 新 cron：查詢未到終態的請求；補查 FAILED 的通知；UNKNOWN 和 TIMEOUT 只查詢、不重送；發出告警 | DigiQuick | BE | 4–6 | 5–7 | A3、A4 | 關掉 webhook 後，請求仍能靠輪詢到達終態 |
| A6 | 發行編排（最小） | ISSUER 與 INVESTOR 申請、4＋N 個錢包、條款、部署、GRANT_ROLE ×6 依序送出、⑩ 確認角色、BATCH_SET_ALLOWLIST、BATCH_MINT、BATCH_BURN，每批最多 20 | DigiQuick | BE | 8–12 | 8–12 | A3–A5 | testnet 走完 ①–⑫ 和 BURN；ConfluxScan 上查得到每筆交易 |
| A7 | 資料模型（最小） | `AegisClient`、`AegisWallet`、`Investor`、`CmtatToken`；bond 加 ISIN、decimals、termsDocumentId、issuanceChannel；`Asset.status` 加 CMTAT 分支；company 加 incorporationNo | DigiQuick | BE | 4–6 | 7–10 | A2 | migration 可執行也可回滾；其他資產的查詢不受影響 |
| A8 | Aegis mock server | 可模擬 202、至少送一次、亂序、重複、TIMEOUT 後完成、UNKNOWN、429、超過 20 筆的批次、409 角色衝突 | DigiQuick | BE、QA | 5–8 | 5–8 | A2 | 每一種異常都能用程式注入 |
| A9 | admin 前端（最小） | bond 詳情加「CMTAT 發行」步驟面板，顯示 Aegis 狀態、referenceId、txHash；bond 隱藏 `AssetMintBtn` | DigiQuick | FE | 4–6 | 10–15 | A6、dev 部署、codegen | 營運能在 UI 上走完 testnet 流程 |
| A10 | 資料與對帳驗證 | 依序翻頁拉 holders（每頁 25），和 DB 的分配比對，產出腳本或報表 | DigiQuick | BE | 3–5 | 3–5 | A6 | 鏈上、Aegis、DB 三方數字一致 |
| A11 | 測試 | 修好 jest；client、webhook、狀態機的單元測試；mock 整合測試；Aegis dev 端到端腳本 | DigiQuick | QA、BE | 6–9 | 8–11 | A3–A8 | 關鍵路徑全部通過；異常案例全部覆蓋 |
| A12 | 部署 | dev 的 env 和 Secrets；自行產生 RSA 金鑰（指南 §2 方案 A）；固定出口 IP；webhook HTTPS URL | DigiQuick（Aegis 核發憑證） | DevOps | 2–4 | 4–6 | Aegis dev | `/v1/me` 呼叫成功；webhook 可以到達 |
| A13 | 營運準備（最小） | 操作說明；Aegis 聯絡窗口與升級路徑 | DigiQuick | OPS、PM | 1–2 | 1–2 | — | 文件交付 |
| | **A 合計** | | | | **54–84** | **71–106** | | |
| A-X1 | Aegis 工作 | 實作 API、dev 環境、testnet CFX、憑證（要兩位主管核准，§2） | Aegis | — | 第三方 | | — | `/v1/me` 可用 |
| A-X2 | Aegis 工作 | 審核 testnet 請求：新發行方約 13 筆，加 2N＋2⌈N/20⌉ 筆和 BURN | Aegis | — | 第三方 | | — | — |

- **關鍵路徑**：A1 取得回覆 → A2 → A3、A4 → A6 → A12（Aegis dev 可用）→ A11 端到端。
- **日曆**：用 mock 6–8 週；Aegis dev 可用後再加 2–3 週。人力不是瓶頸，日曆長是因為工作要依序做，而且要等 Aegis。DigiQuick 不必等 Aegis，就能先完成 A 約 70% 的工作。

---

## 四、B 首筆正式發行版

### 4.1 首發採用的假設（兩位隊員一致）

- 投資人：20 人以內全人工錄入；20–50 人用 CSV 匯入；超過 50 人要先完成 C5。
- 只用 Aegis 託管錢包，不開放外部地址。
- 交收：人工 FoP，先款後券。條款寫明「付款確認後 T+1 至 T+3 交付代幣」。
- 持有人紀錄：鏈下登記冊為權威，鏈上為鏡像。
- 付息與贖回：由付款代理全權負責計算與付款。
- 投資人開戶比交收日提前 2–3 週開始。

### 4.2 逐步流程：誰執行、如何覆核、如何確認完成、如何處理例外

| 步驟 | 執行 | 覆核 | 完成確認 | 例外處理 |
|---|---|---|---|---|
| **1. 發行設定**（文件、條款、ISIN） | 律師與安排行定稿文件，申請 ISIN。DigiQuick OPS 在 admin 錄入 ISIN 和 decimals，上傳已簽署的條款 PDF（指南 §7.3⑦） | 營運主管比對 PDF 和欄位（四眼），取得發行方書面確認，並記錄 PDF 雜湊值 | 律師出具定稿確認；Aegis 回傳 `termsDocumentId` | mint 前要修改：重新上傳並重新覆核。mint 後要修改：用 SET_TERMS（B4b） |
| **2. 發行方開戶** | OPS 在 admin 觸發 ISSUER 申請和 4 個錢包（§7.2①③） | 營運主管核對 clientId、4 個 walletId 和地址，截圖存檔 | client-request 和 wallet 都 COMPLETED | 被 REJECTED：查明原因，用新的 referenceId 重新申請 |
| **3. 投資人准入**（KYC、AML、專業投資者認定） | 券商做 KYC、AML 和專業投資者認定，出具簽署版名單。DigiQuick 只收最少個資（姓名、生日、國籍），由 OPS 錄入或匯入，觸發 INVESTOR 申請和託管錢包（§7.2②④） | 營運主管核對名單和券商簽署版一致 | 每位投資人都有 clientId 和 walletId | Aegis 拒絕：通知券商，取消該筆分配，**不重送**。`CLIENT_EXTERNAL_ID_EXISTS`：查重 |
| **4. 認購與分配** | 安排行簿記和定價，出具最終分配表。OPS 匯入，系統檢查合計等於發行量、每筆是最低面額的整數倍 | 營運主管四眼比對合計和逐筆數字 | admin 顯示分配合計和發行量一致 | mint 前可以修改；mint 後只能用 BURN_AND_MINT |
| **5. 付款與交收確認** | 投資人經券商或付款代理以 CHATS 付款。付款代理出具到帳清單，OPS 標記「已付款」 | 付款旗標由另一人覆核 | 全額到帳的投資人才進入 mint 批次 | 未到帳：不 mint，依券商指示延後或取消。部分到帳：依條款處理，通常不接受 |
| **6. 代幣交付** | OPS 依序觸發部署、6 個角色、⑩ 角色確認、allowlist（可在交收前先做），付款確認後送 BATCH_MINT 到投資人託管錢包。BE 值班核對 txHash | 營運主管對照 run-of-show 逐項打勾 | holders、分配表、登記冊三方簽核一致 | FAILED：查原因（gas、allowlist），修正後用新的 referenceId 重送。TIMEOUT 或 UNKNOWN：**不重送**，等候並聯絡 Aegis，必要時延遲交收並通知投資人。一般 REJECTED：不自動重送 |
| **7. 託管與持有人對帳** | 代幣由 Aegis 託管。OPS 每日或每次異動後拉 holders 快照（§9），用 Excel 和 Aegis 託管帳、登記處的登記冊三方比對 | 營運主管簽核，存檔作為 OZ R04.2 的證據 | 連續 5 天沒有差異，或差異都有解釋 | 有差異：暫停所有代幣異動，通知 Aegis 和登記處，查明後才恢復 |
| **8. 事件處理**（暫停、凍結、強制轉移、更正） | 發行方或受託人書面指示 → DigiQuick 合規覆核 → 書面請 Aegis 在後台操作 → webhook 落地並告警（B4a） | 合規覆核 | `/v1/operations` 查詢結果和鏈上一致，登記冊已更新 | 沒有指示卻出現事件：立即向 Aegis 查證 |
| **9. 付息** | 付款代理在記錄日依登記冊計算並付款。DigiQuick OPS 提供記錄日的 holders 快照，雙方獨立計算後比對 | 營運主管和付款代理雙方核對 | 付款代理出具付款完成報告 | 投資人銀行帳戶失效：付款代理依文件處理（暫留款項）。首次付息在發行後 6–12 個月，**不在首發關鍵路徑上**，但記錄日快照要在首次付息前 1 個月就緒 |
| **10. 到期或提前贖回** | 付款代理確認本金已付 → OPS 觸發 BATCH_BURN（§7.5）→ 請 Aegis 暫停或停用合約 | 營運主管覆核銷毀清單 | holders 歸零，登記冊註銷 | 付款失敗就不 burn（先付款、後銷毀）。BURN 必須在 A 版驗證過 |
| CFX gas | Aegis 或 DigiQuick（待確認） | B10 的餘額告警 | 餘額高於門檻 | `INSUFFICIENT_GAS_FEE`：補充 CFX 後用新的 referenceId 重送 |

### 4.3 工作量表

| # | 工作包 | 具體交付物 | 負責方 | 角色 | 改造 PD | 從零 PD | 前置依賴 | 驗收條件 |
|---|---|---|---|---|---|---|---|---|
| B1 | 項目管理（多方協調） | 整合時程、RACI、兩段式 run-of-show（先開戶，再發行）、go/no-go 清單、週報 | DigiQuick | PM | 10–15 | 10–15 | A 完成 | 各方簽核 go/no-go |
| B2 | 產品與流程設計 | 全生命週期 SOP；四眼覆核點；對齊 OZ R07.1 和 R04.1 | DigiQuick（發行方、券商、付款代理、律師確認） | BA、合規 | 8–12 | 8–12 | 第六章的決策 | 各方確認 SOP |
| B3 | 投資人與分配（後端） | 投資人錄入和 CSV 匯入；Aegis 申請和錢包；分配表校驗；付款旗標；mint 前檢查 | DigiQuick | BE | 8–12 | 10–14 | A7 | 分配表不符時無法送出 mint |
| B4a | Aegis 後台事件落地 | 暫停、凍結、強制轉移、停用的 webhook 落地；`/v1/operations` 查詢；告警 email | DigiQuick | BE | 3–5 | 3–5 | A4 | 每一種事件都有紀錄和告警 |
| B5 | 資料與對帳 | holders 快照表和匯出；人工三方對帳範本 | DigiQuick | BE、OPS | 5–8 | 5–8 | A10 | 連續 5 天對帳沒有差異，或差異都有解釋 |
| B6 | admin 前端 | 投資人列表、分配匯入與覆核、例外佇列、快照匯出、新權限項 | DigiQuick | FE | 8–12 | 12–18 | B3、B5 | 營運不必直接動資料庫就能完成首發 |
| B7 | app 前端 | 首發不改，條款資料由 OPS 在 admin 錄入（改到 C13 做） | DigiQuick | FE | 0–1 | 0–1 | — | — |
| B8 | 安全與稽核 | 私鑰移到 Secrets Manager 或 KMS；稽核日誌；新的 AdminAction；投資人個資加密與存取控制（OZ R06.1、PDPO） | DigiQuick | BE、DevOps | 5–8 | 9–14 | A12 | 稽核日誌可匯出；env 沒有明文私鑰 |
| B9 | 測試與 UAT | Aegis stg 端到端；人工回歸 8 種既有資產；UAT；**主網彩排**（測試用 CMTAT 走一遍部署、角色、allowlist、mint 1 單位、burn，並量出審核時間） | DigiQuick（發行方、券商、付款代理、Aegis 參與） | QA、OPS | 10–15 | 12–18 | Aegis stg、主網憑證 | UAT 簽核；回歸沒有缺陷；有實測的審核時間 |
| B10 | 部署 | 加 Conflux 主網設定；正式環境 env；執行 migration；監控告警（webhook 失敗、請求久未到終態、409 卡住、CFX 餘額）；回滾計畫 | DigiQuick | DevOps | 4–6 | 6–9 | Aegis 主網憑證 | 正式環境 `/v1/me` 成功；告警演練有觸發 |
| B11 | 營運準備 | runbook（TIMEOUT、UNKNOWN、REJECTED、FAILED、CFX、事件、DLT 持續營運）；訓練；hypercare 排班 | DigiQuick | OPS、PM、合規 | 6–10 | 6–10 | B2 | 做過一次桌面演練並留紀錄 |
| B12 | 首發執行 | 第一段：開戶。第二段：條款 → 部署 → 角色 → allowlist → 付款確認後 mint → 發行後對帳 | DigiQuick（Aegis 審核） | OPS、BE | 4–8 | 4–8 | B1–B11 | 三方簽核一致 |
| | **B 首發前合計** | | | | **71–112** | **85–132** | | |
| B4b | 存續操作（延到 hypercare） | BURN_AND_MINT（轉讓、換錢包）、SET_TERMS、SET_INFORMATION | DigiQuick | BE | 3–5 | 3–5 | A6 | 第一筆轉讓或換錢包之前完成 |
| B-X1 | 第三方工作 | stg、主網、憑證、MASTER_KEY、每筆人工審核、CFX 安排、CMTAT 審計證據 | Aegis | — | 第三方 | | | |
| B-X2 | 第三方工作 | 文件定稿：代幣與登記冊的關係、只用託管錢包、強制轉移的依據、T+1 至 T+3 交付；條款 PDF；ISIN | 發行方、律師 | — | 第三方 | | | |
| B-X3 | 第三方工作 | 投資人 KYC 與專業投資者認定、簽署版名單與分配表 | 券商 | — | 第三方 | | | |
| B-X4 | 第三方工作 | 到帳清單、記錄日付息、贖回付款 | 付款或交收機構（付款代理） | — | 第三方 | | | |
| B-X5 | 第三方工作 | 滲透測試（OZ R02.4）、登記冊 | 其他（測試機構、登記處） | — | 第三方 | | | |

- **日曆（從 A 結束起算 14–19 週）**：

  | 階段 | 週數 |
  |---|---|
  | 開發 | 6–8 |
  | UAT | 3–4 |
  | 主網彩排 | 1–2 |
  | 發行窗口 | 2–3 |
  | hypercare | 2 |

  投資人開戶的 2–3 週和 UAT 後段並行。B2、B11 的非技術工作受律師和券商的回覆速度影響，要預留 4–6 週和開發並行。
- **關鍵路徑**：律師與發行方決策 → B2 → B3、B6 → Aegis stg → B9 UAT → Aegis 主網憑證 → 主網彩排 → 投資人開戶 → B12。

---

## 五、C 可重複交付版（增量）

觸發時點：
- 第二筆發行前要完成 C2、C3、C4。
- 投資人超過 50 人之前要完成 C5。
- 首次付息的記錄日前 1 個月，要完成 C7 的快照部分。

| # | 工作包 | 具體交付物 | 負責方 | 角色 | 改造 PD | 從零 PD | 前置依賴 | 驗收條件 |
|---|---|---|---|---|---|---|---|---|
| C1 | 產品設計 | 發行模板、覆核矩陣、報表規格 | DigiQuick | BA | 3–5 | 3–5 | B 上線 | 規格簽核 |
| C2 | 發行模板 | 參數模板、條款版本管理、自動產生 referenceId、一鍵產生發行計畫 | DigiQuick | BE、FE | 6–10 | 6–10 | C1 | 第二筆發行的設定在半天內完成 |
| C3 | 權限與覆核（maker-checker） | 每個 Aegis 寫入都要求發起人和核准人不同；稽核軌跡 | DigiQuick | BE、FE | 6–10 | 10–14 | B8 | 同一人無法發起又核准 |
| C4 | 自動化編排 | 收到 COMPLETED 後自動推進；角色依序送出；自動切批；retryable 自動重試；工作佇列 | DigiQuick | BE | 8–12 | 8–12 | B | testnet 發行全程不需人工介入 |
| C5 | 批次處理 | 券商 CSV 或 API 介接；批量操作可以斷點續送；處理部分失敗 | DigiQuick（券商提供規格） | BE、FE | 5–8 | 5–8 | C4 | 200 位投資人的批次測試通過 |
| C6 | 營運工具 | 例外主控台、webhook 補送 UI、請求搜尋、CFX 監控、Aegis 事件日誌 | DigiQuick | FE、BE | 6–10 | 8–12 | B6 | 處理例外不需要找工程師 |
| C7 | 發行後報表 | 自動快照與差異報告、記錄日快照、贖回報表、對帳簽核報告 | DigiQuick | BE、FE | 6–10 | 8–12 | B5 | 營運可以自行產出報表 |
| C8 | 測試自動化 | mock 整合測試和 lint、test 納入 CI；回歸套件 | DigiQuick | QA、DevOps | 5–8 | 6–9 | A11 | CI 失敗會擋下部署 |
| C9 | 部署 | CI 加 migration 步驟；多環境設定 | DigiQuick | DevOps | 2–4 | 2–4 | — | 自動部署成功 |
| C10 | UAT | 第二筆發行的彩排 | DigiQuick | QA、OPS | 4–6 | 4–6 | C2–C7 | 簽核 |
| C11 | 營運準備 | 更新 runbook、訓練 | DigiQuick | OPS | 2–4 | 2–4 | — | 訓練完成 |
| C12 | 項目管理 | — | DigiQuick | PM | 6–10 | 6–10 | — | — |
| C13 | app 發行方欄位與進度頁 | 發行方自行提交 ISIN、decimals、條款 PDF，並可查看進度 | DigiQuick | FE | 2–4 | 8–12 | C1 | 發行方能自行提交 |
| | **C 合計** | | | | **61–101** | **76–118** | | |

C 版日曆 6–10 週，關鍵路徑為 C1 → C2、C3 → C4 → C5 → C10。

---

## 六、需要確認的項目：對範圍與工期的影響

| 項目 | 選項（**粗體**為首發採用） | 範圍與人日影響 | 工期影響 |
|---|---|---|---|
| **付款與交收模式** | **① 人工 FoP、先款後券** | 加付款旗標和 mint 前檢查，已含在 B3（+1–2） | 發行窗口 2–3 週 |
| | ② 先 mint 到發行持有錢包，到帳後用 BURN_AND_MINT 分配 | B4b 要提前到首發前（+3–5） | 每位投資人多 1 筆 Aegis 審核，窗口多 1–2 週 |
| | ③ 經 CMU 結算，代幣作為鏡像 | 法律結構改變，要和 CMU 對帳，**範圍需重估** | 多數週到數月，不可控 |
| | ④ 鏈上 DvP（代幣化存款或穩定幣） | 指南不支援，要另立專案 | 數月 |
| **持有人正式紀錄的歸屬** | **① 鏈下登記冊為權威** | B 用人工快照（+0–2）；C7 自動化（+5–8） | 無 |
| | ② 鏈上 CMTAT 為權威 | 開發量相近，但需要強力的法律意見，對帳和事件處理要求更嚴 | 法律工作和監管溝通的時間大幅增加 |
| | ③ DigiQuick 擔任登記處 | +15–25，還要受委任或持牌 | 非技術前置很長，不建議 |
| **付息與贖回責任** | **① 付款代理全權負責，DigiQuick 只提供快照** | B 不增加；C7 含快照（+3–5） | 不在首發關鍵路徑上 |
| | ② DigiQuick 計算、付款代理付款 | 付息欄位要結構化（+8–12，浮息再加 3–5） | 要在首次付息前完成 |
| | 贖回 BURN | 和 A6 共用（+1–3） | A 必須驗證；系統化可延到第一次贖回前 |
| **外部地址是否允許** | **① 不允許**（符合 OZ R08.2） | 0 | 無 |
| | ② 首發後開放 | +5–8 | 受 Aegis 主網 KYT 限制，不排日期 |
| | ③ 首發就允許 | +5–8 | **首發日期直接綁在 Aegis 主網 KYT 上**，在那之前會回 422 |
| **Aegis API（dev）何時可用** | 尚未提供 | 只能依 v0.6 用 mock 開發，預留 20–30% 返工（約 +10–20 PD） | A 可以先在 mock 上完成，真環境驗證延後 |
| **Aegis stg 何時可用** | 晚到 | 改在 dev 做 UAT 會有環境差異的風險 | stg 晚多久，B 就延多久 |
| **Aegis 主網何時可用**（含憑證、MASTER_KEY） | 晚到 | — | 主網晚多久，首發就延多久；不開放外部地址就不必等 KYT |

---

## 七、估算假設與檢查

### 7.1 估算口徑
- 1 人日（PD）＝熟悉 NestJS 和 Next.js 的中階工程師工作一天。
- 表中人日不含等待第三方的時間，等待時間在 7.4 另列。

### 7.2 各類工作是否都已計入人日

| 類別 | A | B | C |
|---|---|---|---|
| 產品與流程設計 | A2 | B2 | C1 |
| 前端 | A9 | B6、B7 | C2、C3、C5、C6、C13 |
| 後端與 API 整合 | A3–A7 | B3、B4a、B8 | C4、C5 |
| 資料與對帳 | A10 | B5 | C7 |
| 測試與 UAT | A8、A11 | B9 | C8、C10 |
| 部署 | A12 | B10 | C9 |
| 營運準備 | A13 | B11、B12 | C11 |
| 項目管理 | A1 | B1 | C12 |

### 7.3 「已有平台可改造」與「主要模組從零建置」差異最大的項目

| 工作包 | 改造 | 從零 | 差異 | 原因 |
|---|---|---|---|---|
| A9＋B6 admin 前端 | 12–18 | 22–33 | **約 +10–15** | 從零時沒有 admin 外殼、登入、表格元件和 codegen |
| B8＋C3 權限、稽核、maker-checker | 11–18 | 19–28 | **約 +8–10** | 改造時可以直接擴充 `@WriteAuth` 和 Permission 表 |
| C13 app 發行方提交 | 2–4 | 8–12 | 約 +6–8 | 改造時可以在 `BondAddBtn` 直接加欄位 |
| A7 資料模型 | 4–6 | 7–10 | 約 +3–4 | bond 的 35 個欄位已經有了 |
| 核心 Aegis 整合（A3–A6、A8、A10、B3–B5） | — | — | 約 0–2 | 這部分無論如何都是新開發 |

改造全案約省 30–45 PD。真正的新工作是 Aegis 整合、投資人實體和對帳，這部分兩個區間幾乎一樣。

### 7.4 DigiQuick 開發投入、第三方工作與等待時間

| 版本 | DigiQuick 開發投入 | 第三方工作 | 等待第三方的時間 |
|---|---|---|---|
| A | 54–84 PD | Aegis：實作 API、dev 環境、憑證、testnet 審核 | Aegis dev 交付日未定（「實作完成後提供」）；真環境驗證時審核等待約 1–2 週 |
| B | 71–112 PD，另加 B4b 3–5 | 律師與發行方：文件定稿。券商：KYC 與分配。付款代理：到帳確認。Aegis：stg、主網、審核、CMTAT 審計證據、CFX。其他：滲透測試、登記處 | 律師與券商：預留 4–6 週並行。Aegis stg 與主網：日期未定。主網彩排：1–2 週。投資人開戶：2–3 週。發行窗口：2–3 週，其中首發 Aegis 審核約 5–10 個工作天 |
| C | 61–101 PD | 券商：介接規格。銀行：付款檔格式（若要做） | 很少 |

### 7.5 首次發行的一次性工作，與每新增一筆發行的重複工作

| 項目 | 首次（一次性） | 每新增一筆，B 模式（人工） | 每新增一筆，C 模式 |
|---|---|---|---|
| DigiQuick 開發 | A＋B 125–196 PD（另加 3–5）；要可重複交付再加 C 61–101 | 0 | 0 |
| DigiQuick 營運 | B12 4–8 PD（已含在 B） | 5–10 PD；新發行方加上 20 位以上新投資人時 8–12 PD | 2–4 PD |
| 第三方一次性工作 | Aegis 憑證與環境；法律框架文件；滲透測試；託管協議附錄 | 律師和券商針對每筆發行的工作（不計入 DigiQuick） | 同左 |
| Aegis 審核筆數 | — | 新發行方約 13 筆；已開戶的發行方每檔約 8–9 筆依序審核；每位新投資人 2 筆；allowlist 和 mint 各 ⌈N/20⌉ 筆 | 同左（C 版改善不了 Aegis 端） |
| Aegis 等待（日曆） | — | 首發 5–10 個工作天；之後每檔 3–7 天 | 同左 |

每一筆發行都要重做的工作：條款錄入與覆核、投資人名單與分配、付款確認、Aegis 的 8–13 筆依序審核、發行後三方對帳。Aegis client、webhook、資料模型、admin 面板、runbook 和監控都是一次性工作。

---

## 八、兩位隊員的結論

### 8.1 首發範圍的共同結論

1. **首發前必須由系統完成**：
   - Aegis client 和請求與 webhook 落地（去重、seq、輪詢對帳）。
   - 發行方開戶、4 個錢包、6 個角色（可以由營運逐步觸發）。
   - 投資人最小紀錄和託管錢包（不做投資人前台，也不在 DigiQuick 收 KYC 文件）。
   - bond 的 CMTAT 欄位、條款上傳、部署、allowlist、mint（含分配校驗和付款旗標）。
   - bond 狀態正確顯示。
   - holders 快照匯出。
   - Aegis 後台事件落地。
   - 金鑰移入 Secrets Manager 或 KMS，加上稽核日誌。
   - 主網彩排和 runbook。
2. **首發可以人工處理**：名單與分配的覆核、付款確認、三方對帳（Excel）、四眼覆核（先用稽核日誌代替系統化的 maker-checker）、條款錄入（不改 app）。
3. **可以延後**：
   - BURN_AND_MINT 和 SET_TERMS：hypercare 期間做，最晚在第一筆轉讓前完成。
   - 付息工具：首次付息前完成。
   - 系統化的贖回：第一次贖回前完成，但 BURN 必須在 A 版驗證過。
   - 外部地址與提領：要等 Aegis 主網 KYT。
   - C 版其他項目。
4. **任何版本都不納入，另案處理**：鏈上 DvP、CMU 鏡像、DigiQuick 擔任登記處。

### 8.2 仍有分歧的估算

**數字上已經沒有未解決的分歧。** 第二輪處理過的項目：

| 項目 | pm | bond-expert 第一輪 | 結果 |
|---|---|---|---|
| B5 資料與對帳 | 5–8 | 4–6（認為三方比對交給 Excel 即可） | 採用 5–8。holders 每頁 25 筆、要依序讀，還要定義比對格式 |
| B9 測試與 UAT | 10–15 | 認為回歸部分可能高估 | 維持 10–15。repo 沒有可用的自動化測試，又加了主網彩排 |
| B 增量合計 | 71–112 | 66–104 | 66–104 是加總算錯，已更正 |
| 首發的 Aegis 等待 | 3–7 個工作天 | 首發 5–10 個工作天 | 採用 5–10，發行窗口調為 2–3 週 |

兩位隊員都同意，**剩下的不確定性全來自第三方**，靠 DigiQuick 內部討論無法收斂：
- Aegis 每筆審核的實際耗時：主網每筆 2–4 小時，或半天到 1 天。發行方側的等待因此是 2–4 天或 5–9 天。
- B2、B11 的非技術工作，人日估算不準，日曆主要受律師和券商影響。
- 若 Aegis dev 環境遲遲沒有提供，依 v0.6 開發要預留 20–30% 返工（約 +10–20 PD）。

主會話另外記錄一點：C 版配置在定稿表中寫「約 4」，逐項加總則是 4.7。本報告取「約 4.5 FTE」，對人日沒有影響。

### 8.3 最能縮小估算區間的確認事項（依優先序）

1. **Aegis**：
   - 說明 CMTAT 細節（指南目前沒有寫）：⑧ deployments body、6 個角色、CMTAT 3.0.0 的審計證據、decimals。
   - 給出 dev、stg、主網的交付日期。
   - 承諾審核 SLA。
   - 確認 CFX gas 由誰出、誰補充。
   - 這幾項決定 A 的返工風險，以及 B 的日曆下限。
2. **發行方與律師**：
   - 交收模式。
   - 登記冊權威。
   - 是否經 CMU：若經 CMU，範圍要重估。
   - 是否允許外部地址：若允許，首發就綁在 Aegis 主網 KYT 上。
3. **首發投資人人數**：20 人以內、20–50 人、超過 50 人三種情況做法不同。超過 50 人時 C5 要提前做，開戶時間也會拉長。
4. **付款代理是否全權負責付息與贖回的計算**：若由 DigiQuick 計算，C 版要加 8–12 PD。
5. **投資人 KYC 和專業投資者認定由誰做，DigiQuick 收哪些個資**：決定要不要做投資人前台（+15–25 PD）和存放 KYC 文件。
6. **主網彩排的實測審核時間**：量到之後，可以直接把發行窗口和每筆發行的等待區間收窄。

---

附：兩位隊員的工作底稿（程式碼盤點、流程分析、WBS 草稿、兩輪確認紀錄），位於 `/tmp/claude-1001/-home-huangsq04-dacc-github-push/c2b5c956-d2ab-4cbd-8f65-cc6b61d42bc4/scratchpad/team/`。
