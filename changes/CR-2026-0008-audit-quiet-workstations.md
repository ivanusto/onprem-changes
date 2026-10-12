---
id: CR-2026-0008
title: 收集端部署 onprem-logs v0.5.1，稽核告警排除管理工作站，場域忽略清單的 HDP 規則改成只認金鑰
status: done
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
# tar 部署 v0.5.1（tag 9709e6a），被覆蓋的檔留 *.bak-cr0008
sudo cp -p /opt/onprem-logs/audit/ignore.txt.site /opt/onprem-logs/audit/ignore.txt   # tar 會蓋掉場域清單
sudoedit /opt/onprem-logs/audit/ignore.txt          # HDP 那條：from 192\.168\.2\.2 改成 from \S+
sudo cp -p /opt/onprem-logs/audit/ignore.txt /opt/onprem-logs/audit/ignore.txt.site
sudo cp -p /etc/cron.d/onprem-logs /etc/cron.d/onprem-logs.bak-cr0008
sudoedit /etc/cron.d/onprem-logs                    # Day 27 兩行之前加一行 AUDIT_QUIET_FROM="192.168.2.131 192.168.2.10"（同 audit/collector.cron）
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

- begin: 2026-10-12 17:00:34 +08:00（change-mark begin，Grafana 註記 4，silence 49f009e1）
- end: 2026-10-12 17:03:28 +08:00（change-mark end；註記 4 與 silence 49f009e1 手動關閉，原因見下）
- result: 完成。`docker-compose.yml` 與 v0.5.1 無差異，未重建容器。tar 部署改到 7 個檔（retention.md、audit 下 5 個、tests/smoke-audit.sh），舊檔留 `*.bak-cr0008`。`ignore.txt` 由 `.site` 蓋回，HDP 那條改成 `from \S+ key ...EF90d...` 並加註理由，`.site` 同步；`ignore.txt.repo` 換成 v0.5.1 的 repo 版。`/etc/cron.d/onprem-logs` 在 Day 27 兩行前加 `AUDIT_QUIET_FROM="192.168.2.131 192.168.2.10"` 與三行註解。
- evidence:
  - 17:01 以 metrics 身分手動跑 1h 報告：管理動作 80 筆，忽略清單排除 99 筆，有變更單 45 筆，沒有 35 筆，其中 34 筆來自工作站（ssh-login 23、sudo 11）。報告裡沒有 HDP 金鑰的登入。
  - `audit.prom` 有 `audit_unattributed` 與 `audit_unattributed_quiet` 兩個指標族；Prometheus 的 `sum(audit_unattributed)` 從 71 降到 1，`audit_unattributed_quiet` 讀得到（ssh-login 23、sudo 11）。
  - 剩下的 1 筆是 16:59:03 spark01 上經 tailnet（.30）的密碼登入，CR-2026-0009 關閉密碼登入後不會再出現。
  - 窗內的 SSH 登入與 sudo 都歸到本單（報告「依變更單」51 筆）。
- 重疊執行：另一個工作階段在 17:00:49 到 17:01:44 也執行了本單（第二筆 begin、Grafana 註記 5、silence 300b0b11，17:01:44 的 end 關掉的是它自己的註記與 silence，並清掉 change-mark 的狀態檔，所以 17:03:28 的 end 沒有可關的 id）。它做了同樣的 tar 部署與忽略清單修改，結果與本單一致；另外在 cron 多插一行重複的 `AUDIT_QUIET_FROM`，並在 17:01:21 以 `cp -p` 把 `/etc/cron.d/onprem-logs.bak-cr0008` 蓋成已含新行的版本。處理：17:03 刪掉重複行，`.bak-cr0008` 以現行檔去掉本單新增的 4 行重建（4,045 B，與改前大小相同，mtime 設回 10/11 16:35）；其他 `*.bak-cr0008` 逐一確認與 v0.5.1 不同、mtime 為 10/11，是改前的檔。註記 4 以 PATCH 補上結束時間，silence 49f009e1 以 API 提前結束。之後兩個工作階段約定：本單、CR-2026-0009、CR-2026-0010 只由本工作階段執行。
- followups:
  - 10/13 以 `sum(audit_unattributed) > 0` 的非零取樣比例，對照 CR-2026-0007 之後的 115/124。
  - 一張單同時由兩個執行者開窗，change-mark 的狀態檔會互相覆蓋。change-mark.sh 可考慮在 begin 時發現已有狀態檔就拒絕，另開單處理。
