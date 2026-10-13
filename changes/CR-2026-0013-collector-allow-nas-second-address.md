---
id: CR-2026-0013
title: 收集端的允許清單加上主 NAS 的第二個位址 .30，恢復主 NAS 事件日誌的收集
status: done
type: standard
risk: low
requester: ivan
implementer: ivan
approver: ivan
approval_exception: 單人實驗場域，沒有第二位核准者。以 PR 自審取代，PR 開啟與合併時間在 git log。冷卻未滿 24 小時，因為主 NAS 的事件日誌已斷超過 25 小時、每多一小時稽核軌跡就多一小時的洞，由擁有者以管理員略過合併
assets:
  - collector/Ubuntu
window_start: 2026-10-12T23:00:00+08:00
window_end: 2026-10-12T23:30:00+08:00
silence: 
tags: victorialogs, allowlist, nas, audit, day28
---

## 目的

CR-2026-0011（Day 28 回退演練）在找演練 1 的 Virtualization Station 事件時發現：主 NAS 的事件日誌（QuLog 經 syslog 送到 514/tcp）從 2026-10-11 19:09:25 起沒有再進收集端。主 NAS 有兩個 LAN 位址，10/11 19:12 起出向的來源位址從 .2 變成 .30（同一件事讓 CR-2026-0008 把 HDP 的忽略規則改成只認金鑰），而收集端 `/etc/default/onprem-logs-allowlist` 的 `SYSLOG_FROM` 只有 .2。DOCKER-USER 的 `ONPREM-LOGS` 鏈把 .30 送來的連線丟掉，kernel log 每秒都有 `[DOCKER-USER DROP] SRC=192.168.2.30 ... DPT=514`。

後果：Day 27 稽核報告的 nas-event 來源這段時間只有副 NAS，`AuditUnattributedActions` 的 nas-event 計數是安靜的 0，與「沒有事」分不出來。CR-2026-0011 的演練 1（快照建立、關機、回復、開機、刪除）因此在收集端完全沒有紀錄。

這張單在 `SYSLOG_FROM` 加上 .30，保留 .2（主 NAS 哪天出向又換回 .2 時不會再斷一次）。不改主 NAS 的路由或出向位址，那是另一件事。

## 影響

- `onprem-logs-allowlist.service` 重新套用時，腳本先刪掉 `ONPREM-LOGS` 鏈再重建，中間約一秒沒有鏈，這一秒是放行不是阻擋，既有連線（`ESTABLISHED`）不受影響，不會掉資料。
- 開放之後主 NAS 會重連 514/tcp。QuLog 在連不上的這 25 小時有沒有緩衝、重連後會不會補送，文件沒有寫，這張單會量。若不補送，洞只能從主 NAS 的 QuLog Center 匯出補回，另做。
- 不需要靜音：沒有告警依賴主 NAS 的 514 連線本身。重連後如果補送大量舊事件，`AuditUnattributedActions` 的 nas-event 可能在下一輪報告出現 10/11 19:09 以來沒有單子的 NAS 動作，那些是真的動作，照實看，不靜音。

## 步驟

時間為 +08:00，全部在收集端上。

```sh
sudo -u metrics /opt/onprem-changes/scripts/change-mark.sh begin CR-2026-0013
sudo journalctl -k --since -2min | grep -c 'DOCKER-USER DROP.*SRC=192.168.2.30.*DPT=514'     # 改之前，預期非 0
sudo iptables -S ONPREM-LOGS | grep -- '--ctorigdstport 514' | grep -c RETURN                    # 改之前，預期 6（3 台 x tcp/udp）
sudo cp -p /etc/default/onprem-logs-allowlist /etc/default/onprem-logs-allowlist.bak-cr0013
sudo sed -i 's/^SYSLOG_FROM="192.168.2.2 /SYSLOG_FROM="192.168.2.2 192.168.2.30 /' /etc/default/onprem-logs-allowlist
sudo sed -i '3a # The primary NAS has two LAN addresses (.2 and .30) and sends from either; list both (CR-2026-0013).' /etc/default/onprem-logs-allowlist
grep ^SYSLOG_FROM /etc/default/onprem-logs-allowlist     # SYSLOG_FROM="192.168.2.2 192.168.2.30 192.168.2.22 192.168.2.99"
sudo systemctl restart onprem-logs-allowlist.service
sudo iptables -S ONPREM-LOGS | grep -- '-s 192.168.2.30/32'
sudo -u metrics /opt/onprem-changes/scripts/change-mark.sh end CR-2026-0013
```

`/etc/default/onprem-logs-allowlist` 不是單檔 bind mount，`sed -i` 可以用。開單時實測收集端的 kernel log 在 2 分鐘內有 6 行來自 .30 的 DROP（LOG 規則另有 6/min 的限速，行數不等於被丟的封包數）。

## 驗證

- `iptables -S ONPREM-LOGS` 有 `-s 192.168.2.30/32` 的 514 tcp 與 udp 兩條 RETURN，514 的 RETURN 共 8 條。
- 改完 2 分鐘後，`journalctl -k --since -2min | grep 'SRC=192.168.2.30.*DPT=514'` 是 0 行。
- VictoriaLogs `_time:10m hostname:=<主 NAS 的 hostname>` 有新行。
- 記下第一筆新行的 `_time`，以及 `_time:[2026-10-11T19:09:00+08:00, 2026-10-12T23:00:00+08:00] hostname:=<主 NAS>` 的行數，判斷 QuLog 有沒有補送那 25 小時。
- 下一輪（10 分鐘內）`audit.prom` 的 `audit_actions{source="nas-event"}` 開始計入主 NAS。

## 回退

`/etc/default/onprem-logs-allowlist` 以 `.bak-cr0013` 覆回，`systemctl restart onprem-logs-allowlist.service`。回退之後主 NAS 回到被擋的狀態，與改之前相同。

## 紀錄

- begin: 2026-10-12T23:00:59+08:00（收集端 `change-mark.sh begin`，Grafana 註記 7，沒有靜音）
- end: 2026-10-12T23:06:47+08:00，註記 7 補上 timeEnd
- result: 完成。.30 的 514 放行，主 NAS 23:02:01 起重新進收集端。QuLog 沒有補送斷線的這段：`_time:[2026-10-11T19:09:00+08:00, 2026-10-12T23:00:00+08:00] hostname:=<主 NAS>` 改前改後都是 2 行（都在 10/11 19:09:00 到 19:09:25，斷線之前），約 28 小時的洞只能從主 NAS 的 QuLog Center 匯出補回。`audit_actions{source="nas-event"}` 到 end 時仍是 0：nas-event 只算 QuLog 的 `event log:` 行，重連後進來的 5 行都是 `conn log:`（Day 19 指標 cron 每分鐘以 SSH 登入主 NAS），主 NAS 下一筆事件日誌出現時才會計入，不是本單的缺口
- evidence: 改前 2 分鐘內 kernel log 有 6 行 `[DOCKER-USER DROP] SRC=192.168.2.30 ... DPT=514`（PROTO=TCP、SYN），`ONPREM-LOGS` 的 514 RETURN 6 條。23:01:12 `sed` 兩行並 `systemctl restart onprem-logs-allowlist.service`，`diff` 與 `.bak-cr0013` 只差註解一行與 `SYSLOG_FROM` 多了 .30；`-s 192.168.2.30/32` 的 tcp 與 udp 兩條 RETURN，514 的 RETURN 共 8 條。23:01:13 到 23:06:40 .30 的 DROP 0 行。VictoriaLogs 第一筆新行 `_time` 2026-10-12T23:02:01+08:00（`app_name` qulogd），到 23:06:01 共 5 行，每分鐘一行。改前的 DROP 每分鐘換一個來源埠（23:00:03 的 49048、23:01:01 到 23:01:03 的 47762 重送 SYN），容器網路命名空間裡的 `ss` 在 23:02:10 與 23:06:40 都看不到 .30 的常駐 514 連線，QuLog 看起來是每次送出各開一條連線、連不上就丟掉那一筆，所以沒有緩衝可補
- 隔天補記：2026-10-13 15:30 重查，10/11 19:09 到 10/12 23:00 仍是 2 行，QuLog 確定沒有補送；主 NAS 的第一筆 `event log:` 在 2026-10-13 05:00:10，之後 `audit_actions{source="nas-event"}` 計入主 NAS。08:10 的封存 `syslog-<主 NAS>` 一列 545 行，封存以 UTC 日界切，含 10/13 早上的行
- followups: 從主 NAS 的 QuLog Center 匯出 2026-10-11 19:09 到 2026-10-12 23:02 的事件日誌，以 `--source backfill` 補載（另開單）；主 NAS 下一筆 `event log:` 出現後確認 `audit_actions{source="nas-event"}` 計入主 NAS；onprem-logs 預設允許清單已含 .30（day28 分支，v0.5.3 發布）
