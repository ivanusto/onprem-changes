---
id: CR-2026-0014
title: 收集端自己的 journal-upload 改成持續重試，開機時 VictoriaLogs 還沒起來也不會放棄
status: scheduled
type: standard
risk: low
requester: ivan
implementer: ivan
approver: ivan
approval_exception: 單人實驗場域，沒有第二位核准者。以 PR 自審取代，PR 開啟與合併時間在 git log。冷卻未滿 24 小時，由擁有者以管理員略過合併
assets:
  - collector/Ubuntu
window_start: 2026-10-12T23:30:00+08:00
window_end: 2026-10-13T00:00:00+08:00
silence: 
tags: journald, victorialogs, collector, audit, day28
---

## 目的

2026-10-12 17:24:55 兩台 PVE 的 `stopall` 把收集端這台 VM 關機（17:18 到 17:29 經網頁 shell 做 apt 更新並重開），17:29 開機。收集端的 `systemd-journal-upload` 送往本機的 `http://127.0.0.1:9428`，開機時 VictoriaLogs 的容器還沒起來，原廠 unit 每 100 ms 重試、10 秒內失敗 5 次就放棄（`Restart=on-failure`、`StartLimitBurst=5`），17:29:21 起一直是 failed。

後果：17:29 到 20:00 收集端自己的 journal 沒有上傳，包括收集端上的 sudo、`change-mark.sh` 的 begin 與 end、`drill.sh` 的標記。CR-2026-0011 第一次跑稽核報告時沒有任何 sudo，追下去才發現。20:00:34 在 CR-2026-0011 的窗內 `systemctl restart`，`--save-state` 的游標讓它從斷的地方補回，但下一次開機會再發生一次。

這與 CR-2026-0010 在 spark02 修的是同一個問題。onprem-logs v0.5.2 的 `node/install-journal-upload.sh` 會加 `10-retry.conf`（`StartLimitIntervalSec=0`、`Restart=on-failure`、`RestartSec=30s`），收集端當初跑的是 v0.5.2 之前的版本（`journald.conf.d/10-persistent.conf` 是它寫的），沒有重試的 drop-in。這張單在收集端重跑 v0.5.2 的版本，URL 用本機的 `http://127.0.0.1:9428`，不帶 `--from-now`，保留既有的游標。

## 影響

- `systemd-journal-upload` 重啟一次，從游標續傳，不會重送也不會漏。
- 腳本會確認 `systemd-journal-remote` 已安裝（已裝 255.4-1ubuntu8.17，apt 不會執行）、重寫 `journald.conf.d/10-persistent.conf`（內容相同）並 `systemctl restart systemd-journald`、重寫 `/etc/systemd/journal-upload.conf`（URL 不變）。journald 重啟時寫入端會短暫等待，不會掉行。
- 不需要靜音。

## 步驟

時間為 +08:00，全部在收集端上。腳本取 onprem-logs v0.5.2（65eda95）的 `node/install-journal-upload.sh`，由工作站以 `scp` 放到 `~/cr0014/`。

```sh
sudo -u metrics /opt/onprem-changes/scripts/change-mark.sh begin CR-2026-0014
systemctl show systemd-journal-upload -p Restart -p RestartUSec -p StartLimitBurst -p StartLimitIntervalUSec   # 改之前：on-failure、100ms、5、10s
sudo cp -p /etc/systemd/journal-upload.conf /etc/systemd/journal-upload.conf.bak-cr0014
sudo sh ~/cr0014/install-journal-upload.sh http://127.0.0.1:9428
systemctl show systemd-journal-upload -p Restart -p RestartUSec -p StartLimitBurst -p StartLimitIntervalUSec   # 改之後：on-failure、30s、StartLimitIntervalUSec=0
sudo -u metrics /opt/onprem-changes/scripts/change-mark.sh end CR-2026-0014
```

## 驗證

- `systemctl cat systemd-journal-upload` 末尾有 `10-retry.conf`，`systemctl is-active systemd-journal-upload` 是 active。
- end 那一行（`SYSLOG_IDENTIFIER:change "CR-2026-0014"`）在 1 分鐘內出現在 VictoriaLogs，`_HOSTNAME` 是收集端。
- 不在本單窗內做、但要確認一次的：收集端下一次重開機（或在另一張單裡 `docker stop` VictoriaLogs 3 分鐘再啟動），journal-upload 在容器回來後 30 秒內恢復上傳，不需要人去重啟。

## 回退

`sudo rm /etc/systemd/system/systemd-journal-upload.service.d/10-retry.conf`，`journal-upload.conf` 以 `.bak-cr0014` 覆回，`systemctl daemon-reload` 後 `systemctl restart systemd-journal-upload`。回到原廠的重試行為，與改之前相同。

## 紀錄

執行時填。

- begin: 
- end: 
- result: 
- evidence: 
- followups: 
