---
id: CR-2026-0010
title: spark02 的 journal-upload 改成持續重試並補回停送的日誌，之後關閉 SSH 密碼登入
status: scheduled
type: standard
risk: medium
requester: ivan
implementer: ivan
approver: ivan
approval_exception: 單人實驗場域，沒有第二位核准者。以 PR 自審取代，PR 開啟與合併時間在 git log。冷卻未滿 24 小時，由擁有者以管理員略過合併
assets:
  - spark02/DGX OS
window_start: 2026-10-12T22:00:00+08:00
window_end: 2026-10-12T23:00:00+08:00
silence: 
tags: journald, ssh, spark02, hardening, day27
---

## 目的

1. spark02 從 2026-10-07 起沒有把 journal 送到收集端。開機時 `network-online.target` 已到，但收集端還連不上（`Failed to connect to 192.168.2.49 port 9428 after 0 ms`），原本的 unit 每 100 ms 重試、10 秒內失敗 5 次就放棄，之後再也沒起來。10/07 00:15 與 10/12 01:59 兩次開機都是這樣。Day 27 的稽核因此完全看不到 spark02。onprem-logs v0.5.2 的 `node/install-journal-upload.sh` 加了每 30 秒重試、不設次數上限的 drop-in，在 spark02 上重跑一次。
2. 與 CR-2026-0009 同一個理由，關閉 spark02 的 SSH 密碼登入。90 天內 spark02 的密碼登入有 4 次：經 tailnet（來源 .2）3 次，最後 9/29；直連網段 192.168.100.2 1 次，9/25。.10 桌機的金鑰已在 spark02 上；加上 Dell 筆電的金鑰（同 CR-2026-0009）。

窗排在 CR-2026-0009（21:00 到 22:00）之後，不重疊，報告才不會把 spark02 的動作歸到 CR-0009。

## 影響

- journal-upload 重新啟動後從 10/07 的游標續傳，約 5.5 萬行的積壓會在幾分鐘內進 VictoriaLogs。10/07 到 10/11 的 WORM 封存已經寫好、不含 spark02，補不回去，記在紀錄段。
- 腳本會重啟 systemd-journald，journal 本身不中斷。
- 關閉密碼後只能用金鑰登入；.10、Dell 筆電、spark01（`cyberq@cyberq-dgx-spark` 金鑰）、收集端的 pcap 都是金鑰。不套 spark-baseline 的完整加固，理由同 CR-2026-0009（`AllowGroups` 會擋 pcap）。

## 步驟

時間為 +08:00。

在收集端上：

```sh
sudo -u metrics /opt/onprem-changes/scripts/change-mark.sh begin CR-2026-0010
```

在 spark02 上（安裝腳本是 onprem-logs v0.5.2，65eda95）：

```sh
sudo sh ./install-journal-upload.sh http://192.168.2.49:9428
systemctl cat systemd-journal-upload.service | grep -A4 10-retry
```

在收集端上確認 spark02 的新行與補回的行：

```sh
curl -s http://127.0.0.1:9428/select/logsql/query -d 'query=_time:6d _HOSTNAME:=cyberq-dgxspark2 | stats count() as n, min(_time) as first, max(_time) as last'
```

在 spark02 上，以 cyberq 身分加 Dell 筆電的公鑰（`SHA256:CQIIdgK0…`），從 Dell 經 tailnet 以金鑰登入一次並在收集端確認，然後關密碼：

```sh
cp -p ~/.ssh/authorized_keys ~/.ssh/authorized_keys.bak-cr0010
cat dell.pub >> ~/.ssh/authorized_keys
sudo install -m 644 -o root -g root 10-password-off.conf /etc/ssh/sshd_config.d/10-password-off.conf
sudo sshd -t
sudo sshd -T | grep -Ei '^(passwordauthentication|kbdinteractiveauthentication|pubkeyauthentication) '
sudo systemctl try-reload-or-restart ssh.service
```

`10-password-off.conf` 與 CR-2026-0009 相同。

在收集端上：

```sh
sudo -u metrics /opt/onprem-changes/scripts/change-mark.sh end CR-2026-0010
```

## 驗證

- `systemd-journal-upload` active，drop-in 生效（`RestartSec=30s`、`StartLimitIntervalSec=0`）；收集端看得到 spark02 窗內的新行，以及 10/07 起補回的行。
- `sshd -T` 為 `passwordauthentication no`；從 .10 與 Dell 各一個新的金鑰登入成功；`ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password cyberq@spark02` 被拒。
- 收集端下一次 :40 的 pcap 拉取對 spark02 照常成功。
- 稽核報告把窗內 spark02 的 sudo 與登入歸到本單。

## 回退

```sh
sudo rm /etc/ssh/sshd_config.d/10-password-off.conf
sudo systemctl try-reload-or-restart ssh.service
sudo rm /etc/systemd/system/systemd-journal-upload.service.d/10-retry.conf && sudo systemctl daemon-reload
```

被鎖在外面時從 spark01 經 200GbE 直連網段以金鑰登入，或從本機主控台。`authorized_keys` 以 `.bak-cr0010` 覆回。

## 紀錄

執行時填。

- begin: 
- end: 
- result: 
- evidence: 
- followups: 
