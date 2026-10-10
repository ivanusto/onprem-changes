---
id: CR-2026-0004
title: 60F 的 policy 1 掛上 aup-monitor 網頁過濾（Day 24）
status: done
type: standard
risk: high
requester: ivan
implementer: ivan
approver: ivan
approval_exception: 單人實驗場域，沒有第二位核准者。這是 Day 26 之前做的變更，2026-10-10 依當時紀錄補單，當時沒有事前核准
assets:
  - fgt-edge/FortiOS
  - collector/Ubuntu
window_start: 2026-10-08T00:01:00+08:00
window_end: 2026-10-08T01:25:00+08:00
silence: 
tags: fortigate, webfilter, aup, backfill
---

## 目的

Day 24 的 AUP 要有執行面：知道內網連了哪些類別，擋下三個明確違反 AUP 的類別。

## 影響

- policy 1（內網出口）啟用 UTM，掛 `aup-monitor`（flow）。60F 的 FortiGuard 授權已過期，每筆分類都是 `ftgd_err`，profile 必須 `error-allow`，否則整個網段的網頁都會被擋。這是 risk 標 high 的原因。
- `certificate-inspection` 一開 UTM 就生效，預設擋過期與撤銷的憑證。
- 日誌量約 Day 22 的九倍（每小時約 1 萬筆）。收集端加 aup-report 的 cron、aup.yml 四條規則與 AupReportStale 的 inhibit。

## 步驟

1. 00:01 60F 改動前備份。
2. 00:04 新增 urlfilter 1 `aup-test` 與 profile `aup-monitor`（93 類 monitor、26／61／86 block、error-allow）。
3. 00:07 到 00:09 收集端裝 aup-report、cron、aup.yml，Prometheus reload、Alertmanager SIGHUP。
4. 00:14:40 policy 1：utm-status enable、webfilter-profile aup-monitor、logtraffic all 改 utm。
5. 01:17 從工作站兩次觸發測試網域，確認擋下有紀錄。

## 驗證

- VictoriaLogs 的 60F 串流第一筆 webfilter 在 2026-10-08 00:14:46（rfc5424 標頭只到整秒，eventtime 與它差約一秒）。
- aup.prom 寫入，四條規則載入。

## 回退

依序：先 policy 1（`PUT {"utm-status":"disable","webfilter-profile":"","logtraffic":"utm"}`），再刪 profile `aup-monitor`，再刪 urlfilter 1。收集端 cron 以 `.bak-day24` 覆回，刪 aup.yml 後 reload。

## 紀錄

- begin: 2026-10-08T00:01:00+08:00（60F 備份）
- end: 2026-10-08T01:25:00+08:00（暫時儀表板刪除）
- result: 成功。當時沒有單子，時間取自 Day 24 的變更表
- evidence: 60F 備份 diff；VictoriaLogs 第一筆 webfilter 00:14:46；onprem-logs v0.4.0
- followups: 週報後調 AupBlockedBurst 門檻
