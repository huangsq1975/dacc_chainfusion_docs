以下依 v0.7 串接規格回覆。

**1. 是否有機構投資人？**

有。USDSN27 的發行對象是機構投資人。v0.7 在同一個 `POST /v1/client-requests` 支援這個情境：傳 `kind: "INVESTOR"`、`clientType: "COMPANY"`，並填公司名稱、註冊號碼及聯絡信箱。核准後，機構有自己的投資人帳戶和託管錢包。每個帳戶只有一個業務角色。

```json
{ "referenceId": "dq-investor-org-0001", "externalId": "investor-org-0001",
  "kind": "INVESTOR", "clientType": "COMPANY",
  "companyName": "Example Investment Ltd", "incorporationNo": "87654321", "contactEmail": "ops@example.org" }
```

**2. 個人 INVESTOR 申請還要提供哪些文件？**

端點先按 §7.2 開。除了 `referenceId`、`externalId` 和 `kind: "INVESTOR"`，個人申請只要傳 `firstName`、`lastName`、`birthday`、`countryOfCitizenship`。省略 `clientType` 時預設為 `INDIVIDUAL`，原 v0.6 的寫法仍適用。

串接時不用上傳附件。審核階段是否要收證件影本、住址證明或其他文件，以及如何收件，我這邊目前沒有想法，若你有想法也可以提出來我們一起定義。

**3. CMTAT 3.0.0 怎麼部署、六個角色怎麼指派、decimals 怎麼填？**

先用 `POST /v1/clients/{issuerId}/cmtat/terms-documents` 上傳條款 PDF，取得文件 ID。再用 `POST /v1/clients/{issuerId}/cmtat/deployments` 送出以下 body，將 `<…>` 換成實際值：

```json
{
  "referenceId": "dq-usdsn27-deploy-001",
  "walletId": "<發行方 admin 錢包 ID>",
  "name": "DACCUSD3.50TokenisedStructuredNote2027",
  "symbol": "USDSN27",
  "decimals": 0,
  "tokenId": "SN-03GB2712R-2027",
  "termsName": "USDSN27 Terms",
  "termsDocumentId": "<條款上傳後取得的文件 ID>",
  "information": "<要公開上鏈的產品說明>"
}
```

`decimals` 在部署時指定，範圍是 0–18 的整數，部署後固定。**USDSN27 使用 `0`**，例如發行 100 單位，`amount` 就傳 `"100"`。

部署完成後，從 `result.cmtatTokenId` 取得 ID，再呼叫 `POST /v1/clients/{issuerId}/cmtat/tokens/{cmtatTokenId}/calls`。由 admin 簽署六筆 `GRANT_ROLE`，每筆使用獨立的 `referenceId`，`walletId` 填接受授權的錢包 ID。

| 接受授權的發行方託管錢包 | `role`（每筆選一個） |
|---|---|
| maker | `MINTER`、`BURNER`、`ALLOWLIST` |
| custodian（供 Aegis 操作） | `PAUSER`、`ENFORCER`、`ERC20ENFORCER` |

例如授予 maker 發行權限：

```json
{
  "referenceId": "dq-usdsn27-grant-minter-001",
  "action": "GRANT_ROLE",
  "role": "MINTER",
  "walletId": "<發行方 maker 錢包 ID>"
}
```

三種客戶的申請範例在 §7.2，部署與角色指派在 §7.3。[DigiQuick 串接指南 v0.7](digiquick-integration-guide-v0.7.md)
