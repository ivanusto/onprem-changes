---
id: CR-2026-0008
title: 收集端部署 onprem-logs v0.5.1，稽核告警排除管理工作站，場域忽略清單的 HDP 規則改成只認金鑰
status: scheduled
type: standard
risk: low
requester: ivan
implementer: ivan
approver: ivan
approval_exception: 單人實驗場域，沒有第二位核准者。以 PR 自審取代，PR 開啟與合併時間在 git log。冷卻未滿 24 小時，由擁有者以管理員略過合併
assets:
  - collector/Ubuntu
window_start: 2026-10-12T17:00:00+08:00
window_end: 2026-10-12T18:00:00+08:00
silence: alertname=AuditUnattributedActions
tags: audit, victorialogs, day27
---

## 目的

`AuditUnattributedActions`（info）自 CR-2026-0007 上線起，10/11 16:40 到 10/12 13:20 的 124 個十分鐘取樣有 115 個非零，等於一直在響。原因有兩個：

1. 10/11 19:12 起，NAS 的出向位址從 .2 換成 .30（NAS 有兩個 LAN 位址），場域忽略清單的 HDP inventory 規則寫了來源位址，HDP 每小時 36 次的 root 登入從那時起全部掉出忽略清單。HDP 的身分是金鑰，不是位址，規則改成只認金鑰指紋。以真實資料模擬，21 小時裡有沒單子動作的小時從幾乎全部降到 10 個。
2. 施工的日子裡，管理工作站（131 與 .10）的 SSH 登入與 sudo 讓告警一直亮著。onprem-logs v0.5.1 的 `audit-report.py` 新增 `AUDIT_QUIET_FROM`：從這些位址的 SSH 登入、在那些登入 session 裡的 sudo（以 systemd-logind 的 session 編號串接）、從這些位址的 FortiGate GUI 登入，仍列在報告上並標「工作站」，但不計入 `audit_unattributed`，改記在 `audit_unattributed_quiet`。FortiGate 設定變更、NAS 事件、PVE 任務不排除。

同時把收集端的 audit-report 換成 v0.5.x，補上 v0.5.0 的 `--since` 報告標頭修正。

## 影響

- 收集端只換檔案與 cron，不重建容器（v0.5.1 與部署中的 `docker-compose.yml` 相同，部署前 diff 確認）。
- 窗內 `AuditUnattributedActions` 的數字會變，已靜音。
- 報告的語意變了：`audit_unattributed` 不含工作站的動作。工作站的動作仍在 Markdown 週報裡。

## 步驟

時間為 +08:00，全部在收集端上。

```sh
sudo -u metrics /opt/onprem-changes/scripts/change-mark.sh begin CR-2026-0008
diff /opt/onprem-logs/docker-compose.yml <(tar xOf onprem-logs-v0.5.1.tar docker-compose.yml)   # 預期無差異
# tar 部署 v0.5.1，被覆蓋的檔留 *.bak-cr0008
sudo cp -p /opt/onprem-logs/audit/ignore.txt.site /opt/onprem-logs/audit/ignore.txt   # tar 會蓋掉場域清單
sudoedit /opt/onprem-logs/audit/ignore.txt          # HDP 那條：from 192\.168\.2\.2 改成 from \S+
sudo cp -p /opt/onprem-logs/audit/ignore.txt /opt/onprem-logs/audit/ignore.txt.site
sudo cp -p /etc/cron.d/onprem-logs /etc/cron.d/onprem-logs.bak-cr0008
sudoedit /etc/cron.d/onprem-logs                    # audit-report 那一行加 AUDIT_QUIET_FROM="192.168.2.131 192.168.2.10"
sudo -u metrics env VL=http://127.0.0.1:9428 CHANGES_DIR=/opt/onprem-changes/changes AUDIT_QUIET_FROM="192.168.2.131 192.168.2.10" \
  /opt/onprem-logs/audit/audit-report.py --window 1h --textfile /var/lib/node_exporter/textfile/audit.prom --out /dev/null
sudo -u metrics /opt/onprem-changes/scripts/change-mark.sh end CR-2026-0008
```

## 驗證

- `audit.prom` 有 `audit_unattributed` 與 `audit_unattributed_quiet` 兩個指標族，Prometheus 讀得到。
- 窗內的 SSH 登入與 sudo 歸到本單；窗外工作站的動作出現在 `audit_unattributed_quiet`。
- 之後的 1h 報告裡沒有 HDP 金鑰的登入，不論來源是 .2 或 .30。
- 隔天以 `sum(audit_unattributed) > 0` 的非零取樣比例對照 CR-2026-0007 之後的 115/124。

## 回退

- `/opt/onprem-logs` 以 `*.bak-cr0008` 覆回，`/etc/cron.d/onprem-logs` 以 `.bak-cr0008` 覆回。忽略清單的舊版在 `ignore.txt.bak-cr0008`。

## 紀錄

執行時填。

- begin: 
- end: 
- result: 
- evidence: 
- followups: 
