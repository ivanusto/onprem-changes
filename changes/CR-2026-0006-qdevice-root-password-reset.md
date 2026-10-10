---
id: CR-2026-0006
title: 為取得 sudo，重設 QDevice VM 的 root 密碼並裝 sudo（未經單子，事後補單）
status: done
type: emergency
risk: medium
requester: ivan
implementer: ivan
approver: ivan
approval_exception: 單人實驗場域，沒有第二位核准者。這是執行 CR-2026-0001 前的準備動作，當時沒有開單，24 小時內依規則補單
assets:
  - qdevice/Debian
window_start: 2026-10-10T22:20:00+08:00
window_end: 2026-10-10T22:40:00+08:00
silence: 
tags: qdevice, debian, emergency, backfill
---

## 目的

CR-2026-0001 要在 QDevice VM 上以 sudo 執行升級，但這台沒有安裝 sudo，root 密碼也不在任何紀錄裡。Day 15 套用 SSH 加固後只收金鑰、不允許 root 登入。為了執行 CR-2026-0001，先重設 root 密碼並裝 sudo。

## 影響

- 從 Virtualization Station 主控台以 GRUB `init=/bin/bash` 開機重設 root 密碼，VM 重開機兩次。
- 重開機期間 QDevice 不投票，叢集 2/3 票、Quorate 維持 Yes。`PveQdeviceAbsent` 22:32:50 進 pending，22:34:50 到 22:36:35 firing，沒有 silence，因為這件事沒有單子。
- VM 新增 sudo 套件、`/etc/sudoers.d/day26-temp`（ivan 免密碼）、ivan 的第二把公鑰。

## 步驟

照實際發生的順序。

1. 22:20 ivan 在主控台加入執行用的公鑰。
2. 22:31 從主控台重開機，GRUB 加 `init=/bin/bash`，`passwd root`，再開機（22:35:09 開機完成）。
3. 以 root 安裝 sudo，寫 `/etc/sudoers.d/day26-temp`。

## 驗證

- 22:37 起 ivan 以金鑰登入，`sudo -n true` 成功。
- pve1 `check.sh` 22:37 回 0，三票都在。

## 回退

- 刪 `/etc/sudoers.d/day26-temp` 與 ivan 的第二把公鑰。
- sudo 套件可留可移除。root 密碼無法回到舊值，新密碼由使用者保管。

## 紀錄

- begin: 2026-10-10T22:20:00+08:00
- end: 2026-10-10T22:37:00+08:00
- result: 完成，但這是一次沒有單子的變更。它造成的告警就是 Day 26 第三節說的那種：灰色區間之外的異常
- evidence: Prometheus `pve_quorum_qdevice_votes` 22:32:30 到 22:36:15 為 0；`ALERTS{alertname="PveQdeviceAbsent"}` 22:34:50 到 22:36:35 firing；VM `last -x` 的 22:31 shutdown 與 22:35 boot
- followups: CR-2026-0001 結束後收回暫時權限（sudoers 檔與公鑰）
