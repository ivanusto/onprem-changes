---
id: CR-2026-0011
title: 回退演練，把三張已完成單子的回退段真的做一次（Day 28）
status: done
type: standard
risk: medium
requester: ivan
implementer: ivan
approver: ivan
approval_exception: 單人實驗場域，沒有第二位核准者。以 PR 自審取代，PR 開啟與合併時間在 git log。冷卻未滿 24 小時，因為 Day 28 當天要發表，由擁有者以管理員略過合併。窗排在 CR-2026-0008（17:00 到 18:00）之後、CR-2026-0009（21:00 起）之前，不重疊，報告才不會把動作歸錯單
assets:
  - qdevice/Debian
  - fgt-edge/FortiOS
  - collector/Ubuntu
window_start: 2026-10-12T19:00:00+08:00
window_end: 2026-10-12T21:00:00+08:00
silence: alertname=~"PveQdeviceAbsent|TargetDown"
tags: drill, rollback, qdevice, fortigate, victorialogs
---

## 目的

Day 26 起到窗開始時已完成的八張單（CR-2026-0001 到 0008；0009 與 0010 排在今晚本單之後）都有回退段，沒有一段被執行過。CR-2026-0001 的快照拍了但沒用上，其餘的回退段是寫的時候想的。這張單挑三張單的回退段照著做一次，量從開始回退到驗證通過的時間，記在 `drills/drills.md`（Day 15 的格式）與 `drills/drills.jsonl`（Day 16 到 18 的格式，`drills/drill.sh` 寫），回退之後再往前做回今天的狀態。三張單的選法：一張靠快照（CR-2026-0001），一張靠設定反向（CR-2026-0007 的 60F 那一半），一張靠備份檔覆回並重建容器（CR-2026-0007 的收集端那一半）。兩張 `risk: high` 的單子這次不做。CR-2026-0004 的回退（關 policy 1 的 UTM）會在 Day 24 的網頁過濾紀錄留一個洞，Day 25 的 AUP 週報要連續的資料。CR-2026-0005 的回退是讓一個 silence 失效，會讓二十項授權的告警立刻響，演練的代價是一輪告警與一次 TSV 的來回，留給下一張單。這兩張會留在 Day 28 新加的 `ChangeRollbackUndrilled` 裡，那是刻意的。

每一次回退都會在 Day 27 的稽核軌跡留下動作，也會讓 Day 25 的 EOL 清單與 Day 27 的規則有反應，這張單的驗證段要把這些反應逐一對上。

## 影響

- 演練 1（QDevice VM 快照回復）：VM 要關機、回復、開機，QDevice 缺席約 1 到 3 分鐘（Day 15 演練 4 量過從關到回來約 6 分鐘，CR-2026-0001 的重開機量到約 30 秒）。期間叢集 2/3 票，Quorate 維持 Yes，`check.sh` 回 1，`PveQdeviceAbsent`（for 2m）可能響，在 `silence` 欄靜音。窗不與任何節點或 NAS 的維護重疊。快照是今天拍的新快照 `CR-2026-0011-before`，CR-2026-0001 的 `CR-2026-0001-before` 已在 10/11 依該單的 followups 刪除。回復快照會把 VM 回到拍快照當下，包含它之後寫進去的 journald 與 apt 狀態，所以回復之前先在 VM 上做一個看得見的無害變更（放一個標記檔），回復之後標記檔消失就是回退成功的證據之一。
- 演練 2（60F syslogd3 停用再啟用）：停用期間 60F 的事件日誌只留在它自己的記憶體日誌，收集端 515 收不到。預期的現象，停用那一筆設定變更不會送到收集端（送出時 syslogd3 已停），啟用那一筆會送到，而且要被 `audit-report.py` 歸到本單。Day 27 的 `AuditUnattributedFortigateChange` 不應響，因為在窗內。
- 演練 3（收集端 VictoriaLogs 以 `.bak-day27` 覆回，單一監聽器）：容器重建兩次（回退一次、往前一次），每次停約 10 秒，`TargetDown`（for 5m）正常不會響，仍在靜音範圍。回退期間 515 關閉，60F 的 syslogd3 連不上，reliable 模式會重試，期間產生的事件是否在重連後補送，這是本單要量的事（待查，FortiOS 文件沒有明說）。514 的 traffic 與 NAS、9428 的 journald 在重建後要重連，Day 27 量過最慢的是 Spark 的 journal-upload，2 分 30 秒內。
- 三次演練的 begin 與 end 都以 `drill.sh` 寫進 journal（tag `drill`），稽核報告會看到它們落在本單的窗內。
- Day 25 的清單不會變，QDevice 回復的是今天拍的 Debian 13 快照，`qdevice/Debian` 兩列仍是 13。這一點在驗證段確認。

## 步驟

時間為 +08:00。工作站上有 onprem-changes 的 clone（`~/onprem-changes`，以下稱 `$CH`），drills 的三行會在窗結束後與本單的 `紀錄` 一起以 PR 合併。

在收集端上：

```sh
sudo -u metrics /opt/onprem-changes/scripts/change-mark.sh begin CR-2026-0011
```

### 準備：收集端換上 Day 28 的規則、cron 與載入器

onprem-metrics v0.3.3 的 `changes.yml`（多兩條規則）、`collector.cron`（`drills-textfile.py` 多一個來源 `changes`）、Alertmanager 的 inhibit 多兩個規則名；onprem-logs v0.5.3 的 `fortigate-export-load.py`（`--field`）。舊檔留 `*.bak-cr0011`。新檔由工作站以 `scp` 放到收集端的 `~/cr0011/`（onprem-metrics 與 onprem-logs 的 day28 分支，保留目錄結構）。Alertmanager 與規則是單檔 bind mount，用 `cp` 覆寫原檔（同一個 inode），不用 `sed -i`。

```sh
cd /opt/onprem-metrics
sudo cp -p prometheus/rules/changes.yml prometheus/rules/changes.yml.bak-cr0011
sudo cp -p /etc/cron.d/onprem-metrics /etc/cron.d/onprem-metrics.bak-cr0011    # cron 不讀有 . 的檔名
sudo cp -p alertmanager/alertmanager.yml alertmanager/alertmanager.yml.bak-cr0011
sudo cp ~/cr0011/onprem-metrics/prometheus/rules/changes.yml prometheus/rules/changes.yml
sudo install -m 0644 ~/cr0011/onprem-metrics/systemd/collector.cron /etc/cron.d/onprem-metrics
sed 's/Change(WindowOverrun|InProgress|StartsSoon)"/Change(WindowOverrun|InProgress|StartsSoon|RollbackUndrilled|RollbackDrillStale)"/' \
  alertmanager/alertmanager.yml > /tmp/am.yml && sudo cp /tmp/am.yml alertmanager/alertmanager.yml
curl -s -X POST http://127.0.0.1:9090/-/reload && curl -s -X POST http://127.0.0.1:9093/-/reload
sudo cp -p /opt/onprem-logs/firewall/fortigate-export-load.py /opt/onprem-logs/firewall/fortigate-export-load.py.bak-cr0011
sudo cp ~/cr0011/onprem-logs/firewall/fortigate-export-load.py /opt/onprem-logs/firewall/fortigate-export-load.py
```

Prometheus 的 Rules 頁 `changes` 群組六條，Alertmanager 的 status 頁 inhibit 有新的兩個名字。這一步的回退是把三個 `*.bak-cr0011` 覆回並 reload。

### 演練 1：CR-2026-0001 的回退，Virtualization Station 快照回復

在 NAS 的 Virtualization Station 上，對 QDevice VM 拍快照 `CR-2026-0011-before`（VM 運作中）。

QDevice VM 上目前沒有工作站的公鑰（CR-2026-0006 收尾時刪了），標記檔與驗證命令都要登入 VM。拍快照之前，由擁有者在 VM 主控台把工作站的公鑰暫時加進 `~ivan/.ssh/authorized_keys`，先加再拍，回復快照之後公鑰仍在，演練結束後刪除（見收尾）。

在 QDevice VM 上（ivan 以金鑰登入）：

```sh
echo "CR-2026-0011 $(date -Is)" | sudo tee /var/tmp/CR-2026-0011-marker
```

在 Virtualization Station 上關閉 VM。在 pve1 上確認 `check.sh` 回 1（QDevice 缺席）。然後在工作站上啟動計時，再回到 Virtualization Station 回復快照 `CR-2026-0011-before` 並開機：

```sh
$CH/drills/drill.sh qdevice-snapshot-revert CR-2026-0011 CR-2026-0001 -t 600 -- \
  sh -c 'ssh root@pve1 /root/pve-qdevice-on-qnap/check.sh && ssh qdevice test ! -e /var/tmp/CR-2026-0011-marker'
```

驗證命令要兩件事同時成立，pve1 的 `check.sh` 回 0（三票都在），而且標記檔不在（VM 真的回到了快照）。通過後，在 Virtualization Station 刪快照 `CR-2026-0011-before`。

### 演練 2：CR-2026-0007 的回退（60F 那一半），syslogd3 停用再啟用

在工作站上，先備份，再記下 60F 記憶體日誌目前最後一筆事件的 `eventtime`：

```sh
curl -sk -X POST -H "Authorization: Bearer $FGT_TOKEN" \
  "https://<fgt-edge>:10443/api/v2/monitor/system/config/backup?scope=global&destination=file" -o fgt-backup-CR-2026-0011.conf
curl -sk -H "Authorization: Bearer $FGT_TOKEN" "https://<fgt-edge>:10443/api/v2/cmdb/log.syslogd3/setting" | python3 -m json.tool | grep -E '"(status|server|port|mode|format)"'
```

停用（這一筆設定變更預期不會到收集端），等 60 秒，再啟用並同時計時。驗證命令查收集端最近 5 分鐘有沒有 `log.syslogd3.setting` 的 `status[disable->enable]`：

```sh
curl -sk -X PUT -H "Authorization: Bearer $FGT_TOKEN" -H 'Content-Type: application/json' \
  "https://<fgt-edge>:10443/api/v2/cmdb/log.syslogd3/setting" -d '{"status":"disable"}'
sleep 60
$CH/drills/drill.sh fgt-syslogd3-reenable CR-2026-0011 CR-2026-0007 -t 300 -- \
  sh -c 'ssh collector curl -s http://127.0.0.1:9428/select/logsql/query --data-urlencode "query=_time:5m audit:=fortigate \"log.syslogd3.setting\" \"status[disable->enable]\"" | grep -q cfgtid' &
sleep 2
curl -sk -X PUT -H "Authorization: Bearer $FGT_TOKEN" -H 'Content-Type: application/json' \
  "https://<fgt-edge>:10443/api/v2/cmdb/log.syslogd3/setting" -d '{"status":"enable"}'
wait
```

`drill.sh` 以背景啟動，2 秒後送啟用，所以 `rto_s` 含這 2 秒。驗證通過後讀回 setting 與 Day 27 的四個欄位相同，與備份比對。

### 演練 3：CR-2026-0007 的回退（收集端那一半），compose 以 `.bak-day27` 覆回

在收集端上，記下重建前每個來源的最後時間與 515 的連線，然後覆回並重建，同時計時。回退的驗證是容器回答 `/health` 而且 515 不再監聽：

```sh
curl -s http://127.0.0.1:9428/select/logsql/query -d 'query=_time:30m | stats by (hostname, _HOSTNAME) max(_time) as last'
vlns() { sudo nsenter -t "$(sudo docker inspect -f '{{.State.Pid}}' onprem-logs-victorialogs-1)" -n "$@"; }
vlns ss -tn state established '( sport = :514 or sport = :515 )'
cd /opt/onprem-logs
sudo cp docker-compose.yml docker-compose.yml.day28-current
$CH/drills/drill.sh vlogs-single-listener-restore CR-2026-0011 CR-2026-0007 -t 300 -- \
  sh -c 'curl -sf http://127.0.0.1:9428/health >/dev/null && ! ss -ltn | grep -q ":515 "' &
sleep 2
sudo cp docker-compose.yml.bak-day27 docker-compose.yml && sudo docker compose up -d
wait
```

回退狀態停留 10 分鐘。這 10 分鐘內在工作站上對 60F 做一個無害的設定變更（例如改一條 policy 的 `comments`），讓 60F 的記憶體日誌裡有一筆收集端收不到的設定變更。10 分鐘後往前做回今天的狀態，一樣計時，驗證是 515 回來而且 60F 的 TCP 連線重建：

```sh
$CH/drills/drill.sh vlogs-second-listener-forward CR-2026-0011 CR-2026-0007 -t 300 -- \
  sh -c 'curl -sf http://127.0.0.1:9428/health >/dev/null && sudo nsenter -t "$(sudo docker inspect -f "{{.State.Pid}}" onprem-logs-victorialogs-1)" -n ss -Htn state established "( sport = :515 )" | grep -q .' &
sleep 2
sudo cp docker-compose.yml.day28-current docker-compose.yml && sudo docker compose up -d
wait
```

連線要在容器的網路空間裡看。515 是 docker 以 DNAT 轉進容器的，主機上的 `ss -tn` 看不到 60F 的連線（主機上只看得到 docker-proxy 的 `LISTEN`），所以回退的驗證用主機的 `ss -ltn`，往前的驗證用 `nsenter` 進容器，容器重建後 PID 會變，每一次檢查都重新取。

往前這一行的 `rollback_of` 仍寫 CR-2026-0007，因為它量的是同一張單回退之後再往前的路。`label` 不同，`changes-textfile.py` 只看 `rollback_of`，所以 CR-2026-0007 的「最後一次回退演練」會是這一行，`紀錄` 要寫清楚。

重建後 5 分鐘內，重建前列出的每個 `hostname` 與 `_HOSTNAME` 都要再出現。然後查 10 分鐘的洞，60F 記憶體日誌裡那一筆無害變更的 `cfgtid`，在收集端查不查得到：

```sh
curl -s http://127.0.0.1:9428/select/logsql/query -d 'query=_time:30m audit:=fortigate cfgpath:* | unpack_logfmt from _msg fields (cfgtid, cfgpath, cfgattr) | fields _time, cfgtid, cfgpath, cfgattr'
```

查不到（預期如此，待查）就把洞補起來，從 60F 匯出那段時間的事件日誌，以 onprem-logs 的 `fortigate-export-load.py` 載入收集端，加上 `--field audit=fortigate`、`--devname fgt-edge`、`--source backfill`，這樣它們會進 `audit-fgt-edge` 的封存，`source` 欄又能把它們與即時收到的分開：

```sh
./firewall/fortigate-export-load.py fgt-event-export.log --vl http://127.0.0.1:9428 \
  --field audit=fortigate --devname fgt-edge --source backfill
```

### 收尾

在收集端上：

```sh
sudo -u metrics env CHANGES_DIR=/opt/onprem-changes/changes /opt/onprem-logs/audit/audit-report.py --since 2026-10-12T19:00:00+08:00
sudo -u metrics /opt/onprem-changes/scripts/change-mark.sh end CR-2026-0011
```

在 QDevice VM 主控台，由擁有者把演練前暫時加入的工作站公鑰從 `~ivan/.ssh/authorized_keys` 刪除，從工作站再 ssh 一次確認被拒。

在工作站上，`drills/drills.jsonl` 的四行、`drills/drills.md` 的觀察欄、本單的 `紀錄` 一起開 PR 合併。收集端的 clone 15 分鐘內拉到，`changes-textfile.py` 下一輪寫出 `change_rollback_last_drill_*`。

## 驗證

- `drills/drills.jsonl` 四行，`result` 都是 OK，`rto_s` 各自低於 `-t`。
- 演練 1：pve1 `check.sh` 回 0，`pvecm status` 的 Qdevice 列一票，Day 20 的 `pve_quorum_qdevice_votes` 回到 1。VM 上 `cat /etc/debian_version` 仍是 13.x，`/var/tmp/CR-2026-0011-marker` 不存在。Day 25 的 `eol/assets.tsv` 不用改，`AssetEolPassed` 不出現。NAS 事件日誌有建立快照、關機、回復、開機、刪快照的行，`audit-report.py` 把它們歸到本單。
- 演練 2：收集端有 `status[disable->enable]` 那一筆（logid `0100044546`，`cfgpath="log.syslogd3.setting"`），沒有 `status[enable->disable]` 那一筆。60F 的記憶體日誌兩筆都有。`AuditUnattributedFortigateChange` 不響。
- 演練 3：回退後 `ss -ltn` 沒有 515，往前後有，60F 的 TCP 515 連線在 `ss -tn` 裡。重建前的每個來源在 5 分鐘內都有新行。10 分鐘洞裡的那一筆設定變更，收集端在補載之前查不到、補載之後查得到，`source=backfill`。隔天 08:10 的 `MANIFEST.tsv` 的 `audit-fgt-edge` 列包含補載的行（行數比對）。
- 稽核：`audit-report.py --since 2026-10-12T19:00:00+08:00` 把窗內的 `drill` 標記、sudo、SSH 登入、NAS 事件、60F 的設定變更都歸到本單，「沒有變更單的動作」沒有這幾類。
- 指標：收集端 clone 拉到 PR 之後，`changes.prom` 有 `change_rollback_last_drill_timestamp{id="CR-2026-0001"}` 與 `{id="CR-2026-0007"}`，`change_rollback_last_drill_result` 都是 1，`drills.prom` 有 `drill_last_result{source="changes",label=...}` 四列。Day 28 的 `ChangeRollbackUndrilled` 列出 CR-2026-0004 與 CR-2026-0005（兩張 high 而沒演練過回退的單），不列其他。

## 回退

這張單的每一步本身就是回退與往前的一對。中途失敗的處置：

- 演練 1 回復快照後 VM 起不來或 `check.sh` 不回 0：照 CR-2026-0001 的第二條回退，依 qnetd-vm.md 重建，`pvecm qdevice remove` 再 `setup`。這會超出窗，先在 pve1 確認 Quorate 仍是 Yes，再決定今晚做還是另開單。
- 演練 2 啟用後收集端收不到：讀回 setting 與備份比對，必要時 `PUT` 整組四個欄位（Day 27 的 `fortigate-syslogd3.md`）。60F 這一側最壞的狀況是 syslogd3 維持停用，與 Day 27 之前相同，事件仍在記憶體日誌裡約 8 天。
- 演練 3 往前失敗：`docker-compose.yml.day28-current` 是今天開始前的檔，`docker compose up -d` 回去。VictoriaLogs 的資料在 volume 裡，兩次重建都不碰它。
- 任何一步失敗，`drill.sh` 會在 `-t` 到期時寫 `result: FAIL`，那一行照樣 commit，`DrillFailed` 會響，演練的結果照實寫。

## 紀錄

- begin: 2026-10-12T19:37:35+08:00（收集端 `change-mark.sh begin`，Grafana 註記 6，silence 73cdbefa 到 21:00）。PR #7 在 16:59 合併，窗開始後 37 分鐘才施作
- end: 2026-10-12T20:00:59+08:00，註記 6 補上 timeEnd，silence 失效
- result: 完成，四行 `drills.jsonl` 都是 OK。演練 1 rto_s 19、演練 2 rto_s 8、演練 3 回退 rto_s 4、往前 rto_s 14。三個預期有兩個不成立，照實寫在 `drills/drills.md`：運作中拍的快照連記憶體一起回復，VM 是接續不是開機（時鐘落後 3 分 31 秒，pve1 `check.sh` 通過之後兩台節點的 qdevice 用戶端又過了約 1 分鐘才連上，`pve_quorum_qdevice_votes` 0 一個取樣，`PveQdeviceAbsent` 只到 pending）；syslogd3 停用那一筆設定變更也送到了收集端；515 關閉 10 分鐘期間 60F 的事件只遺失斷線後的第一筆，其餘重連後補送
- evidence: 部署 19:37 到 19:38，`changes` 群組六條、Alertmanager inhibit 含兩個新名字，舊檔 `*.bak-cr0011`。演練 1 標記檔 19:38:29 寫入，`check.sh` 19:39:21 回 1、19:39:41 回 0，回復後 Debian 13.7、標記檔不在。演練 2 60F 記憶體日誌與收集端都有 cfgtid 10289368（`status[enable->disable]`，19:45:29）與 10289369（`status[disable->enable]`，19:46:31），讀回 setting 與備份相同。演練 3 洞內 cfgtid 10289370 收集端沒有，以 `fortigate-export-load.py --source backfill` 補 1 行；10289371 與兩筆效能統計由 60F 重連後補送。`audit-report.py --since 19:00` 管理動作 118 筆全部歸到本單（fortigate-config 4、nas-event 3、ssh-login 66、sudo 45），沒有變更單的 0 筆
- followups: 收集端 17:25 被 PVE 的 stopall 關機（17:18 到 17:29 兩台 PVE 經網頁 shell 做 apt 更新並重開，沒有單子），開機時 `systemd-journal-upload` 一秒內放棄，17:29 到 20:00 收集端自己的 journal 沒有上傳，20:00:34 在本單窗內 `systemctl restart`，以 `--save-state` 的游標補回，收集端也要裝 v0.5.2 的重試 drop-in（另開單）；主 NAS 的出向位址 10/11 19:12 起改成 .30，收集端的 `SYSLOG_FROM` 只有 .2，DOCKER-USER 一直在丟，10/11 19:09 起主 NAS 的事件日誌沒有進收集端，演練 1 的 Virtualization Station 事件也因此查不到（另開單）；窗內 19:15 副 NAS 的 App Center 更新 3 筆也被歸到本單，報告只看時間不看動作；QDevice VM 的暫時公鑰 20:00 前已由擁有者刪除，快照 `CR-2026-0011-before` 已刪；隔天 08:10 看 `audit-<devname>` 的 MANIFEST 列是否含補載的那一行（2026-10-13 已確認：10/12 的 `MANIFEST.tsv` 中 `audit-<devname>` 500 行，封存檔內有 `source` 為 backfill 的 cfgtid 10289370 那 1 行，archive status 0）
