---
id: CR-2026-0012
title: 恢復 hermes 的 Telegram 通知與封面生成、恢復 pbs-nas，並在每日摘要加入 hermes 監控（未經單子，事後補單）
status: done
type: emergency
risk: medium
requester: ivan
implementer: ivan
approver: ivan
approval_exception: 單人實驗場域，沒有第二位核准者。三件事都是排查告警與故障時順手做下去的，當時沒有開單，24 小時內依規則補單
assets:
  - nas-primary/QuTS hero
  - pve1/Proxmox VE
  - pve2/Proxmox VE
  - spark01/DGX OS
window_start: 2026-10-12T14:29:00+08:00
window_end: 2026-10-12T17:16:00+08:00
silence: 
tags: hermes, pbs, ops-digest, emergency, backfill
---

## 目的

10/11 晚上主 NAS 重開後，有三件事壞了，第三件更早就壞了：

- hermes 的 Telegram 通知 10/11 19:14 起是 `fatal`（`adapter_unavailable`）。容器重建後缺 `python-telegram-bot`，啟動時的自動安裝被 uv 在 ZFS 上的 reflink 失敗（`os error 95`）擋下。10/12 的 `hn-daily-blog-post` 與 `github-trends-weekly` 投遞失敗，沒有人知道。
- hermes 的自動發文封面生成失敗。腳本寫死 `http://comfyui:8188`，ComfyUI 與 A2000 已移到 TVS-h877AX（192.168.2.139）。10/12 早上兩篇文章發佈時沒有特色圖片。
- `PveStorageUnavailable` 從 Day 20 部署起就在響。`pbs-nas` 指向主 NAS 上 JustBackup QPKG 的 PBS 容器，PVE 至少從 9/25 起就連不上。

另外，在每日摘要（ops-digest）加入 hermes 的排程與通知狀態，讓前兩件事下次在隔天早上就會被看到。這一項本身不急，但跟前兩件在同一段時間做完，一併記在這裡。

## 影響

- hermes 容器在 15:32 重啟一次，Telegram 與 API 中斷約 20 秒。
- TVS-h877AX 的 ComfyUI 在 14:34 到 14:38 跑了兩次測試生成，各約 2 分鐘佔用 GPU。TVS-h877AX 不在 `eol/assets.tsv`，所以不列在上面的 `assets`。
- JustBackup 的兩個容器原本都不在，16:11 起回後主程式（28899）恢復。PBS 容器在修權限後重啟一次。
- `/etc/pve/storage.cfg` 是叢集共用設定，17:03 與 17:09 各改一次。之後到 17:11 之間，兩台 PVE 的 pvestatd 持續記錄 `pbs-nas` 找不到 datastore，沒有其他儲存受影響。
- ops-digest 在 15:31 與 15:50 各手動跑一次，兩次都覆寫了當天的 `ops-digest-2026-10-12.md`。
- 17:03 到 17:15 的動作落在 CR-2026-0008 的窗（17:00 到 18:00）裡，Day 27 的稽核報告會把它們歸到 CR-2026-0008。實際上那段時間的 PVE 與主 NAS 動作屬於本單，見紀錄。
- 沒有 silence。`PveStorageUnavailable` 本來就在響，修好後 17:12:44 解除。

## 步驟

照實際發生的順序，時間為 +08:00。

1. 14:29 到 14:31，從 spark01 經兩個 NFS 掛載，把封面工作流程的四個模型（Krea2 Turbo nvfp4、Qwen3-VL 4B fp8、Qwen Image VAE、krea2_darkbrush LoRA，約 13.6 GB）從主 NAS 的 `Public/models` 複製到 TVS-h877AX 的 `Public/models`，兩邊在 NAS 上各算一次 SHA-256 比對。
2. 14:33:42，hermes 的四支腳本（`hn_daily_post.py`、`cyberq_automation.py`、`github_trends_weekly.py`、`pcwatch_post.py`）把 `COMFYUI_API_URL` 改成 `http://192.168.2.139:8188`。原檔留 `*.bak-20261012-comfyui`，檔案擁有者維持 uid 10000。
3. 14:34 到 14:38，在 TVS-h877AX 用同一個工作流程測試生成兩張封面。
4. 14:41，ops-digest 的 `collect.py`、`publish.py`、`prompt/SYSTEM.md` 加入 hermes 段落（本機 repo 925a875），15:31 前由使用者執行 `install.sh` 安裝。
5. 約 15:20，使用者在主 NAS 兩次執行 `hermes pm install --extra telegram`（第二次加 `-e UV_LINK_MODE=copy`），都以 `os error 95` 失敗。hermes PM 會濾掉 `UV_LINK_MODE`，失敗的暫存環境由 hermes 自行清除。
6. 約 15:31，使用者改用 hermes 內附的 uv 0.12.3，帶 `UV_LINK_MODE=copy`，把 `python-telegram-bot[webhooks]==22.8`（連同 `tornado 6.5.10`）裝進執行中的環境。15:32 重啟 `hermes-agent`。
7. 15:49:57，手動跑一次 ops-digest 驗證 hermes 段落。
8. 16:11，主 NAS 上備份 JustBackup 的 `justbackup.sh`（`.bak-20261012`），換成會等 docker 就緒的版本，執行 `start`。兩個容器起來，但 PBS 的 proxy 讀不到 `authkey.pub`（v0.5.1 安裝程式的 `chown -R 0:0` 造成）而崩潰。
9. 16 點後段，在 PBS 容器內把 `authkey.pub`、`csrf.key`、`proxy.key`、`proxy.pem`、`acl.cfg`、`user.cfg`、`domains.cfg` 改回 `root:backup 640`，`shadow.json` 改 600，`/var/lib/proxmox-backup` 與 `/var/log/proxmox-backup` 改回 `backup:backup`，重啟容器。8007 恢復。
10. 17:03:37，在 cyberq-i5 備份 `/etc/pve/storage.cfg` 到 `/root/storage.cfg.bak-20261012-pbs`，以 `pvesm set pbs-nas --fingerprint` 換成 9/23 重建的 PBS 的憑證指紋。
11. 17:0x，datastore 目錄權限改成 PBS 要求的值：`/share/JustBackup/pbs` 755，`.chunks` 與其下 65,536 個子目錄 750，`.lock` 644，擁有者都是 `backup:backup`。
12. 17:09:43，以 `pvesm set pbs-nas --password` 換成 PBS 目前的 root 密碼。
13. 約 17:11，`proxmox-backup-manager datastore create main /backups --reuse-datastore true`，沿用既有的 chunk store。
14. 17:15:11，主 NAS 的 `justbackup.sh` 換成 just_backup_qnap c06f121 的版本（啟動前修 PBS 權限、等 API、自動 reuse）。

## 驗證

- 兩台 PVE 的 `pvesm status`：`pbs-nas` 為 active。`pvesm list pbs-nas` 列出 VM 100 三版（7/24 兩版、8/9 一版）與 CT 101 七版（8/9），與目錄內容一致。
- Prometheus `pve_up{id=~"storage/.*/pbs-nas"}` 兩台都是 1；`ALERTS{alertname="PveStorageUnavailable"}` 17:12:44 起為 0，Alertmanager 的 active 也是 0。
- hermes 的 `gateway_state.json`：telegram 與 api_server 都是 connected。gateway 日誌 15:32:46 `✓ telegram connected`。
- TVS-h877AX 的 ComfyUI 回應 `/system_stats`，看得到新放的模型。兩次測試生成都成功：冷 150 秒、暖 123 秒，都在腳本的 300 秒逾時內。
- ops-digest 15:50 那份的附錄有「hermes 排程與通知」一段，正確列出當天兩筆投遞失敗。

## 回退

- 封面：`*.bak-20261012-comfyui` 覆回四支腳本。不過舊位址指向的 comfyui 容器已經不在主 NAS 上，回退等於回到 10/12 早上沒有封面的狀態。TVS-h877AX 上的四個模型可以直接刪除。
- Telegram：以同一個 uv 執行 `pip uninstall python-telegram-bot tornado`，再重啟 `hermes-agent`。
- ops-digest：`git revert 925a875` 後重跑 `install.sh`。
- JustBackup：`justbackup.sh.bak-20261012` 覆回，`justbackup.sh stop` 停掉兩個容器。PBS 權限不回退，因為回退只會讓 proxy 再度崩潰。
- pbs-nas：`/root/storage.cfg.bak-20261012-pbs` 覆回 `/etc/pve/storage.cfg`。舊密碼沒有備份，但它在 9/23 PBS 重建後就已經失效。datastore 可以 `proxmox-backup-manager datastore remove main` 移除設定，資料不會被刪。

## 紀錄

- begin: 2026-10-12T14:29:00+08:00
- end: 2026-10-12T17:16:00+08:00
- result: 完成，但這是一次沒有單子的變更。三件事都已恢復，`PveStorageUnavailable` 17:12:44 解除
- evidence:
  - TVS-h877AX `Public/models` 四個檔的 mtime 14:30:43 到 14:31:07，SHA-256 兩邊一致；hermes 四支腳本 mtime 14:33:42。
  - ops-digest 的 systemd 紀錄：15:31:46 與 15:49:57 各一次 collect，publish 都是 `md5 ok; llm ok`。
  - hermes `gateway.log` 15:32:46 `✓ telegram connected`；`errors.log` 00:01:34 與 00:16:01 的 `python-telegram-bot not installed` 是修好前的兩筆投遞失敗。
  - cyberq-i5：`/root/storage.cfg.bak-20261012-pbs` mtime 17:03:37，`/etc/pve/storage.cfg` 與 `priv/storage/pbs-nas.pw` mtime 17:09:43；pvestatd 最後一筆 `Cannot find datastore 'main'` 在 17:10:52。
  - 主 NAS：`justbackup.sh.bak-20261012` 16:11:24，現行 `justbackup.sh` 17:15:11。
  - Prometheus `ALERTS{alertname="PveStorageUnavailable"}` 17:12:44 起為 0。
- followups:
  - PBS root 密碼在 17 點前曾以明文出現在一段工作階段紀錄中，另開單更換：改 `pbs_root_password`、重啟 JustBackup、`pvesm set pbs-nas --password`。
  - hermes 更新映像後會建新的環境，Telegram 套件會再掉，自動安裝也會再被同一個問題擋下。上游已回報 NousResearch/hermes-agent#137036，社群有 draft PR #137040。在那之前，更新後要照步驟 6 重裝。
  - just_backup_qnap c06f121 只在本機，沒有 push、沒有發版。
  - 9/23 之後 PBS 的 `datastore.cfg` 為什麼不見，還沒有查到原因。
  - 17:03 到 17:15 的動作在稽核報告裡被歸到 CR-2026-0008。
  - TVS-h877AX 現在承擔正式的出圖工作，可以考慮加進 `eol/assets.tsv`。
