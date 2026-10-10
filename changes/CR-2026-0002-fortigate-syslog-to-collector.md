---
id: CR-2026-0002
title: 讓 60F 把流量日誌送到收集端（Day 22）
status: done
type: standard
risk: medium
requester: ivan
implementer: ivan
approver: ivan
approval_exception: 單人實驗場域，沒有第二位核准者。這是 Day 26 之前做的變更，2026-10-10 依當時紀錄補單，當時沒有事前核准
assets:
  - fgt-edge/FortiOS
  - collector/Ubuntu
window_start: 2026-10-06T11:32:00+08:00
window_end: 2026-10-06T13:20:00+08:00
silence: 
tags: fortigate, syslog, victorialogs, backfill
---

## 目的

Day 21 的管線收得到節點與 NAS 的日誌，收不到邊界防火牆的。60F 的第二組 syslog 目的地（syslogd2）原本是停用的，打開並指到收集端，Day 22 的防火牆週報與六條規則才有資料。

## 影響

- 60F：syslogd2 以 reliable（TCP 514）、rfc5424 送出 forward 與 local 流量，排除 NTP 與 event 類；policy 4（攝影機的 deny）開始記錄。設定改動不中斷轉送。
- 收集端：DOCKER-USER 白名單加 60F，收集端自己的 journal 開始上傳，加 fw-report 的 cron、firewall.yml 規則與 FwReportStale 的 inhibit。
- 日誌量：每小時約 1,000 筆（Day 22 量測）。

## 步驟

照當時順序，時間為 +08:00。

1. 11:32 60F 改動前備份（`POST /api/v2/monitor/system/config/backup`）。
2. 11:37 收集端白名單加 60F，DOCKER-USER 每條 DROP 前加限速 LOG，重啟 `onprem-logs-allowlist`。
3. 11:37 60F `config log syslogd2 filter`：severity information，forward-traffic、local-traffic enable，其餘 disable。
4. 11:37:55 60F `config log syslogd2 setting`：status enable、mode reliable、port 514、format rfc5424、facility local7。
5. 11:39 free-style 2 排除 `(type event)`。
6. 11:42 policy 4 `logtraffic` disable 改 all。
7. 11:43 收集端裝 systemd-journal-upload（journal 進 VictoriaLogs）。
8. 11:51 收集端 fw-report、cron、firewall.yml、inhibit，Prometheus reload、Alertmanager SIGHUP。
9. 13:07:12 free-style 1 改成排除 `(service NTP)`（每個 category 只能一條，第三條被拒，錯誤 -15）。
10. 13:10 封存腳本加 `ARCHIVE_SKIP`；13:20 FortiGate 突發門檻 4000 改 1200。

## 驗證

- VictoriaLogs 的 60F 串流第一筆在 2026-10-06 11:37:57，第一筆 policy 4 的 deny 在 11:42:05。
- `fw.prom` 寫入，Day 22 的六條規則在 Prometheus 載入。

## 回退

- 60F：`config log syslogd2 setting` 改回 `set status disable`，filter 改回全部 enable、刪 free-style；policy 4 `set logtraffic disable`。以 11:32 的備份比對。
- 收集端：白名單與腳本以 `.bak-day22` 覆回並重啟服務；`systemctl disable --now systemd-journal-upload`；cron 以備份覆回，刪 firewall.yml 後 reload。

## 紀錄

- begin: 2026-10-06T11:32:00+08:00（60F 備份）
- end: 2026-10-06T13:20:00+08:00（門檻調整後 reload）
- result: 成功。當時沒有單子、沒有 change-mark，開始與結束取自 Day 22 的變更表與 60F 備份時間
- evidence: 60F 備份 diff；VictoriaLogs 的 60F 串流第一筆 11:37:57；Day 22 文章與 onprem-logs v0.2.0
- followups: 無
