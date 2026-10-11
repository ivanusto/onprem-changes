---
id: CR-2026-0007
title: 60F 的事件日誌送到收集端 515，收集端加第二個監聽器與稽核報告、規則（Day 27）
status: done
type: standard
risk: low
requester: ivan
implementer: ivan
approver: ivan
approval_exception: 單人實驗場域，沒有第二位核准者。以 PR 自審取代，PR 開啟與合併時間在 git log。冷卻未滿 24 小時，因為 Day 27 當天要發表，由擁有者以管理員略過合併
assets:
  - fgt-edge/FortiOS
  - collector/Ubuntu
window_start: 2026-10-11T16:30:00+08:00
window_end: 2026-10-11T18:30:00+08:00
silence: alertname=TargetDown job=victorialogs
tags: fortigate, syslog, victorialogs, audit
---

## 目的

Day 22 的 syslogd2 只送 traffic，event 被刻意排除；Day 22 與 Day 26 都留了一句，設定變更與管理員登入要進 WORM，留給 Day 27。管線裡其他五種管理動作（PVE 任務、sudo、SSH 登入、NAS 事件、change 標記）早就天天進 WORM，缺的只有邊界防火牆。這張單把 60F 的事件日誌接進來，並讓 onprem-logs 的 `audit/audit-report.py` 每 10 分鐘把管理動作對上變更單的窗，沒有單子的 FortiGate 設定變更成為 warning。

## 影響

- 60F：新增第三組 syslog 目的地 syslogd3，reliable（TCP 515）、rfc5424，只送 event，排除 webfilter 與 wireless 子類型。syslogd2 不動。設定改動不中斷轉送。60F 自己的事件日誌在記憶體環狀保留約 8 天，一天約 3,900 筆，其中約 86% 是 wireless，排除後預期一天數百筆。
- 收集端：VictoriaLogs 的 compose 改了 command 與 ports，要重建容器，停約 10 秒。期間節點的 journal-upload 與 60F 的 reliable 模式會重送，NAS 的 Log Sender（TCP 514）要確認有重新連線。`TargetDown` 的 for 是 5 分鐘，正常不會響，仍在窗內靜音。
- 收集端：DOCKER-USER 白名單加 515 只給 60F；cron 加 audit-report 兩行；Prometheus 加 `rules/audit.yml`；Alertmanager 加一條 inhibit，因為是單檔掛載，改完要重啟容器（Day 26 學到的）。

## 步驟

時間為 +08:00。

在收集端上：

```sh
sudo -u metrics /opt/onprem-changes/scripts/change-mark.sh begin CR-2026-0007
```

在工作站上，60F 改動前備份：

```sh
curl -sk -X POST -H "Authorization: Bearer $FGT_TOKEN" \
  "https://<fgt-edge>:10443/api/v2/monitor/system/config/backup?scope=global&destination=file" -o fortigate-backup-$(date +%Y%m%d-%H%M).conf
head -3 fortigate-backup-*.conf           # #config-version=FGT60F-...
```

在收集端上，記下重建前每個來源的最後時間，再部署 onprem-logs 新版（舊檔留 `*.bak-day27`）：

```sh
curl -s http://127.0.0.1:9428/select/logsql/query -d 'query=_time:10m | stats by (hostname, _HOSTNAME) max(_time) as last'
sudoedit /etc/default/onprem-logs-allowlist         # AUDIT_FROM="<fgt-edge>"
sudo systemctl restart onprem-logs-allowlist
sudo iptables -S ONPREM-LOGS | grep 515
cd /opt/onprem-logs && sudo docker compose up -d   # 第二個監聽器與 515 的 port，容器重建
```

在 60F 上（REST，等同 CLI 的 `config log syslogd3 setting` 與 `filter`，實際接受的寫法記在 onprem-logs 的 `audit/fortigate-syslogd3.md`）：

1. filter：severity information，forward、local、multicast、sniffer、ztna traffic 與 http-transaction、anomaly、voip、forti-switch 全部 disable；free-style 1 排除 webfilter，free-style 2 排除 event 的 wireless 子類型。
2. setting：status enable、server 收集端、mode reliable、port 515、format rfc5424。

在收集端上：

```sh
sudo cp onprem-metrics/prometheus/rules/audit.yml /opt/onprem-metrics/prometheus/rules/
curl -s -X POST http://127.0.0.1:9090/-/reload
sudoedit /opt/onprem-metrics/alertmanager/alertmanager.yml   # AuditReportStale 的 inhibit
sudo docker restart onprem-metrics-alertmanager-1
sed -n '/Day 27/,$p' /opt/onprem-logs/audit/collector.cron | sudo tee -a /etc/cron.d/onprem-logs
sudo -u metrics /opt/onprem-changes/scripts/change-mark.sh end CR-2026-0007
```

## 驗證

- 重建後 5 分鐘內，重建前列出的每個 `hostname` 與 `_HOSTNAME` 都有新行（514 的兩台 NAS 與 60F traffic、9428 的節點與收集端）。
- `_time:1h audit:=fortigate | unpack_logfmt from _msg fields (type, subtype) | stats by (type, subtype) count()` 只有 `event`，沒有 `wireless`。
- 第一批事件是 syslogd3 本身的設定變更，`audit-report.py` 把它們歸到 CR-2026-0007。
- `audit.prom` 有四個指標族；Prometheus 的 `audit` 規則群組載入；Alertmanager 的 `/api/v2/status` 設定原文含新 inhibit。

## 回退

- 60F：`config log syslogd3 setting` 改回 `set status disable`，filter 改回全部 enable、刪 free-style。以窗前的備份比對。
- 收集端：/opt/onprem-logs 以 `*.bak-day27` 覆回，`docker compose up -d` 重建回單一監聽器；白名單覆回並重啟服務；刪 cron 的 Day 27 兩行、`rules/audit.yml` 後 reload；alertmanager.yml 以 `.bak-day27` 覆回並重啟容器。

## 紀錄

- begin: 2026-10-11T16:30:15+08:00
- end: 2026-10-11T16:39:17+08:00
- result: 完成。窗原訂 14:00 到 16:00，PR #2 在 15:51 才合併，以 PR #3 移到 16:30 到 18:30 後施作，窗前兩台設備都沒動
- evidence: 16:30 設定備份 `#config-version=FGT60F-7.6.7-FW-build3704`；16:31:09 容器重建，重建前 30 分鐘有資料的 6 個來源在 16:33:36 前都有新行（9428 四台、514 兩台）；60F 的 TCP 514 在 16:31:10 重連，兩台 NAS 走 UDP 514，不需重連；515 第一筆 16:34:21 logid `0100022905` Log device status has changed，下一筆是 syslogd3 setting 本身的 `0100044546`，報告歸到本單；16:39 為止 `audit:=fortigate` 只有 event/system，60F 記憶體日誌同時段的兩筆 wireless（16:35:43、16:37:43）沒有送出；`audit.prom` 四個指標族、Prometheus `audit` 群組四條 health ok、Alertmanager `/api/v2/status` 含新 inhibit，begin 的 silence 撐過容器重啟；封存演練（非 WORM 暫存目錄）的 MANIFEST 有 `audit-<devname>` 一列，sha256 OK
- followups: syslogd3 filter 的 `ztna-traffic` 以 REST 送 disable 仍讀回 enable（`forward-traffic` 第一次合併送出也沒生效，單獨再送才生效），這台沒有 ZTNA，以收到的內容為準，下次用 CLI 確認；場域 `audit/ignore.txt` 含金鑰指紋，tar 部署會覆蓋，副本在 `ignore.txt.site`；10/12 08:10 看真正的 MANIFEST 列、08:45 看第一份週報（週一）
