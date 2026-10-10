---
id: CR-2026-0005
title: 接受 60F 二十項 FortiGuard 授權過期的風險（Day 25）
status: done
type: standard
risk: high
requester: ivan
implementer: ivan
approver: ivan
approval_exception: 單人實驗場域，沒有第二位核准者。這是 Day 26 之前做的決定，2026-10-10 依當時紀錄補單
assets:
  - fgt-edge/FortiGuard licences
window_start: 2026-10-09T19:59:00+08:00
window_end: 2026-10-09T20:01:00+08:00
silence: 
tags: fortigate, licence, risk-acceptance, backfill
---

## 目的

60F 的 20 項 FortiGuard 服務在 2025-09-20 同日過期，包括韌體更新。這台暫時無法續約，決定接受風險，並讓告警不再每天提醒同一件事，但有到期日。

## 影響

- 無設定變更。60F 維持無 FortiGuard 簽章與分類的狀態（Day 24 的 error-allow 是這個狀態的後果）。
- `AssetLicenceExpired`（asset=fgt-edge）在 silence 期間不送出，Alertmanager 介面上看得到 suppressed。

## 步驟

1. onprem-metrics `eol/assets.tsv` 的授權列寫明接受風險（37af72b），部署到收集端。
2. Alertmanager silence `alertname=AssetLicenceExpired, asset=fgt-edge`，到 2027-01-07 00:00 +08:00。

## 驗證

- `amtool silence query` 看得到這個 silence，comment 寫明原因。
- 告警狀態為 suppressed。

## 回退

`amtool silence expire 840fecc9-66e0-458f-80ca-ed6d558831bf`；TSV 以前一版覆回。風險本身的處理是續約或換機，不在這張單的範圍。

## 紀錄

- begin: 2026-10-09T19:59:00+08:00
- end: 2026-10-09T20:01:00+08:00（silence 寫入資料卷）
- result: 完成。到期日 2027-01-07 是檢討日；若 FortiOS 出現影響 7.6.7 的 critical PSIRT，提前檢討
- evidence: silence 840fecc9；onprem-metrics 37af72b；Day 25 文章第四節
- followups: 2027-01-07 檢討
