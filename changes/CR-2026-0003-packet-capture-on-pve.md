---
id: CR-2026-0003
title: 在兩台 PVE 節點裝封包擷取，收集端加拉取與驗證（Day 23）
status: done
type: standard
risk: medium
requester: ivan
implementer: ivan
approver: ivan
approval_exception: 單人實驗場域，沒有第二位核准者。這是 Day 26 之前做的變更，2026-10-10 依當時紀錄補單，當時沒有事前核准
assets:
  - pve1/Proxmox VE
  - pve2/Proxmox VE
  - collector/Ubuntu
window_start: 2026-10-07T01:41:00+08:00
window_end: 2026-10-07T09:55:00+08:00
silence: 
tags: pcap, worm, pve, backfill
---

## 目的

Day 23 要在事件發生時有封包可看，又不讓擷取檔留在節點上被竄改。節點用 `pcap@<profile>` 依需要擷取，收集端定時拉走寫進 WORM 並驗證。

## 影響

- pve1、pve2：新增使用者 `pcap`（無密碼、只收收集端的拉取金鑰）、群組 `pcap-ops`、`pcap@.service`、`/var/lib/pcap`。不擷取時不佔資源。
- 收集端：拉取金鑰、`/var/lib/onprem-pcap`、cron（:40 拉、:55 驗）。WORM 裡寫入的批次鎖定 180 天，無法回退。
- 兩台 DGX Spark 的 `pcap@.service` 由使用者自行更新（10:46）。

## 步驟

1. 01:41 收集端建拉取金鑰與目錄，WORM 建 `pcap/`。
2. 01:42 pve1、pve2 跑 `install-pcap.sh --pull-key`。
3. 01:43 pve1 `systemctl start pcap@corosync`，01:45 以暫時 timer 在 01:53 停止。
4. 08:35 第一批 pve1 寫入 WORM，verify OK。
5. 09:36 換上加了 `SuccessExitStatus=124` 的 unit，daemon-reload。
6. 09:39 收集端 cron 加拉取與驗證，09:55 第一次自動驗證兩批皆 OK。

## 驗證

- VictoriaLogs：pve1 的 `Started pcap@corosync.service` 在 2026-10-07 01:43:24。
- WORM 批次 `sha256sum -c` OK，`rm` 回 Operation not permitted（09:12）。
- drills.jsonl 有 09:55 的驗證紀錄。

## 回退

- 節點：停用並刪 `pcap@.service`、使用者 `pcap`、`/var/lib/pcap`、`/etc/pcap`。
- 收集端：cron 以備份覆回，刪拉取金鑰與 `/var/lib/onprem-pcap`。
- WORM 裡已鎖定的批次與金絲雀檔無法回退，保留期滿自動到期（已同意）。

## 紀錄

- begin: 2026-10-07T01:41:00+08:00
- end: 2026-10-07T09:55:00+08:00（第一次自動驗證）
- result: 成功，有一次失誤：08:37 在鎖定生效前做刪除測試，批次被刪，08:38 以已驗雜湊的本機副本還原。當時沒有單子，時間取自 Day 23 的變更表
- evidence: VictoriaLogs pve1 `pcap@corosync` 01:43:24；WORM 第一批 `pve1/20261007T003534Z`；drills.jsonl
- followups: 交換器埠鏡射實抓尚未做
