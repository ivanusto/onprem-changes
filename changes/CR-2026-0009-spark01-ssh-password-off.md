---
id: CR-2026-0009
title: spark01 關閉 SSH 密碼登入，經 tailnet 的裝置改用金鑰
status: scheduled
type: standard
risk: medium
requester: ivan
implementer: ivan
approver: ivan
approval_exception: 單人實驗場域，沒有第二位核准者。以 PR 自審取代，PR 開啟與合併時間在 git log。冷卻未滿 24 小時，由擁有者以管理員略過合併
assets:
  - spark01/DGX OS
window_start: 2026-10-12T21:00:00+08:00
window_end: 2026-10-12T22:00:00+08:00
silence: 
tags: ssh, spark01, hardening, day27
---

## 目的

Day 27 第一份稽核報告的發現：spark01 有經 tailnet 進來的密碼登入（90 天內 cyberq 帳號從 NAS 的兩個位址 .2 與 .30 共 64 次，最後一次 10/12 13:10）。管理者已確認是本人，但這條路徑是「從外部經 tailnet、以密碼登入 GPU 主機」，改成金鑰。spark01 目前 `sshd_config.d` 是空的，主設定只關了 `KbdInteractiveAuthentication`，密碼登入是開著的。

不套 spark-baseline 的完整 `10-hardening.conf`：其中的 `AllowGroups sshusers` 會擋掉 Day 23 的 pcap 帳號（收集端每小時 :40 以金鑰登入拉擷取檔），這次只關密碼。

## 影響

- 以密碼的 SSH 登入不再可行。90 天內用密碼的只有上述 tailnet 裝置；.10 的 cyberq 與收集端的 pcap 都是金鑰，不受影響。
- 本機主控台與桌面登入不經 sshd，不受影響。
- sshd 只 reload，已建立的 session 不中斷。

## 步驟

時間為 +08:00。

在收集端上：

```sh
sudo -u metrics /opt/onprem-changes/scripts/change-mark.sh begin CR-2026-0009
```

在 tailnet 裝置上，產生金鑰並把公鑰交給實施者：

```sh
ssh-keygen -t ed25519 -C "<裝置名稱> tailnet"
```

在 spark01 上，以 cyberq 身分：

```sh
cp -p ~/.ssh/authorized_keys ~/.ssh/authorized_keys.bak-cr0009
cat <公鑰檔> >> ~/.ssh/authorized_keys
```

從 tailnet 裝置以金鑰登入一次，在收集端確認 `Accepted publickey for cyberq from 192.168.2.(2|30)` 與指紋。確認之後才在 spark01 上關密碼：

```sh
sudo install -m 644 -o root -g root 10-password-off.conf /etc/ssh/sshd_config.d/10-password-off.conf
sudo sshd -t
sudo sshd -T | grep -Ei '^(passwordauthentication|kbdinteractiveauthentication|pubkeyauthentication) '
sudo systemctl try-reload-or-restart ssh.service
```

`10-password-off.conf` 的內容：

```
PasswordAuthentication no
KbdInteractiveAuthentication no
```

在收集端上：

```sh
sudo -u metrics /opt/onprem-changes/scripts/change-mark.sh end CR-2026-0009
```

## 驗證

改之前開著的 session 保持不關，直到下列都通過：

- `sshd -T` 為 `passwordauthentication no`、`kbdinteractiveauthentication no`、`pubkeyauthentication yes`。
- 從 .10 與從 tailnet 裝置各開一個新的金鑰登入，成功。
- `ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password cyberq@spark01` 被拒，`Permission denied (publickey)`。
- 收集端下一次 :40 的 pcap 拉取照常成功（`pcap-pull.log`）。

## 回退

```sh
sudo rm /etc/ssh/sshd_config.d/10-password-off.conf
sudo systemctl try-reload-or-restart ssh.service
```

被鎖在外面時從本機主控台執行。`authorized_keys` 以 `.bak-cr0009` 覆回。

## 紀錄

執行時填。

- begin: 
- end: 
- result: 
- evidence: 
- followups: 
