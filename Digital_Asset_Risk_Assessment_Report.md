# Digital Asset Risk Assessment Report

**Digital Asset Clearing Center**  
Hong Kong — “DACC.HK”  
Tokenised Bond SFC Technical Assessment

**Date:** 2 November 2026  
**Copyright:** © 2026 Zeppelin Group Ltd.

The contents in this document are provided for informational purposes only and do not constitute professional advice of any kind. Security assessments are limited by factors such as scope, time, and techniques used and therefore this document does not constitute a certification, guarantee, or warranty regarding the proper functioning of any system or the absence of security vulnerabilities.

---

## DACC.HK Tokenised Bond

Technical Assessment against the Appendix to SFC Circular 23EC52

| Field | Detail |
| --- | --- |
| Client | Digital Asset Clearing Center Limited (“DACC.HK”) |
| Subject | DACC29: DACC.HK 5.00% Tokenised Bond due 2029, issued on the DigiQuick V1 platform on Ethereum and Conflux eSpace |
| Reference framework | SFC Circular on intermediaries engaging in tokenised securities-related activities, 2 Nov 2023 (ref. 23EC52), Appendix Parts A and B: ten selected factors |
| Version and date | v1.0 · 2 November 2026 |
| Onchain evidence cut-off | 29 October 2026 (onchain state at Ethereum block 26,297,100 and Conflux eSpace block 160,229,698) |
| Prepared by | OpenZeppelin |
| Classification | Confidential — prepared for DACC.HK |

## Table of contents

1. [Executive Summary](#executive-summary)
2. [Purpose and audience](#purpose-and-audience)
3. [Scope](#scope)
4. [Methodology](#methodology)
5. [Summary of outcomes](#summary-of-outcomes)
6. [System overview](#system-overview)
7. [Factor assessments](#factor-assessments)
8. [Consolidated recommendations](#consolidated-recommendations)
9. [Appendix A: Evidence index](#appendix-a-evidence-index)
10. [Appendix B: Onchain verification log](#appendix-b-onchain-verification-log)
11. [Appendix C: Questionnaire](#appendix-c-questionnaire)
12. [Appendix D: Glossary](#appendix-d-glossary)

Source page numbers: Executive Summary 4; Purpose and audience 6; Scope 6; Methodology 7; Summary of outcomes 11; System overview 13; Factor assessments 17; Consolidated recommendations 48; Appendix A 51; Appendix B 53; Appendix C 54; Appendix D 55.

## Executive Summary

DACC.HK engaged OpenZeppelin to assess the technical readiness of the DACC.HK 5.00% Tokenised Bond due 2029 (DACC29), issued on its DigiQuick V1 platform on Ethereum and Conflux eSpace, against ten factors from the Appendix to the SFC’s Circular on intermediaries engaging in tokenised securities-related activities (23EC52). The aim is to give DACC.HK an evidence-based view of which controls are in place, where they fall short, and what to fix first.

This is an assessment, a point-in-time review based on the client’s specific requirement, followed by a round of questionnaire, the documents DACC.HK supplied and our own onchain checks. It is not a smart contract audit, a certification or a legal opinion.

The deployed contracts match the CertiK-audited code on both networks. They have no owner, upgrade or forced-transfer powers, and they publish no investor personal data. On each network, 2,999,900 of the 3,000,000 tokens are held at a single custody address. The remaining 100 were transferred to a separate wallet. The custodian describes an air-gapped HSM signing setup with a four-person approval chain for withdrawals. DACC.HK states that tokens can be withdrawn from custody only to whitelisted addresses that have passed KYT (Know your transaction) screening.

The gaps are concentrated in the offchain arrangements around the token. Most controls are described in the questionnaire answers but not evidenced. The policies and attestations supplied are generic group documents or cover a different Aegis entity, and the bond documents, which settle how the token relates to the bond, are not final.

### Assessment results

Of the ten factors, one is Addressed, seven are Partially addressed, one is Not addressed and one is Not evidenced. The Summary of outcomes gives the result and headline for each factor.

| Outcome | Factors | Count |
| --- | --- | --- |
| Addressed | 01 | 1 |
| Partially addressed | 02, 03, 06, 07, 08, 09, 10 | 7 |
| Not addressed | 05 | 1 |
| Not evidenced | 04 | 1 |
| Total |  | 10 |

### Themes across factors

- **One bond on two networks, with no reconciliation (factors 02, 04).** Each network carries 3,000,000 tokens for the same US$3,000,000 bond, and there is no defined link to the single register and no reconciliation evidence.
- **A single hot key creates and vouches for the bond tokens (factors 01, 03, 09).** The operator wallet, a single-signer MetaMask key, deployed the factory, created DACC29 and signs the custodian attestation in the token metadata. It sits outside the custodian’s HSM and approval controls.
- **Tokens can leave custody with no onchain limits (factors 07, 08).** The contracts do not restrict transfers, the token terms allow self-custody, and 100 tokens already sit in a wallet whose controller has not been established.
- **The evidence is generic or covers a different entity (factors 03, 05, 07, 09).** The policies are Aegis group templates last reviewed in Sep 2024, the SOC 2 reports cover the US affiliate up to Mar 2023, and the STR evidence is a sales deck. None is specific to DACC.HK or to Aegis Custody Company Limited.

## Purpose and audience

This report sets out OpenZeppelin’s assessment of the DACC.HK tokenised bond and the arrangements surrounding it against ten factors set out in the Appendix to the Circular on intermediaries engaging in tokenised securities-related activities (23EC52), issued by the Securities and Futures Commission (SFC), Hong Kong’s statutory regulator of the securities and futures markets.

This report is a point-in-time technical readiness assessment. It is not a smart contract audit, a certification, a legal opinion or a regulatory approval, and it does not state that any arrangement complies with SFC requirements. OpenZeppelin does not submit this report to the SFC.

## Scope

This report covers the DACC29 tokenised bond (DACC.HK 5.00% Tokenised Bond due 2029), issued through the DigiQuick V1 platform on Ethereum and Conflux eSpace, and assesses it against ten factors from the Appendix to SFC Circular 23EC52: seven from Part A and three from Part B, listed in the Methodology section. DigiQuick V2 (ERC-3643), other assets issued on the platform, testnet deployments and any offchain representations of the bond are excluded. Background on the bond, the token model and the custody structure is given only as context for the factors and carries no outcome. Legal, tax and regulatory compliance matters are outside the scope of this report.

### Scope at a glance

| Item | Detail |
| --- | --- |
| Client | Digital Asset Clearing Center Limited (“DACC.HK”), Hong Kong |
| Product | DACC29: DACC.HK 5.00% Tokenised Bond due 2029. Nominal US$3,000,000 per the token terms, represented on each network by 3,000,000 Ownership Tokens of US$1 each (token ID 1) and one Issuance Token (a companion one-of-one NFT) |
| Platform | DigiQuick V1: TokenFactory (hybrid ERC-1155/ERC-721) with EIP-1167 ERC-20 adapter clones. Not upgradeable, with no owner or admin role. Deployed 24 October 2026 from commit `545d47a` |
| Networks and contracts | Ethereum mainnet and Conflux eSpace mainnet, at the same addresses on both. TokenFactory `0x0d4F619fBDe172a7Dc132E929c8f2BFA23B50804`. ERC-20 adapter `0x7877ca6c7d136efb2eCBA26525018371A8695F7c` |
| Custodian | Aegis Custody Company Limited, a wholly owned subsidiary of DACC.HK (Hong Kong TCSP licence TC006700). Custody address `0x736fD36d84e2dd541623E6Bd5Bb238D7067f2e33` |
| Factors assessed | Part A: A(b)(i), A(b)(iii), A(b)(iv), A(c)(iii), A(d), A(e), A(f). Part B: B(a), B(e), B(f) |
| Onchain evidence cut-off | 29 October 2026 (onchain state at Ethereum block 26,297,100 and Conflux eSpace block 160,229,698) |
| Out of scope | DigiQuick V2 (ERC-3643); the other Appendix factors; test-network deployments and the 2021 legacy factory; a re-audit of the smart contracts; testing of the DigiQuick web applications, APIs and custody systems; legal, regulatory, tax and accounting matters |

## Methodology

Each of the ten factors was mapped to a section of the evidence questionnaire issued to DACC.HK.

### Factors assessed

| # | Appendix ref | Factor | Questionnaire section |
| --- | --- | --- | --- |
| 01 | A(b)(i) | Smart contract audit | 2.1–2.4 |
| 02 | A(b)(iii) | Interoperability between DLT networks and back-end systems | 3.1–3.4 |
| 03 | A(b)(iv) | Administrative controls, policies and procedures | 4.1–4.10 |
| 04 | A(c)(iii) | Extrinsic rights and reconciliation with underlying records | 5.1–5.6 |
| 05 | A(d) | Business continuity planning for DLT-related events | 6.1–6.4 |
| 06 | A(e) | Data privacy | 7.1–7.4 |
| 07 | A(f) | Money laundering and terrorist financing risks; KYC/AML gating | 8.1–8.4 |
| 08 | B(a) | Immobilisation with central custody | 9.1–9.6 |
| 09 | B(e) | Custodian’s private key management | 10.1–10.7 |
| 10 | B(f) | Segregation of client assets | 11.1–11.4 |

### Applicability of Part B

Part B of the Appendix addresses bearer-form tokenised securities using permissionless tokens on public-permissionless networks. DACC29 uses permissionless tokens on two public-permissionless networks. DACC.HK states that the bond is in registered form (Q9.5), but the onchain token terms describe digitally native bonds representing legal and beneficial ownership that may be self-custodied (E-05). Given this uncertainty, and because DACC.HK selected factors B(a), B(e) and B(f), we apply them as stated. Where the bond is finally confirmed as registered form, intermediaries may give these factors less weight; that is a matter for their own judgement.

### Approach

| Step | What we did |
| --- | --- |
| 1. Evidence request | Structured questionnaire (58 questions in 11 sections) issued 19 Oct 2026; responses and 19 evidence files received 26 Oct 2026. |
| 2. Documentary review | Read every document supplied (Appendix A) and checked consistency between documents and with the questionnaire. |
| 3. Onchain verification | Queried both networks directly (Ethereum via a commercial RPC provider; Conflux eSpace via its public RPC and ConfluxScan) for every client-stated deployment, transaction, role holder, balance, approval and event (Appendix B). |
| 4. Code reconciliation | Compiled the supplied source (`545d47a`) with the stated compiler settings and compared the runtime bytecode with both deployments; spot-checked each CertiK fix in the source. |
| 5. Metadata review | Retrieved the IPFS metadata for both networks, compared it across chains and with the questionnaire, and verified the two embedded signatures. |
| 6. Follow-up and readout | One round of clarification questions; readout call; one round of factual corrections before the final issue. |

### Sources of evidence and reliance

We distinguish three classes of evidence: independently verified (onchain data and code we checked ourselves), documented (policies, reports and agreements supplied by DACC.HK, accepted as genuine but not verified with their authors), and client-stated (questionnaire answers without supporting documents). A control supported only by client-stated evidence is generally treated as not evidenced. We relied on the CertiK report for factor 01 without re-auditing the code, and on the SOC 2 reports and penetration test only for the entities, systems and periods they cover.

### Assessment categories

| Category | Definition |
| --- | --- |
| Addressed | The control or arrangement the factor calls for is in place, documented, and supported by evidence we could verify, onchain where applicable. |
| Partially addressed | A control exists but has a material gap: for example, it is undocumented or untested, or it relies on a compensating measure weaker than the factor contemplates. |
| Not addressed | The control is absent, or the evidence shows it does not work as described. |
| Not evidenced | The question was not answered, or was answered without supporting evidence, so we could not assess it. This counts as a gap in the same way as Not addressed. |

A factor’s outcome reflects all of the questions that bear on it.

### Limitations

- Findings reflect the evidence received and onchain state as at the evidence cut-off date.
- Statements by DACC.HK that we could not verify independently are identified as client-stated.

## Summary of outcomes

| # | SFC ref | Factor | Outcome | Headline |
| --- | --- | --- | --- | --- |
| 01 | A(b)(i) | Smart contract audit | Addressed | CertiK audit covers the deployed code; bytecode on both chains matches source; two findings acknowledged. |
| 02 | A(b)(iii) | Interoperability between DLT networks and back-end systems | Partially addressed | One cycle reconciles onchain; no finality/reorg handling, free-tier RPC, no cross-chain reconciliation. |
| 03 | A(b)(iv) | Administrative controls, policies and procedures | Partially addressed | Small contract privilege surface; single-signer hot key creates and attests tokens; token policies not provided. |
| 04 | A(c)(iii) | Extrinsic rights and reconciliation with underlying records | Not evidenced | Register-of-record model described but no reconciliation evidence; bond terms not final and inconsistent. |
| 05 | A(d) | Business continuity planning for DLT-related events | Not addressed | No DLT-specific continuity or migration plan; no DLT scenario ever tested. |
| 06 | A(e) | Data privacy | Partially addressed | No personal data onchain; offchain KYC handling and PDPO coverage weak. |
| 07 | A(f) | Money laundering and terrorist financing risks; KYC/AML gating | Partially addressed | Custodian AML programme exists; no onchain eligibility control; STR evidence is not DACC-specific. |
| 08 | B(a) | Immobilisation with central custody | Partially addressed | Almost all tokens at one custody address; transfers out of custody allowed and already made. |
| 09 | B(e) | Custodian’s private key management | Partially addressed | Sound HSM design described; no attestation for the custodian entity or HSM stack. |
| 10 | B(f) | Segregation of client assets | Partially addressed | Contractual segregation favourable; client attribution and omnibus ledger not evidenced. |

## System overview

This section describes the parties, architecture, token model and lifecycle of DACC29 as stated by DACC.HK, and the deployment as we verified it onchain. It provides context for the factor assessments and is not itself assessed.

### Parties and roles

| Party | Role (as stated) | Source |
| --- | --- | --- |
| DACC.HK (Digital Asset Clearing Center Limited) | Platform operator of DigiQuick; holds the operator wallet; named as issuer in the token metadata | Q1.1, Q1.5; E-05 |
| Aegis Custody Company Limited | Custodian, transfer agent and registrar; Hong Kong TCSP licensee (TC006700) since 2019; wholly owned by DACC.HK | Q1.1, Q10.1; E-14 |
| Guotai Junan Securities (Hong Kong) Limited | Named as bond issuer in Q1.1; named as placement agent in the token metadata; performs investor KYC as broker-dealer | Q1.1, Q8.1; E-05 |
| CMB Wing Lung Bank | Paying agent; also listed as performing KYC | Q1.1 |
| CertiK | Smart contract auditor | E-01 |
| Service providers | Alchemy and Etherscan (Ethereum data), public Conflux RPC and ConfluxScan, Pinata (IPFS), AWS ap-east-1 (hosting), Beosin and Elliptic (KYT), Refinitiv and LexisNexis (screening), TTL (brokerage system) | Q3.4, Q8.1 |

### Architecture

DigiQuick consists of a User Portal (issuers), an Admin Portal (DACC.HK operators), an API and a Listener, running as containers on AWS EC2 in Hong Kong with a PostgreSQL database. The back end holds no keys. Onchain writes are signed in the Admin Portal by the operator wallet (`0x88A3BdF2C2263a76C73854D155280FfEb90fb5Ab`) through MetaMask or WalletConnect. The Listener reads both chains through third-party RPC and explorer APIs. Token metadata is pinned to IPFS through Pinata. Custody of minted tokens is with Aegis Custody Company Limited, which describes an air-gapped Thales HSM cold wallet and an HSM hot wallet API used by broker-dealers.

### Token model

| Token | Standard | Represents | Held by (onchain, both networks) |
| --- | --- | --- | --- |
| Ownership Token (token ID 1) | ERC-1155 fungible balance with an ERC-20 adapter (DACC29, 0 decimals) | US$1 nominal of the bond each; 3,000,000 per network | 2,999,900 at the custody address; 100 at `0x67D1e25180dbAdD501d008E0c288aB2eb6bAbea1` |
| Issuance Token | ERC-721-style one-of-one (companion NFT), created in the same transaction as the Ownership Token | Proof of asset | 1 at the custody address |
| Recording ledger (token ID 1) | Parallel internal ledger | Not used after creation (Q1.4) | 3,000,000 at the operator wallet |

### Bond lifecycle

| Stage | Onchain | Offchain | Actors |
| --- | --- | --- | --- |
| Issuance | `createTokenWithRecording` mints the full supply to the custody address and sets the ERC-20 name | Issuer submits and signs the asset content; Admin Portal review | Issuer, DACC.HK operator |
| Distribution | Custody transfer to a whitelisted address (done once, 100 tokens) | Broker-dealer subscription and payment in fiat; register update | Broker-dealer, registrar, custodian |
| Coupon | None | Paying agent computes and pays in fiat from the register at the record date | Paying agent, registrar, custodian |
| Redemption | Tokens moved to a redemption address (no burn) | Fiat payment; register marks units cancelled; or conversion to shares at the issuer’s discretion | Paying agent, custodian, registrar |

### DACC29 deployment

DACC29 is deployed at the same addresses on Ethereum mainnet and Conflux eSpace mainnet.

| Item | Address or value | Verified on Ethereum and Conflux eSpace |
| --- | --- | --- |
| TokenFactory | `0x0d4F619fBDe172a7Dc132E929c8f2BFA23B50804` | Deployed by the operator wallet at Ethereum block 26,259,693 and Conflux eSpace block 159,805,865 |
| ERC-20 adapter template | `0x711F613D64211Cb1b0257147568DC119Dfd715d7` | Created by the factory (factory nonce 1) on both networks |
| DACC29 ERC-20 adapter | `0x7877ca6c7d136efb2eCBA26525018371A8695F7c` | EIP-1167 clone of the template on both networks |
| Deployer, operator, setting and recording operator | `0x88A3BdF2C2263a76C73854D155280FfEb90fb5Ab` | EOA; 4 transactions on each network |
| Custody address | `0x736fD36d84e2dd541623E6Bd5Bb238D7067f2e33` | 2,999,900 Ownership Tokens and the Issuance Token on each network |
| Out-of-custody holder | `0x67D1e25180dbAdD501d008E0c288aB2eb6bAbea1` | 100 Ownership Tokens on each network; never transacted |
| Issuer signing wallet | `0xDb1eBe734A2636a1f76D639947Cf6D0582f9B255` | Signature in the token metadata verified; no onchain activity |
| Trusted forwarder | Zero address | Verified on both networks |
| Source commit and compiler | `545d47a`; solc 0.8.1, 200 runs | Runtime bytecode matches source on both networks |

## Factor assessments

Each factor follows the same structure: the SFC expectation, what we assessed, the evidence reviewed, our observations, the outcome and our recommendations.

### Factor 01 — A(b)(i) Smart contract audit

| Field | Detail |
| --- | --- |
| Outcome | Addressed |
| Questionnaire | Section 2 (questions 2.1–2.4) |
| Evidence IDs | E-01, E-02, E-03, E-04 |

**SFC expectation.** (b) the technical aspects of the Tokenised Securities, in particular: (i) smart contract deployed (if any). Note: if a smart contract is deployed in the Tokenised Securities’ operation, the intermediary may need to ensure a smart contract audit has been conducted on the smart contract. If the intermediary relies on a smart contract audit conducted by a third party, it should be able to demonstrate that it is reasonable to rely on such smart contract audit. The smart contract audit should focus on reviewing whether the smart contract is not subject to any contract vulnerabilities or security flaws with a high level of confidence.

**What we assessed.** Whether an audit was performed on the code actually deployed, whether all findings were resolved or consciously accepted, and whether reliance on the audit is reasonable. We did not re-audit the code. Areas: audited commit vs. deployed commit vs. onchain bytecode; finding status (14 findings; 12 resolved, 2 acknowledged: DAC-02, DAC-13); acknowledged-risk rationale and compensating controls; reasonableness of reliance (auditor, scope, recency, CertiK post-deployment review).

**Evidence reviewed**

| Evidence | Type | Basis |
| --- | --- | --- |
| E-01 | CertiK final report: 14 findings (1 Critical, 1 Medium, 5 Minor, 6 Informational, 1 Centralization); 12 resolved, 2 acknowledged | Document reviewed |
| E-02 | Contracts source at `545d47a` (CertiK’s final fix commit): all CertiK fixes spot-checked as present | Source code, independently verified |
| E-03 | Questionnaire 1.3, 2.1–2.4 | Client-stated |
| E-04 | Deployed runtime bytecode on both networks compared with a local compilation of E-02 | Onchain, independently verified |

**Observations**

- An independent smart contract audit by CertiK was completed before deployment. The final report is dated 6 Aug 2026; the contracts were deployed on 24 Oct 2026. The report lists the commits reviewed up to `32fdcb5` but does not name CertiK’s final fix commit, `545d47a`, which is the commit deployed. The report’s findings and resolution statuses are accurate for `545d47a` (E-01, E-02, E-04).
- The critical finding (DAC-01: NFTs could be taken through the ERC-20 adapter without an ownership check) and the Medium finding (DAC-03) are marked Resolved. We confirmed the fixes for the Critical and Medium findings, and a sample of the others, are present in `545d47a` (E-02).
- The deployed code matches the source. We compiled the `545d47a` source with solc 0.8.1, optimizer 200 runs. On both networks, the deployed TokenFactory and ERC-20 adapter template are identical byte for byte to our compiled code, except for the compiler’s metadata hash at the end, which is expected to differ between build environments (E-04).
- Both deployments were made with the GSN trusted forwarder set to the zero address, so meta-transactions are disabled. This removes the forwarder-related risks CertiK described in DAC-02 and DAC-10 (E-04).
- No contracts outside the audit scope are deployed. The DACC29 adapter is a standard EIP-1167 clone of the audited template, and the factory has created only the template and this one clone (E-04).

**Gaps and inconsistencies**

- DAC-02 (centralization) was acknowledged on a basis that does not match the questionnaire. DACC.HK told CertiK that the privileged roles are held by “controlled custodian-operated keys”. The questionnaire says the operator, setting operator and recording operator are a single-signer MetaMask wallet held by DACC.HK’s operations Manager (E-01 DAC-02; E-03 Q1.5, Q4.1). Factor 03 assesses what this means.
- CertiK said it would review the role holders once the contracts were deployed. No post-deployment update or status change was provided (E-01 DAC-02).
- DAC-13 was acknowledged on the basis that users would be told, in user-facing documentation, about the shared ERC-721/ERC-1155 approval. That documentation was not provided.
- Apart from CertiK, there has been no other review: no penetration test of the DigiQuick portals and API, and no bug bounty (E-03 Q2.3). This matters for factors 02 and 03 rather than for the contract audit itself.

**Outcome rationale.** A third-party audit covering every deployed contract was completed, all Critical, Medium and Minor findings were resolved, and the two acknowledged items were disclosed. We tied the code running on both networks to CertiK’s final fix commit by independently matching the bytecode. These points support reliance on the audit. Whether reliance is reasonable for its purposes is for each intermediary to decide. The remaining items concern documentation and the accuracy of one acknowledgement, not the audit’s coverage.

**Recommendations**

| ID | Recommendation | Priority |
| --- | --- | --- |
| R01.1 | Ask CertiK to name the final fix commit (`545d47a`) in its report, and publish the verified source links alongside the report. | Low |
| R01.2 | Correct the DAC-02 acknowledgement with CertiK so it reflects the actual key arrangement, and ask CertiK to complete its post-deployment review of the role holders. | Medium |
| R01.3 | Publish the disclosure promised for DAC-13. | Low |

### Factor 02 — A(b)(iii) Interoperability between DLT networks and back-end systems

| Field | Detail |
| --- | --- |
| Outcome | Partially addressed |
| Questionnaire | Section 3 (questions 3.1–3.4) |
| Evidence IDs | E-03, E-04, E-05, E-16 |

**SFC expectation.** (b) the technical aspects of the Tokenised Securities, in particular: (iii) inter-operability issues between DLT networks and the back-end systems of the product issuer and other parties throughout the security lifecycle (such as custodians/wallet service providers).

**What we assessed.** Whether the onchain records and DACC.HK’s back-end systems stay consistent across the whole bond lifecycle on Ethereum and Conflux eSpace, and how failures between them are handled. Areas: lifecycle flows (issuance, subscription, coupon, redemption); read/write paths (Admin Portal, API, Listener); finality, reorg and RPC-dependency handling; cross-chain (Ethereum / Conflux eSpace) consistency.

**Evidence reviewed**

| Evidence | Type | Basis |
| --- | --- | --- |
| E-03 | Questionnaire 1.4, 1.5, 3.1–3.4 | Client-stated |
| E-04 | Issuance and transfer cycle traced onchain on both networks | Onchain, independently verified |
| E-05 | Token metadata on IPFS for each network | Onchain/IPFS, independently verified |
| E-16 | RWA Tokenisation deck (workflow) | Document, context only |

**Observations**

- The back end holds no signing keys; every onchain write is signed in the Admin Portal by the configured operator wallet, and the portal checks the connected address (E-03 Q3.2).
- One complete issuance-and-transfer cycle took place on each network. We matched every transaction the client listed (creation, ERC-20 naming, and the 100-token transfer) onchain, with the same balances on both networks: 2,999,900 tokens plus the Issuance Token at the custody address, and 100 tokens at `0x67D1e25180dbAdD501d008E0c288aB2eb6bAbea1` (E-04).
- Coupon, redemption and subscription payments are offchain, and no tokens move for coupons. This limits the number of onchain and offchain interactions that need to stay in step (E-03 Q3.1).
- Database writes from the Listener are idempotent, and missed events are picked up by a re-read every 10 minutes (E-03 Q3.2).

**Gaps and inconsistencies**

- No finality or reorganisation handling. The Listener processes blocks at the chain head with no confirmation threshold and does not detect or roll back reorganised blocks. DACC.HK acknowledges this as “not in place” (E-03 Q3.2).
- Weak infrastructure dependencies. Ethereum access relies on Alchemy’s free plan (no SLA) plus the Etherscan free API. Conflux eSpace uses the public RPC endpoint and ConfluxScan only. There is no own node. Q3.4 says a second RPC provider is tested, but it is not named and no automatic failover is evidenced. Metadata is pinned on a free Pinata plan (E-03 Q3.4, Q7.1).
- The two networks are not reconciled with each other. DACC29 exists as two separate sets of 3,000,000 tokens with the same asset ID (DIG78-1) and the same US$3,000,000 description. No process explains how one bond position maps to tokens on two chains, or which chain prevails if they diverge (E-04, E-05; see factor 04).
- No automated check exists for transactions dropped after the Admin Portal is closed (E-03 Q3.2).
- Subscription and redemption depend on broker-dealer, paying-agent and registrar steps that run outside any system interface. No interface specification, message format or reconciliation evidence was provided, and the evidence folder for this factor was empty.
- The lifecycle described in Q3.1 (the custodian credits tokens to the investor’s custody position) differs from what happened onchain: tokens were transferred out of custody to a separate wallet (E-03 Q3.3; E-04).
- There has been no penetration test or security review of the DigiQuick portals, API or Listener (E-03 Q2.3, Q4.9).

**Outcome rationale.** The onchain records for the one completed cycle are consistent with what the client described, and the back-end write path is controlled. However, the controls that keep the back end consistent with the chains are missing: there is no finality or reorg handling, no service-level backing for the data providers, and no defined way to keep the two network representations of one bond aligned. No interface or reconciliation evidence was supplied. These are material gaps, so the factor is Partially addressed.

**Recommendations**

| ID | Recommendation | Priority |
| --- | --- | --- |
| R02.1 | Add a confirmation threshold and reorg detection/rollback to the Listener for each network, and record finality status against each stored event. | High |
| R02.2 | Define and document how one bond position is represented across Ethereum and Conflux eSpace, including a regular cross-chain supply and holder reconciliation. | High |
| R02.3 | Move to paid RPC plans with SLAs, or run own nodes, and add a second independent provider per network with tested failover. | Medium |
| R02.4 | Commission a penetration test of the DigiQuick portals, API and Listener. | Medium |

### Factor 03 — A(b)(iv) Administrative controls, policies and procedures

| Field | Detail |
| --- | --- |
| Outcome | Partially addressed |
| Questionnaire | Section 4 (questions 4.1–4.10) |
| Evidence IDs | E-02, E-03, E-04, E-05, E-06, E-07, E-12, E-13 |

**SFC expectation.** (b) the technical aspects of the Tokenised Securities, in particular: (iv) robust and properly maintained policies and procedures, systems and controls underpinning the operation of the Tokenised Securities to manage eg, the private key and risks of theft, fraud, errors and omissions, and cybersecurity risks, etc. Note: For example, adequate administrative controls in the form of transfer restrictions, mint-and-burn mechanism, transaction reversals or redemption procedures should be implemented to protect investors’ ownership interest, which is of particular importance for Tokenised Securities deployed on public-permissionless networks.

**What we assessed.** Who can do what to the tokens, how those powers are protected and exercised, and what happens when something goes wrong. Where the contracts themselves have no restriction, we assess the operational and custodial controls that stand in for it. Areas: privileged roles and key custody (operator, setting operator, recording operator); transfer restrictions, mint/burn, reversal and redemption controls; maker-checker and change management; policy coverage, ownership and review cadence; monitoring and incident response.

**Evidence reviewed**

| Evidence | Type | Basis |
| --- | --- | --- |
| E-02 | Contract roles and functions (source review) | Source code, independently verified |
| E-03 | Questionnaire 1.5, 4.1–4.10 | Client-stated |
| E-04 | Role holders, approvals and full event history on both networks | Onchain, independently verified |
| E-05 | Issuer and operator signatures in the token metadata | Onchain/IPFS, independently verified |
| E-06 | ScienceSoft penetration test (Aegis Vault) | Document reviewed |
| E-07 | Nine Aegis Custody policies | Document reviewed |
| E-12, E-13 | SOC 2 Type 1 and Type 2 reports (Aegis Trust Company) | Document reviewed |

**Observations**

- The contracts have a small privileged surface. There is no owner, admin, pause, mint-after-creation, burn, upgrade or forced-transfer function. Supply is fixed at creation, and no role can move another holder’s DACC29 balance (E-02).
- We checked onchain what the operator wallet can still do to DACC29. It can no longer change anything material: the ERC-20 name and symbol are already set and can be set only once, and holding-time accounting is not enabled for this token. What remains is a parallel internal ledger in the contract, the “recording ledger”, in which the operator holds 3,000,000 units that have never been used, and the ability to hand over the recording-operator role (E-02, E-04).
- No ApprovalForAll or Approval events have been emitted on either network, so no third party is approved to move holders’ tokens (E-04).
- DACC.HK states that movements out of custody are protected by the custodian’s approval chain: one Supervisor, two Auth Officers and one Auth Manager, plus whitelisting and KYT. This is assessed in factor 09 (E-03 Q4.2, Q10.4).
- The issuance was tied to the bond nominal through content signed by both the issuer and the operator wallet. We confirmed both signatures recover to the listed addresses (E-05).

**Gaps and inconsistencies**

- A single hot key underpins issuance and authenticity. `0x88A3BdF2C2263a76C73854D155280FfEb90fb5Ab` is the deployer, the setting and recording operator, and the address labelled “Custodian (Aegis)” that signs the attestation in the token metadata (E-04, E-05). The questionnaire describes it as a single-signer MetaMask wallet held by DACC.HK’s Operations Manager (Q1.5, Q4.1), yet Q4.6 says the custody function holds it. Because the factory lets anyone create tokens, DACC.HK’s stated way to identify a genuine token is that it was created and signed by this address (Q4.3). Whoever holds this one key could therefore create tokens that pass that test.
- There are no onchain transfer restrictions. The investor-protection controls the SFC note lists (transfer restrictions, reversal, redemption) exist only offchain, through custody procedures and the “void ab initio” clause in the terms (Q4.2, Q4.4). Tokens have already left custody once (factor 08).
- None of the seven token-operation policies listed in Q4.7 were provided. These cover issuance and redemption, transfer eligibility, freeze and recovery, deployment and change, key management, reconciliation, and migration. The nine policies supplied (E-07) are generic Aegis group templates. They were last reviewed on 30 Sep 2024, have no approver or version history, and do not mention tokenised securities, smart contracts or the DigiQuick platform. Some contain unfilled template fields.
- Monitoring and alerting are largely described as planned. SIEM is “(planned)”. Q4.8 says current 24-hour response-time targets are defined; faster targets are proposed but not confirmed. Identity, network and vulnerability controls for DigiQuick are also mostly described as planned (Q4.9).
- Independent assurance does not cover the platform. The penetration test (E-06) covers the Aegis Vault web portals, was tested in Dec 2022 (report dated 31 Jan 2023) and gives no findings list. The SOC 2 reports (E-12, E-13) cover Aegis Trust Company (South Dakota) for 2022–23. Neither covers DigiQuick or Aegis Custody Company Limited.
- Redeemed tokens cannot be burned. They are to be moved to a dedicated redemption address and marked cancelled offchain, so onchain supply will never reflect redemptions (Q4.5).

**Outcome rationale.** The contract design keeps privileged powers small, and custody approvals stand in for the missing onchain transfer controls. However, the key that creates and attests tokens is a single-signer hot wallet, and the answers are inconsistent about who holds it. The written procedures for token operations were not provided, and the policies that were supplied are generic, out of date and not specific to DLT. Several monitoring and security controls are still planned. Material gaps remain, so the factor is Partially addressed.

**Recommendations**

| ID | Recommendation | Priority |
| --- | --- | --- |
| R03.1 | Move the deployer, operator and attestation role to a multi-signature or HSM-backed key held under the custodian’s approval controls, and publish the role-holder addresses. | High |
| R03.2 | Provide the seven token-operation policies listed in Q4.7, with owners, approval records and review history, and bring the group policies up to date for DLT operations. | High |
| R03.3 | Implement the planned monitoring, alerting and response targets (Q4.8), including alerts for any `createToken` call from the operator address and any movement from custody addresses. | Medium |
| R03.4 | Commission a penetration test of the DigiQuick platform and refresh the Aegis assurance so it covers the entity that holds the bond tokens. | Medium |

### Factor 04 — A(c)(iii) Extrinsic rights and reconciliation with underlying records

| Field | Detail |
| --- | --- |
| Outcome | Not evidenced |
| Questionnaire | Section 5 (questions 5.1–5.6) |
| Evidence IDs | E-03, E-04, E-05, E-14 |

**SFC expectation.** (c) the legal and regulatory status associated with the Tokenised Securities, in particular: (iii) enforceability of any rights extrinsic to the Tokenised Securities and the potential impact of the Tokenised Securities’ trading activity on the underlying markets. Note: For example, whether the ownership interest of the underlying assets of the Tokenised Securities are legally valid under the applicable governing laws, whether there are regular reconciliations between the records for the extrinsic rights and the records for the distributed ledger tokens, or whether any encumbrance affects the extrinsic rights.

**What we assessed.** How the Ownership Token relates to the bond, which record is authoritative, how often and how well the records are reconciled, and what happens when they disagree. Several of these questions have a legal side. We assessed them against DACC.HK’s stated position and the documents provided, and checked whether that position is consistent with the on-chain records and the operating controls described. We did not assess whether the position is valid or enforceable under Hong Kong law, and nothing in this section is a legal opinion.

Areas covered: the token-to-bond legal relationship, as stated by DACC.HK; the register of record (on-chain or off-chain); reconciliation frequency, method and break handling; record date, coupon and redemption mechanics; encumbrance and secondary trading.

**Evidence reviewed**

| Evidence | Type | Basis |
| --- | --- | --- |
| E-03 | Questionnaire 1.1, 1.2, 1.4, 5.1–5.6, 9.5 (the 5.1 answer is cut off) | Client-stated |
| E-04 | Supply and holders on both networks | Onchain, independently verified |
| E-05 | Bond terms embedded in the token metadata | Onchain/IPFS, independently verified |
| E-14 | Custody agreement (redacted, unsigned template) | Document reviewed |

**Observations**

- DACC.HK states that the offchain register kept by the registrar (Aegis Custody Company Limited) is the register of record. It also states that the offchain register prevails if the records disagree, and that a mismatch stops movements of the bond (Q5.1, Q5.3).
- A daily balance check across onchain balances, custody records and the register is described, with sign-off by the custody team (Q5.2).
- Onchain supply on each network (3,000,000 tokens) matches the nominal stated in that network’s metadata (E-04, E-05).

**Gaps and inconsistencies**

- No reconciliation evidence. No reconciliation report, register extract, sign-off record or break log was provided, and the evidence folder for this factor was empty. The recording ledger, which could support onchain bookkeeping, is not used (Q1.4).
- Two token sets, one bond. Each network carries 3,000,000 tokens whose metadata describes the same asset (ID DIG78-1, “USD 3,000,000”, “3,000,000 Bond Tokens”), and Q1.2 describes US$3,000,000 issued “on each network”. The same investor wallet holds 100 tokens on each network. Without a reconciliation across both networks, the 6,000,000 tokens in total could be read as twice the bond nominal (E-04, E-05).
- The bond terms and the questionnaire describe the instrument differently. The token metadata describes “digitally native bonds” where the token represents “legal and beneficial ownership”, states “Trustee / Fiscal Agent: None”, and says “Bond Tokens may be self-custodied by holders” (E-05). Q5.1 describes a tokenised bond “constituted by off-chain bond documents (such as a trust deed)”, where transferring the token does not by itself transfer legal title, and Q9.5 says the bond is in registered form. Q6.2 says the bond documents are not finalised.
- Q1.1 names Guotai Junan Securities (Hong Kong) Limited as the issuer. The metadata names DACC.HK as issuer and Guotai Junan as placement agent (E-05).
- The 5.1 answer is cut off, and the terms that make an off-register transfer “void ab initio” were not provided, so the legal link between token and bond cannot be checked (Q4.4, Q5.1).
- Encumbrance: Q5.5 says pledging is not permitted. The custody agreement, however, gives the custodian a right to retain or set off client assets against any obligations the client owes it (E-14 §15.8).

**Outcome rationale.** The arrangement DACC.HK describes (an offchain register of record, a daily three-way reconciliation, and the offchain register prevailing) would fit the factor. However, none of it is supported by evidence, the bond documents are not final, and the onchain terms contradict the questionnaire on the nature of the instrument and on who the issuer is. Representing one bond on two networks also has no defined reconciliation. We could not assess the control, so the factor is Not evidenced.

**Recommendations**

| ID | Recommendation | Priority |
| --- | --- | --- |
| R04.1 | Finalise the bond documents and align the onchain metadata with them: issuer, form of the bond, role of the token, trustee, self-custody and the network(s). | High |
| R04.2 | Provide a dated reconciliation report and register extract showing that onchain balances on both networks, custody records and the register agree, with sign-off. | Medium |
| R04.3 | Document how the two network representations relate to the single register, and how total supply across both networks is kept equal to the outstanding nominal. | Medium |

### Factor 05 — A(d) Business continuity planning for DLT-related events

| Field | Detail |
| --- | --- |
| Outcome | Not addressed |
| Questionnaire | Section 6 (questions 6.1–6.4) |
| Evidence IDs | E-03, E-05, E-07, E-08 |

**SFC expectation.** (d) business continuity planning (eg, redeployment plans to migrate DLT-related records to a different blockchain) for DLT-related events which are not adequately addressed with other measures.

**What we assessed.** Whether a plan exists for each DLT-related event, whether the migration or redeployment path is credible and tested, and whether offchain records are protected well enough to rebuild from. Areas: coverage of DLT events (fork, outage, key loss, unauthorised transfer, contract defect); the migration and redeployment path; testing evidence and recency; RTO, RPO and backup protection.

**Evidence reviewed**

| Evidence | Type | Basis |
| --- | --- | --- |
| E-03 | Questionnaire 5.1, 6.1–6.4 | Client-stated |
| E-05 | Fork and business continuity clause in the token terms | Onchain/IPFS, independently verified |
| E-07 | Aegis BC/DR Plan and Security Incident Response Plan (Sep 2024) | Document reviewed |
| E-08 | Two tabletop exercise records, Aegis Custody Co., Ltd (Jan 2023) | Document reviewed |

**Observations**

- The token terms include a fork clause: if the blockchain forks or changes materially, the issuer decides which version is recognised and notifies holders within 5 Hong Kong business days (E-05).
- The offchain register is the record of title, which in principle allows onchain positions to be rebuilt (Q5.1).
- A full restore of the production database was performed on 24 Oct 2026 during the move to AWS Hong Kong. Database point-in-time recovery and daily volume snapshots are in place (Q6.4).
- A custody-key backup and recovery procedure is described (factor 09).

**Gaps and inconsistencies**

- No DLT-specific continuity plan was provided. The token terms state that the platform operator “maintains a business continuity plan covering system outages, data loss and cyber attacks” (E-05), but no DACC.HK plan was supplied. The BC/DR and incident response plans provided (E-07) are generic Aegis group templates covering cloud and office outages and data breaches. They contain no chain outage, fork, reorganisation, contract-compromise, key-loss or migration scenario. Their backup frequency and retention fields are blank, and their RTO and RPO are “to be defined”.
- The Q6.1 answer describes ten scenarios in general terms (“should”, “must be confirmed”, “where configured”) rather than DACC.HK’s actual procedures. The Q6.3 answer reads as guidance, not a record: it says the details of the latest test “have not been supplied and must be confirmed by Operations”.
- No migration plan exists for the bond. Q6.2 describes intended steps only. The bond documents are not final, so it is not known whether they allow migration. No trigger criteria, decision record, snapshot procedure or timeline is documented.
- The only tests on record are two tabletop exercises from Jan 2023 (E-08). They were run by Aegis Custody Co., Ltd in Taipei, are not DLT-related, and contain no action register or follow-up. No DLT event has ever been tested.
- Backups are kept in the same AWS region as production, with no immutable or offline copy and short retention: 7 days for the database and 14 days for volumes (Q6.4).
- The 24-hour RTO and RPO stated in the questionnaire do not appear in any supplied plan (Q6.4; E-07).

**Outcome rationale.** The factor asks for business continuity planning for DLT-related events, including how records would be moved to another blockchain. Apart from the fork clause in the token terms, no such plan was evidenced. The supplied plans are generic and incomplete, the questionnaire answers describe intentions, and no relevant test has been carried out. The control is absent, so the factor is Not addressed.

**Recommendations**

| ID | Recommendation | Priority |
| --- | --- | --- |
| R05.1 | Write a DLT continuity runbook covering chain outage or finality failure, fork, contract defect, operator or custody key loss, RPC failure and custodian failure. Name the owners, decision rights and investor notification steps. | High |
| R05.2 | Document and approve a migration and redeployment procedure (snapshot source, re-issuance, invalidation of old tokens, timelines), and make sure the bond documents permit it. | High |
| R05.3 | Run and record a tabletop exercise on at least one DLT scenario (for example, operator key compromise or a Conflux eSpace outage) with an action register. | Medium |
| R05.4 | Add a cross-region, immutable backup copy of the platform database and uploaded documents, and set retention in line with record-keeping obligations. | Medium |

### Factor 06 — A(e) Data privacy

| Field | Detail |
| --- | --- |
| Outcome | Partially addressed |
| Questionnaire | Section 7 (questions 7.1–7.4) |
| Evidence IDs | E-03, E-04, E-05, E-07 |

**SFC expectation.** (e) appropriate measures to address data privacy risks (eg, as public blockchains store transaction data publicly, this may expose sensitive information of token holders if their identities are revealed).

**What we assessed.** What the contracts and metadata put on public networks, whether an observer can link addresses or amounts to investors, and how the offchain identity data is protected. Areas: onchain and IPFS data exposure; linkability of addresses to investors; offchain KYC data handling; alignment with Hong Kong’s Personal Data (Privacy) Ordinance (PDPO), stated as fact only.

**Evidence reviewed**

| Evidence | Type | Basis |
| --- | --- | --- |
| E-03 | Questionnaire 3.4, 7.1–7.4, 8.1 | Client-stated |
| E-04 | Holder addresses and transfers on both networks | Onchain, independently verified |
| E-05 | Token metadata on IPFS (both networks) | Onchain/IPFS, independently verified |
| E-07 | Aegis Data Classification and Data Retention and Disposal policies (Sep 2024) | Document reviewed |

**Observations**

- No investor personal data is on the public record. We reviewed the metadata on both networks. It holds the bond terms, issuer details, two signer addresses and their signatures, and no investor names or identifiers. The ERC-20 name and symbol contain no personal data (E-05).
- The onchain URI is the IPFS content identifier, not a gateway URL, so the metadata cannot be changed without changing the identifier recorded onchain (E-04, E-05; Q7.1).
- While tokens stay at the single custody address, individual investor positions are not visible onchain (Q7.2).
- The link between addresses and investor identities is held offchain: by the broker-dealer (Guotai Junan, through TTL’s brokerage system) for investors onboarded through it, and by the custodian for its own clients (Q7.3).

**Gaps and inconsistencies**

- Positions held outside custody become visible. The 100 tokens at `0x67D1e25180dbAdD501d008E0c288aB2eb6bAbea1` show an individual balance on both networks (E-04; Q7.2).
- Offchain KYC data is weakly protected. KYC submissions are stored in a Google Drive folder and sent to the onboarding team by email. Asset documents marked “encrypted” are not encrypted at application level (Q7.3, Q8.1).
- The privacy policy (last edited 30 Apr 2022) refers to US, EEA and UK law and does not address the PDPO. None of the supplied policies mentions the PDPO (Q7.3, E-07).
- The question on data-processing agreements was not answered, and none was provided for the KYC tools, Pinata, Mailgun or Google. The Pinata account is on a free plan owned by the engineering team (Q3.4, Q7.1, Q7.3).
- Investor disclosure about the public nature of blockchain records is limited to a link to the Terms of Use and privacy policy (Q7.4). The evidence folder for this factor was empty.

**Outcome rationale.** Onchain exposure is well controlled: no personal data is published, and positions in custody cannot be attributed to investors. However, the handling of offchain identity data has material gaps: email and shared-drive storage, documents marked “encrypted” that are not encrypted, no PDPO coverage and no processor agreements. The factor is Partially addressed.

**Recommendations**

| ID | Recommendation | Priority |
| --- | --- | --- |
| R06.1 | Move KYC submissions out of email and shared drives into an access-controlled, encrypted store, and actually encrypt documents marked “encrypted”. | High |
| R06.2 | Update the privacy policy and data policies to address the PDPO, and put data-processing agreements in place with each processor. | Medium |
| R06.3 | Explain clearly to investors what becomes public onchain, particularly for positions held outside the custody address. | Low |

### Factor 07 — A(f) Money laundering and terrorist financing risks; KYC/AML gating

| Field | Detail |
| --- | --- |
| Outcome | Partially addressed |
| Questionnaire | Section 8 (questions 8.1–8.4) |
| Evidence IDs | E-02, E-03, E-04, E-05, E-09, E-10 |

**SFC expectation.** (f) money laundering and terrorist financing risks associated with the Tokenised Security (eg, whether there are administrative controls over the Tokenised Securities being transferred to parties which have not undergone know-your-client and anti-money laundering procedures).

**What we assessed.** Whether a token can reach a party that has not completed KYC and AML checks. The onchain perimeter controls are covered in factors 03 and 08; this section covers the KYC and AML process around them. Areas: ownership of KYC and customer due diligence across the parties (issuer, broker-dealer, paying agent, custodian); gating before transfer (whitelisting, KYT); sanctions and PEP screening; the suspicious transaction reporting (STR) process.

**Evidence reviewed**

| Evidence | Type | Basis |
| --- | --- | --- |
| E-02 | Transfer functions in the contract source | Source code, independently verified |
| E-03 | Questionnaire 1.1, 7.2, 8.1–8.4 | Client-stated |
| E-04 | Transfer of 100 tokens out of custody on both networks | Onchain, independently verified |
| E-05 | Investor eligibility, transfer restriction and minimum subscription in the token terms | Onchain/IPFS, independently verified |
| E-09 | Aegis Custody Company Limited AML & CTF Policy (30 Sep 2024) | Document reviewed |
| E-10 | “DACC.HK — STR Filing Process” deck (15 pages) | Document reviewed |

**Observations**

- The custodian, Aegis Custody Company Limited, is a TCSP licensee supervised for AML/CTF by the Companies Registry under the AMLO. Its AML & CTF Policy covers risk assessment, customer due diligence and enhanced due diligence, sanctions and PEP screening (Refinitiv, LexisNexis), KYT (Beosin), ongoing monitoring, STR reporting to the JFIU, and record keeping for five years (E-09).
- DACC.HK states that withdrawals from custody go only to whitelisted client addresses that pass KYT screening, and the platform will not create assets for an issuer whose KYC has not been approved (Q8.1–8.3).
- The token terms limit holders to onboarded Professional Investors and state that non-compliant transfers are void and will not be registered (E-05).

**Gaps and inconsistencies**

- The contracts do not stop transfers to parties that have not been onboarded. Once tokens leave custody, any holder can transfer them to any address, and the only protection is the offchain “void ab initio” rule (E-02, E-05). Tokens have already left custody once, to a wallet whose controller and KYC status have not been evidenced (E-04; factor 08).
- Responsibility for investor KYC is split between Guotai Junan, CMB Wing Lung Bank and Aegis and is not settled. Q8.1 says eligibility and who performs investor KYC are “planned to be set for each issue in the bond documents”, which are not final.
- The STR evidence is not a DACC.HK procedure. Q8.1, Q8.2 and Q8.4 refer to it for the STR process. The 15-page file supplied as “DACC.HK — STR Filing Process” (E-10) is an Aegis and Beosin product presentation for brokers’ spot-ETF in-kind transfers, with general guidance on filing STRs with the JFIU. It names no DACC.HK roles, escalation steps or records, and does not mention bonds or tokenised securities.
- The AML policy lets a client override screening: a client can “set a flag to ignore the block list” to force through a withdrawal that hit a screening rule (E-09, Appendix II). This conflicts with Q8.2, which says a withdrawal that fails screening cannot be created.
- The AML policy contains Singapore template text, names no Compliance Officer or MLRO, sets no fixed cadence for independent audit and has no Travel Rule procedure. It does not mention Elliptic or reporting to the SFC, both of which Q8.1 claims (E-09).
- The token terms set a minimum subscription of USD 3,000,000, the whole issue, in integral multiples of USD 1,000. Yet 100 tokens (US$100 nominal) were transferred to a separate wallet on each network (E-04, E-05).

**Outcome rationale.** A KYC and AML programme exists at the custodian, and withdrawals from custody are gated by whitelisting and KYT. However, the tokens carry no onchain eligibility control, responsibility for investor KYC is not yet fixed, the STR evidence supplied is not a DACC.HK procedure, and the policy allows screening to be overridden. Material gaps remain, so the factor is Partially addressed.

**Recommendations**

| ID | Recommendation | Priority |
| --- | --- | --- |
| R07.1 | Fix in the bond documents which party performs investor KYC and eligibility checks before any token reaches a holder, and evidence those checks for the existing out-of-custody holder. | High |
| R07.2 | Remove or tightly control the client override of the blocklist, and provide a DACC.HK-specific STR procedure with a named Compliance Officer and MLRO. | High |
| R07.3 | Consider onchain transfer restrictions (for example, an allowlist) for tokens that may leave custody. | Medium |
| R07.4 | Update the AML policy: Hong Kong references only, a Travel Rule procedure, a fixed independent-audit cadence, and the screening tools actually used. | Medium |

### Factor 08 — B(a) Immobilisation with central custody

| Field | Detail |
| --- | --- |
| Outcome | Partially addressed |
| Questionnaire | Section 9 (questions 9.1–9.6) |
| Evidence IDs | E-02, E-03, E-04, E-05 |

**SFC expectation.** (a) the immobilisation of the Tokenised Securities with central custody. Generally, financial instruments are “immobilised” by being safekept with a custodian in such way that it is not possible for them to enter into circulation and by having the entitlement to them recorded with account-entry records and transferred with book entries.

**What we assessed.** Whether all tokens are safekept with a custodian so that they cannot enter circulation, and whether investor entitlements are recorded and transferred by book entry, as the SFC’s footnote describes. Areas: the custody model and how each holding address is classified; whether and how tokens can leave custody; book-entry recording of entitlements.

**Evidence reviewed**

| Evidence | Type | Basis |
| --- | --- | --- |
| E-02 | Transfer functions in the contract source | Source code, independently verified |
| E-03 | Questionnaire 9.1–9.6, including the custody-model image in 9.1 | Client-stated |
| E-04 | Holders, balances, approvals and transaction history on both networks; gas funding of the custody address | Onchain, independently verified |
| E-05 | Custody and self-custody statements in the token terms | Onchain/IPFS, independently verified |

**Observations**

- The whole issue was minted directly to a single custody address. On both networks that address holds 2,999,900 Ownership Tokens and the Issuance Token, which has never left it (E-04).
- No ERC-1155 or ERC-721 approvals and no ERC-20 allowances exist on either network, so no third party can move tokens out of the custody address (E-04; Q9.6).
- DACC.HK states that a movement out of custody needs a request from an authorised client user, a whitelisted destination that passes KYT screening, and four custodian approvals before the HSM signs (Q9.2, Q9.3; factor 09).
- The custodian states that it keeps book-entry records of each client’s custody wallets and the tokens held in them (Q9.4).

**Gaps and inconsistencies**

- Tokens can and do leave custody. 100 Ownership Tokens were sent from the custody address to `0x67D1e25180dbAdD501d008E0c288aB2eb6bAbea1` on each network. That address has never sent a transaction. Q9.1 calls it an “investor wallet” and Q9.6 says investors hold no keys, yet Q9.1 also lists “investor-controlled wallet” as a supported category. Who controls it has not been established (E-04).
- The design allows tokens to leave custody. The token terms say “Bond Tokens may be self-custodied by holders” (E-05). Q9.2 says Ownership Tokens “technically can leave the Custodian” and that sending them to investors outside custody “depends on regulations in Hong Kong”. It also says tokens sent to broker-dealers are “theoretically” still in custody because Aegis holds the keys of their hot wallets, which are outside the cold-wallet controls assessed in factor 09.
- There is no onchain immobilisation. The contracts impose no restriction on transfers, so immobilisation depends entirely on custody procedures and on where the custodian sends tokens (E-02; factor 07).
- Book entry is not yet confirmed as the operative record. Q9.4 says the issuer “may” keep the register and that the offchain record “may” prevail depending on the issuance documents, which are not final, while Q9.5 states that the bond is in registered form.
- We could not verify that the custody address is held in the HSM cold wallet; onchain it is an ordinary externally owned account. On Ethereum, its only transaction (the 100-token transfer) came 60 seconds after it received 0.01 ETH of gas from the DACC.HK operator wallet (E-04).

**Outcome rationale.** Almost the whole issue sits at one custody address, and the Issuance Token has never moved. However, the arrangement permits transfers out of custody, and one has already been made, to a wallet whose controller has not been established. The token terms expressly allow self-custody, and book entry is not yet confirmed as the operative record. Immobilisation is partial, so the factor is Partially addressed.

**Recommendations**

| ID | Recommendation | Priority |
| --- | --- | --- |
| R08.1 | Confirm who controls `0x67D1e25180dbAdD501d008E0c288aB2eb6bAbea1` and, if it is not a custody address, either bring the position back into custody or document why the exception is acceptable, and confirm whether the 100-token transfer complies with the minimum subscription and eligibility terms. | High |
| R08.2 | Align the bond terms with an immobilised model (no self-custody), or state clearly that Part B immobilisation does not apply and explain the compensating controls. | High |
| R08.3 | Provide evidence that the custody address is held in the HSM cold wallet, for example a message signed from that address through the custodian’s approval flow. | Medium |

### Factor 09 — B(e) Custodian’s private key management

| Field | Detail |
| --- | --- |
| Outcome | Partially addressed |
| Questionnaire | Section 10 (questions 10.1–10.7) |
| Evidence IDs | E-03, E-04, E-06, E-07, E-11, E-12, E-13, E-14 |

**SFC expectation.** (e) the custodian’s internal policies and procedures, systems and controls including those governing private key management to ensure the secure storage of the private keys.

**What we assessed.** Whether key management for the wallets holding the bond tokens follows recognised practice (generation, storage, access, approval, backup, rotation) and whether it is independently attested. Areas: key generation and storage (HSM); approval quorum and access; backup, recovery and rotation; independent attestation (entity, scope, recency).

**Evidence reviewed**

| Evidence | Type | Basis |
| --- | --- | --- |
| E-03 | Questionnaire 9.2, 10.1–10.7 and the image in 10.7 | Client-stated |
| E-04 | Custody and operator addresses on both networks | Onchain, independently verified |
| E-06 | ScienceSoft penetration test, Aegis Vault portals (Dec 2022) | Document reviewed |
| E-07 | Aegis Encryption and Key Management Policy (Sep 2024) | Document reviewed |
| E-11 | Custody Wallet (Cold) Product Overview 2025 (14 pages) | Document reviewed |
| E-12, E-13 | SOC 2 Type 1 and Type 2, Aegis Trust Company (2022–23) | Document reviewed |
| E-14 | Custody agreement (redacted, unsigned template) | Document reviewed |

**Observations**

- The described design follows recognised practice. Keys are generated and held as non-extractable objects in a Thales Luna HSM (FIPS 140-3 Level 3), and signing runs in an air-gapped setup (Q10.2, Q10.3; E-11). DACC.HK states that the HSMs are in the custodian’s own server room in Hong Kong (Q10.2).
- Approvals: DACC.HK states that each withdrawal needs the client’s own review, then one Supervisor, two Auth Officers and one Auth Manager, who approve in an authenticator app protected by FIDO2 and a PIN. The administrators who approve wallet changes are kept separate from those who approve transactions (Q10.4; E-11).
- Backups: DACC.HK states that keys are copied to Thales Luna 8700 backup HSMs, one on site and one off site. The credentials needed to unlock them are split across five holders, and recovery is tested every six months (Q10.5).
- The custody platform is Aegis Vault, which the group’s custodians license from Aegis Custody Co., Ltd (Taiwan). The 2023 SOC 2 Type 2 report on Aegis Vault, as operated by the group’s US affiliate, has an unqualified opinion with no exceptions (E-11, E-13).

**Gaps and inconsistencies**

- No independent attestation covers the custodian that holds the bond tokens. Both SOC 2 reports are for Aegis Trust Company (South Dakota) and cover a period that ended on 6 Mar 2023. At that time Aegis Vault signed through Cybavo’s key-management service, part of which was hosted by Cybavo, not through the Thales HSM. The reports do not test Aegis Custody Company Limited’s own operations. Q10.6 states that they cover Aegis Custody Company Limited. No bridge letter or later report was provided (E-12, E-13).
- The key-management model is documented inconsistently. The Encryption and Key Management Policy describes multiparty-computation key shares stored on “secure servers or secure cloud services” and does not cover quorum, backup or compromise (E-07). The SOC 2 reports describe Cybavo’s service. Only the 2025 product overview and the questionnaire describe the Thales HSM.
- The product overview is a generic brochure, not a record of the custodian’s own setup. It lists a range of Thales models without saying which is used, labels the QR signing steps as performed “by the bank”, and names no supported networks or token standards (E-11).
- Q10.3 says signing requests cross the air gap “by QR code or USB file”. The product overview says data moves only by QR code, to avoid “any risk of hardware inserting into the HSM” (E-11).
- Key-ceremony records, backup and restore test records, access logs and HSM room controls were not provided. Several controls are written in the future tense (“will be present”, “will be recorded”, “will be card-controlled”) (Q10.2).
- The operator key that creates and attests tokens, `0x88A3BdF2C2263a76C73854D155280FfEb90fb5Ab`, is a single-signer MetaMask hot wallet outside the HSM. It is a different wallet from the custody address `0x736fD36d84e2dd541623E6Bd5Bb238D7067f2e33` that holds the tokens (E-04; factor 03).
- The custody agreement’s insurance (Canopius, a Lloyd’s syndicate) covers cold storage wallets only and states no limit. Assets in API-controlled hot wallets are not covered, yet Q9.2 and Q10.3 say tokens held for broker-dealers’ clients sit in hot wallets (E-14 §30).

**Outcome rationale.** The key-management design described for the bond’s cold wallet follows recognised practice. However, it is supported only by a product brochure and questionnaire answers. The independent attestation supplied covers the same platform as operated by a different entity, before the move to the HSM, for a period that ended in 2023, and the written key policy describes a different model. A control exists but is not independently evidenced for the entity and system in question, so the factor is Partially addressed.

**Recommendations**

| ID | Recommendation | Priority |
| --- | --- | --- |
| R09.1 | Provide the later Aegis Vault SOC 2 reports (the Type 2 report gives 7 April 2024 as the next issue date) or a bridge letter, and extend the attestation scope to Aegis Custody Company Limited and the HSM cold-wallet signing setup that holds the bond tokens. | High |
| R09.2 | Update the Encryption and Key Management Policy to the HSM model actually used (quorum, key ceremonies, backup, rotation, compromise, and how data crosses the air gap), and keep records of ceremonies and restore tests. | High |
| R09.3 | Bring the operator and attestation key under the same HSM and approval controls (see factor 03). | High |
| R09.4 | Confirm the insurance limit and whether any bond tokens are, or will be, held in hot wallets outside the cold-storage insurance. | Medium |

### Factor 10 — B(f) Segregation of client assets

| Field | Detail |
| --- | --- |
| Outcome | Partially addressed |
| Questionnaire | Section 11 (questions 11.1–11.4) |
| Evidence IDs | E-03, E-04, E-05, E-09, E-14, E-15 |

**SFC expectation.** (f) whether the custodian has appropriate segregation arrangements in place such that the Tokenised Securities are, throughout the custody chain, segregated from: (i) the assets of the custodian/sub-custodian; and (ii) the assets of other customers of the custodian (unless the Tokenised Securities are held in an omnibus client account).

**What we assessed.** Whether the onchain wallet structure, the custody books and the legal arrangements keep client tokens separate from the custodian’s own assets and attributable to each client, at every layer of the custody chain. Areas: separation from the custodian’s own assets; separation between clients and treatment of omnibus wallets; segregation at each custody layer (cold wallet, hot wallet, broker-dealer); the legal basis (custody agreement or trust).

**Evidence reviewed**

| Evidence | Type | Basis |
| --- | --- | --- |
| E-03 | Questionnaire 9.1, 11.1–11.4 and the workflow image in 11.2 | Client-stated |
| E-04 | Holder addresses on both networks | Onchain, independently verified |
| E-05 | Issuer named in the token metadata | Onchain/IPFS, independently verified |
| E-09 | Aegis Custody Company Limited AML & CTF Policy (deposit pooling) | Document reviewed |
| E-14 | Custody agreement (redacted, unsigned template) | Document reviewed |
| E-15 | Sample custody statement (marked SAMPLE) | Document reviewed |

**Observations**

- The standard custody agreement says that title stays with the client, that the custodian has no independent right to pledge client assets, and that each client gets unique blockchain addresses segregated from all other assets held by the custodian (E-14 §5.4, §5.5, §6).
- DACC.HK states that wallets holding client bond tokens are never commingled with DACC.HK’s own assets or with assets of clients outside these bonds (Q11.1).
- Onchain, the whole issue sits at a dedicated custody address that holds no other DACC.HK token (E-04).

**Gaps and inconsistencies**

- Who the tokens belong to is not shown. No ledger extract was provided showing whose client account the custody address belongs to: the issuer, a broker-dealer or investors. Q11.2 asked for an anonymised extract and offered a sample statement instead. DACC.HK is both the issuer named in the token metadata and the parent of the custodian, so the separation between the issuer’s own position and client positions matters here (E-05; Q9.1, Q11.2).
- Per-client addresses and omnibus wallets are both described. The agreement promises unique per-client addresses, but the workflow image in Q11.2 shows per-client deposit addresses swept into omnibus deposit and withdrawal wallets, and the AML policy describes the same pooling (E-03 Q11.2; E-09). The SFC factor allows an omnibus client account, but how omnibus balances are attributed to individual clients is not evidenced.
- The sample statement (E-15) is marked SAMPLE and is illustrative only. It shows a DACC29 deposit of 100 tokens on 24 Oct 2026, but its holding and sender addresses are placeholders that fail the EIP-55 checksum, and it names no legal entity.
- The custody agreement is an unsigned template. Its insolvency clause 5.2 is an incomplete sentence (“that Custodian becomes the subject of a proceeding under the Cap. 6 Bankruptcy Ordinance.”), it describes the custodian as “an excluded fiduciary under SDCL 55-1B”, a South Dakota statute (§15.11), and it gives the custodian a right of set-off against client assets (§15.8). It does not mention tokenised securities (E-14).
- It is a custody arrangement, not a trust (Q11.3). How client assets would be protected if the custodian became insolvent was not evidenced.

**Outcome rationale.** The custody agreement template provides for segregation, and the issue is held at a dedicated address. However, how tokens are attributed to clients is not evidenced, the custody model for broker-dealer clients uses omnibus wallets with no ledger evidence, and the agreement supplied is a template with defects that affect protection on insolvency. The factor is Partially addressed.

**Recommendations**

| ID | Recommendation | Priority |
| --- | --- | --- |
| R10.1 | Provide an anonymised custody ledger extract showing which client account(s) hold the DACC29 position(s) on each network, and how omnibus balances are attributed to individual clients. | High |
| R10.2 | Fix the custody agreement template (the incomplete insolvency clause 5.2, the South Dakota statutory reference and the set-off right) and add a tokenised-securities addendum. | Medium |
| R10.3 | Obtain a legal view on how client tokens are treated if the custodian becomes insolvent, given that the arrangement is not a trust. | Medium |

## Consolidated recommendations

The table lists every recommendation from the factor assessments, ordered by priority and then by factor: 16 High, 16 Medium and 3 Low. Each factor assessment gives the full wording and the finding it addresses.

| ID | Recommendation | Factor | Priority |
| --- | --- | --- | --- |
| R02.1 | Add a confirmation threshold and reorg detection/rollback to the Listener for each network. | 02 | High |
| R02.2 | Define how one bond position is represented across Ethereum and Conflux eSpace, with a regular cross-chain supply and holder reconciliation. | 02 | High |
| R03.1 | Move the deployer, operator and attestation role to a multi-signature or HSM-backed key under the custodian’s approval controls. | 03 | High |
| R03.2 | Provide the seven token-operation policies listed in Q4.7, and update the group policies for DLT operations. | 03 | High |
| R04.1 | Finalise the bond documents and align the onchain metadata with them. | 04 | High |
| R05.1 | Write a DLT continuity runbook with owners, decision rights and investor notification steps. | 05 | High |
| R05.2 | Document and approve a migration and redeployment procedure that the bond documents permit. | 05 | High |
| R06.1 | Move KYC submissions into an access-controlled, encrypted store, and encrypt documents marked “encrypted”. | 06 | High |
| R07.1 | Fix in the bond documents which party performs investor KYC and eligibility checks before any token reaches a holder. | 07 | High |
| R07.2 | Remove or tightly control the blocklist override, and provide a DACC.HK-specific STR procedure with a named Compliance Officer and MLRO. | 07 | High |
| R08.1 | Confirm who controls the out-of-custody holder address, bring the position back into custody or document the exception, and confirm the transfer met the minimum subscription and eligibility terms. | 08 | High |
| R08.2 | Align the bond terms with an immobilised model, or state that Part B immobilisation does not apply and explain the compensating controls. | 08 | High |
| R09.1 | Provide the later Aegis Vault SOC 2 reports or a bridge letter, and extend the attestation to Aegis Custody Company Limited’s HSM cold wallet. | 09 | High |
| R09.2 | Update the key management policy to the HSM model actually used, and keep records of key ceremonies and restore tests. | 09 | High |
| R09.3 | Bring the operator and attestation key under the same HSM and approval controls. | 09 | High |
| R10.1 | Provide an anonymised custody ledger extract showing which clients the DACC29 positions belong to. | 10 | High |
| R01.2 | Correct the DAC-02 acknowledgement with CertiK, and ask CertiK to complete its post-deployment review. | 01 | Medium |
| R02.3 | Use paid RPC plans or own nodes, with a tested second provider per network. | 02 | Medium |
| R02.4 | Commission a penetration test of the DigiQuick portals, API and Listener. | 02 | Medium |
| R03.3 | Implement the planned monitoring, alerting and response targets. | 03 | Medium |
| R03.4 | Commission a penetration test of the DigiQuick platform, and refresh the Aegis assurance so it covers the entity holding the bond tokens. | 03 | Medium |
| R04.2 | Provide a dated, signed-off reconciliation report and register extract covering both networks. | 04 | Medium |
| R04.3 | Document how the two network representations relate to the single register. | 04 | Medium |
| R05.3 | Run and record a tabletop exercise on at least one DLT scenario. | 05 | Medium |
| R05.4 | Add a cross-region, immutable backup copy, and set retention in line with record-keeping obligations. | 05 | Medium |
| R06.2 | Address the PDPO in the privacy and data policies, and put data-processing agreements in place. | 06 | Medium |
| R07.3 | Consider onchain transfer restrictions for tokens that may leave custody. | 07 | Medium |
| R07.4 | Update the AML policy (Hong Kong references, Travel Rule, audit cadence, tools actually used). | 07 | Medium |
| R08.3 | Provide evidence that the custody address is held in the HSM cold wallet. | 08 | Medium |
| R09.4 | Confirm the insurance limit and whether any bond tokens are held in hot wallets outside the cold-storage insurance. | 09 | Medium |
| R10.2 | Fix the custody agreement template and add a tokenised-securities addendum. | 10 | Medium |
| R10.3 | Obtain a legal view on how client tokens are treated if the custodian becomes insolvent. | 10 | Medium |
| R01.1 | Ask CertiK to name the final fix commit (`545d47a`) in its report, and publish the verified source links. | 01 | Low |
| R01.3 | Publish the disclosure promised for DAC-13. | 01 | Low |
| R06.3 | Explain clearly to investors what becomes public onchain. | 06 | Low |

## Appendix A: Evidence index

Evidence is cited by reference, and confidential documents are not reproduced in this report. The evidence folders for factors 02, 04, 06 and 08 were empty.

| ID | Document | Supplied by | Date | Q ref | Factor(s) |
| --- | --- | --- | --- | --- | --- |
| E-01 | CertiK, “Digital Asset Clearing Services — audit”, final report (42 pp.) | DACC.HK | 6 Aug 2026 | 2.1 | 01 |
| E-02 | DigiQuick contracts source at commit `545d47a` (CertiK’s final fix commit) | DACC.HK | 24 Oct 2026 | 1.3, 2.1 | 01, 03, 07, 08 |
| E-03 | Completed questionnaire: 58 questions in 11 sections, including 7 embedded images (Appendix C) | DACC.HK | 26 Oct 2026 | All | All |
| E-04 | Onchain verification by OpenZeppelin on Ethereum and Conflux eSpace (Appendix B) | On-chain | 29 Oct 2026 | 1.3, 3.3, 9.1, 9.6 | All except 05 |
| E-05 | DACC29 token metadata on IPFS. Ethereum `bafkreicdwynlq6tyectcakgal6xpheibp2oqltasyf4r5sb2rdtt2uladi`. Conflux eSpace `bafkreibbgjzwnlkornfqbm64zs26w7mtpqnhjbmiqtuxlowupmssuyycua` | Public (IPFS) | 24 Oct 2026 | 3.3, 7.1 | 02–08, 10 |
| E-06 | ScienceSoft, Penetration Testing Executive Report, Aegis Vault (prepared for Aegis Custody Co., Ltd) | DACC.HK | 31 Jan 2023 | 2.3, 4.9 | 03, 09 |
| E-07 | Aegis Custody policies (9): Information Security; Access Control and Termination; Change Management; Encryption and Key Management; BC/DR Plan; Security Incident Response Plan; Data Classification; Data Retention and Disposal; Vendor Management | DACC.HK | Last reviewed 30 Sep 2024 | 4.7, 6.1, 7.3 | 03, 05, 06, 09 |
| E-08 | Tabletop exercise records: BC/DR (power outage) and security incident response (unstable patch), Aegis Custody Co., Ltd | DACC.HK | 10–11 Jan 2023 | 6.3 | 05 |
| E-09 | Aegis Custody Company Limited, AML & CTF Policy (19 pp.) | DACC.HK | Last edited 30 Sep 2024 | 8.1 | 07, 10 |
| E-10 | “DACC.HK — STR Filing Process” (Aegis and Beosin slide deck, 15 pp.) | DACC.HK | Undated | 8.1–8.4 | 07 |
| E-11 | Aegis Custody, Custody Wallet (Cold) Product Overview (slide deck, 14 pp.) | DACC.HK | 2025 | 10.2, 10.5 | 09 |
| E-12 | SOC 2 Type 1 report, Aegis Trust Company (Prescient Assurance) | DACC.HK | As of 17 Jan 2023; report dated 2 Feb 2023 | 10.6 | 03, 09 |
| E-13 | SOC 2 Type 2 report, Aegis Trust Company (Prescient Assurance) | DACC.HK | 6 Dec 2022 – 6 Mar 2023 | 10.6 | 03, 09 |
| E-14 | Aegis Custody Company Limited, Custody Agreement (redacted, unsigned template) | DACC.HK | Undated | 5.5, 11.3 | 04, 09, 10 |
| E-15 | Sample custody statement (marked SAMPLE, illustrative) | DACC.HK | Oct 2026 | 11.2 | 10 |
| E-16 | Aegis Tokenisation Product deck (2 pp.) and RWA Tokenisation 2025 deck (pp. 3–9) | DACC.HK | 2024–2025 | — | 02 (context) |

## Appendix B: Onchain verification log

Checks were performed on 30 Oct 2026 by direct RPC queries to Ethereum mainnet (chain ID 1) and Conflux eSpace mainnet (chain ID 1030), and selected checks were repeated on 2 Nov 2026. Block numbers are given as Ethereum / Conflux eSpace. Transaction hashes are as listed in questionnaire answers 1.3 and 3.3.

| # | Network | Check | Result |
| --- | --- | --- | --- |
| B1–B2 | Both | TokenFactory deployments (blocks 26,259,693 / 159,805,865) | Confirmed: deployed by the operator wallet at nonce 0, successful, same address on both networks |
| B3–B4 | Both | DACC29 creation (blocks 26,261,738 / 159,827,675) | Confirmed: `createTokenWithRecording`; 3,000,000 supply and recording supply; receiver is the custody address; operator wallet set as setting and recording operator; ERC-20 adapter and companion NFT enabled |
| B5–B6 | Both | Adapter naming (blocks 26,261,742 / 159,827,726) | Confirmed: “DACC.HK 5.00% Tokenised Bond due 2029” / DACC29 / 0 decimals |
| B7–B8 | Both | 100-token transfer (blocks 26,261,829 / 159,829,515) | Confirmed: the custody address’s only transaction. On Ethereum it came 60 seconds after the address received 0.01 ETH of gas from the operator wallet (block 26,261,824) |
| B9 | Both | Deployed code against source | Runtime bytecode identical to a local compilation of `545d47a`; only the metadata hash differs |
| B10 | Both | Roles and configuration | Trusted forwarder is the zero address; operators are the operator wallet; the factory has created only the template and one adapter |
| B11 | Both | Balances and supply | Supply 3,000,000; custody address 2,999,900 plus the Issuance Token; holder `0x67D1e25180dbAdD501d008E0c288aB2eb6bAbea1` holds 100; recording ledger 3,000,000 at the operator wallet |
| B12 | Both | Full event history | Only the creation and the one transfer; no approvals; no other tokens created |
| B13 | Both | Metadata and signatures | Metadata identical except for the network name and signatures; signatures recover to the operator wallet (labelled “Custodian (Aegis)”) and the issuer signing wallet |
| B14 | Both | Other listed addresses | Legacy factory exists on Ethereum only; the staging operator has two transactions on Ethereum mainnet and none on Conflux eSpace |
| B15 | Both | Re-check on 2 Nov 2026 (blocks 26,325,074 / 160,557,207) | Supply and balances unchanged; the custody and holder addresses are externally owned accounts with one and zero transactions respectively on each network |

## Appendix C: Questionnaire

OpenZeppelin issued the evidence questionnaire to DACC.HK on 19 Oct 2026, and DACC.HK returned its responses on 26 Oct 2026 (E-03). It has 58 questions in 11 sections. Section 1 collects background information, and sections 2 to 11 each cover one of the ten factors. References in this report such as “Q4.7” are to question numbers in the questionnaire.

Questionnaire with DACC.HK’s responses: DACC Tokenised Bond – SFC Technical Assessment Questionnaire.

Access to the questionnaire is restricted. Readers who need it should request access.

| Section | Topic | Questions | Factor |
| --- | --- | --- | --- |
| 1 | Background (applies to all factors) | 1.1–1.5 | All |
| 2 | Smart contract audit | 2.1–2.4 | 01 |
| 3 | Interoperability between DLT networks and back-end systems | 3.1–3.4 | 02 |
| 4 | Administrative controls, policies and procedures | 4.1–4.10 | 03 |
| 5 | Extrinsic rights and reconciliation with underlying records | 5.1–5.6 | 04 |
| 6 | Business continuity planning for DLT-related events | 6.1–6.4 | 05 |
| 7 | Data privacy | 7.1–7.4 | 06 |
| 8 | Money laundering and terrorist financing risks; KYC/AML gating | 8.1–8.4 | 07 |
| 9 | Immobilisation with central custody | 9.1–9.6 | 08 |
| 10 | Custodian’s private key management | 10.1–10.7 | 09 |
| 11 | Segregation of client assets | 11.1–11.4 | 10 |

## Appendix D: Glossary

| Term | Meaning |
| --- | --- |
| Onchain / offchain | Recorded on, or off, the blockchain |
| Air-gapped | Kept with no network connection, so data moves in and out only by hand (for example by QR code) |
| AMLO | Hong Kong Anti-Money Laundering and Counter-Terrorist Financing Ordinance (Cap. 615) |
| Cold wallet / hot wallet | A cold wallet’s keys are kept offline and sign only through a manual process; a hot wallet’s keys are online and can sign automatically |
| EIP-55 checksum | The mixed-case format of an Ethereum address that lets software detect mistyped addresses |
| EIP-1167 clone | A minimal proxy contract that forwards all calls to a fixed implementation |
| EOA | Externally owned account: a wallet controlled by a single private key, with no contract code |
| ERC-20 / ERC-721 / ERC-1155 | Ethereum token standards for fungible tokens, unique tokens (NFTs) and multi-token contracts |
| HSM | Hardware security module: tamper-resistant hardware that generates and uses keys without exposing them |
| IPFS | InterPlanetary File System: content-addressed storage in which a file’s identifier changes if its content changes |
| JFIU | Joint Financial Intelligence Unit, which receives suspicious transaction reports in Hong Kong |
| KYC | Know your customer: verifying a client’s identity before providing services |
| KYT | Know your transaction: screening of blockchain addresses and transactions |
| MLRO | Money Laundering Reporting Officer |
| Omnibus wallet | A wallet holding assets for several clients, with each client’s share recorded in the custodian’s books |
| PDPO | Personal Data (Privacy) Ordinance (Cap. 486) |
| PEP | Politically exposed person: a person with a prominent public function, who needs enhanced due diligence |
| Recording ledger | A parallel internal ledger in the DigiQuick contract, kept by the recording operator for bookkeeping; it does not move tokens |
| Reorganisation (reorg) | Replacement of recent blocks by a competing chain, which can reverse recently observed transactions |
| RPC | Remote procedure call endpoint used to read from and submit to a blockchain |
| RTO / RPO | Recovery time objective (how quickly a service must be restored) and recovery point objective (how much recent data may be lost) |
| SIEM | Security information and event management: a system that collects security logs, correlates them and raises alerts |
| SOC 2 | An independent auditor’s report on a service organisation’s controls; Type 1 covers design at a point in time, Type 2 covers operation over a period |
| STR | Suspicious transaction report |
| TCSP | Trust or company service provider (Hong Kong licence) |
| Travel Rule | The requirement to send originator and beneficiary information with virtual asset transfers between institutions |
