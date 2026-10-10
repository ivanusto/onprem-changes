---
id: CR-2026-0001
title: 把 QDevice VM 從 Debian 12 升到 13
status: done
type: standard
risk: medium
requester: ivan
implementer: ivan
approver: ivan
approval_exception: 單人實驗場域，沒有第二位核准者。以 PR 自審加 24 小時冷卻期取代，PR 開啟時間與合併時間在 git log。這一張的冷卻沒有滿 24 小時，因為它同時是 change-mark.sh 的第一次實測，核准與執行在同一晚，PR 由擁有者以管理員略過合併
assets:
  - qdevice/Debian
window_start: 2026-10-10T22:45:00+08:00
window_end: 2026-10-10T23:45:00+08:00
silence: alertname=PveQdeviceAbsent
tags: pve, qdevice, debian, eol
---

## 目的

Day 25 的 `AssetEolPassed` 在響：`qdevice/Debian` 列的 `security_support_end` 是 2026-07-11，已過 90 天以上。這台 VM 跑 `corosync-qnetd`，是 Proxmox VE 雙節點叢集的第三票（Day 15）。Debian 12 的一般安全支援已結束，LTS 到 2028-06-30，但 LTS 不涵蓋所有套件，而這台的工作只有一個套件，升到 13 的成本最低。升完把 `eol/assets.tsv` 的兩列改成 Debian 13 的日期，告警自己消。

## 影響

- 升級中 `corosync-qnetd` 會重啟一次，重開機時停約 2 到 3 分鐘（Day 15 演練 4 量到 qnetd 從關到回來約 6 分鐘，其中 VM 開機佔大半）。期間叢集 2/3 票，Quorate 維持 Yes，`check.sh` 回 1，Day 20 的 `PveQdeviceAbsent` 會響，用 `silence` 欄的 matcher 在窗內靜音。
- 窗內若有一台節點同時故障，就是 Day 15 演練 3 的情境，另一台會失去 quorum 並自我隔離。窗要短，而且不跟任何節點或 NAS 的維護重疊。
- 兩台 PVE、NAS、收集端都不改。`pvecm qdevice` 的憑證在 VM 的 `/etc/corosync/qnetd/` 裡，就地升級會保留，不用重做 `pvecm qdevice setup`。

## 步驟

在 pve1 上：

```sh
logger -t change -p notice "CR-2026-0001 begin"
/root/pve-qdevice-on-qnap/check.sh                    # 預期 0，三票都在
```

在收集端上：

```sh
sudo -u metrics /opt/onprem-changes/scripts/change-mark.sh begin CR-2026-0001   # journal、Grafana 註記、Alertmanager silence
```

在 NAS 的 Virtualization Station 上，對 QDevice VM 拍一個快照，名稱 `CR-2026-0001-before`。Day 15 說這台不需要快照，那是對日常。對一次就地升級，一個能在一分鐘內回到升級前的快照是最便宜的回退。

在 QDevice VM 上（以金鑰登入，ivan 經 sudo）：

```sh
sudo -i
cat /etc/debian_version                               # 12.x
apt update && apt full-upgrade -y && apt autoremove -y   # 先把 12 補到最新
sed -i 's/bookworm/trixie/g' /etc/apt/sources.list   # 這台只有 sources.list，sources.list.d 是空的
grep -r trixie /etc/apt/sources.list /etc/apt/sources.list.d/           # 確認沒有漏掉的 bookworm
apt update
apt upgrade --without-new-pkgs -y                     # Debian 發行說明的兩階段升級
apt full-upgrade -y
apt autoremove -y
systemctl reboot
```

重開機後，在 QDevice VM 上：

```sh
cat /etc/debian_version                               # 13.x
systemctl is-active corosync-qnetd nftables           # 都是 active
ss -tlnp | grep 5403
apt modernize-sources                                 # 13 的 apt 3.0 提供，把 sources.list 轉成 deb822（待查，可選）
```

在 pve1 上：

```sh
/root/pve-qdevice-on-qnap/check.sh                    # 回到 0
pvecm status | grep -A3 Qdevice
logger -t change -p notice "CR-2026-0001 end"
```

在收集端上：

```sh
sudo -u metrics /opt/onprem-changes/scripts/change-mark.sh end CR-2026-0001
```

在 Virtualization Station 上，驗證通過 24 小時後刪除快照 `CR-2026-0001-before`。

## 驗證

- `check.sh` 在 pve1 上回 0，`pvecm status` 的 Qdevice 列有一票。
- Day 20 的 `pve_quorum_qdevice_votes` 回到 1，`PveQdeviceAbsent` 不再 pending。
- 收集端查 `_time:[2026-10-10T22:45:00+08:00, 2026-10-10T23:45:00+08:00) "CR-2026-0001"`，看得到 pve1 與收集端的 begin 與 end 四行。
- 從收集端以外的任何主機 `ssh -V` 風格探一下 VM 的 SSH 版本字串，應為 `deb13`（Day 25 用這個方法確認過它是 `deb12u10`）。
- 後續：onprem-metrics `eol/assets.tsv` 的 `qdevice/Debian` 兩列改成 13（`security_support_end` 2028-08-09，`lts_end` 2030-06-30，`eol_ref` debian/13），commit 訊息帶 `CR-2026-0001`，隔天 `AssetEolPassed` 消失。

## 回退

- 升級中斷或重開機後 `corosync-qnetd` 起不來：在 Virtualization Station 把 VM 回復到快照 `CR-2026-0001-before`，開機，pve1 上 `check.sh` 回 0。一分鐘內。
- 快照也壞：照 [qnetd-vm.md](https://github.com/ivanusto/pve-qdevice-on-qnap/blob/main/qnetd-vm.md) 重建（Day 15 寫的是十分鐘），重建的 qnetd 憑證是新的，要在 pve1 上 `pvecm qdevice remove` 再 `pvecm qdevice setup <ip>`。
- 窗內節點故障：不做回退，先處理節點，QDevice 的升級等下一個窗。叢集的行為見 Day 15 演練 3。

## 紀錄

- begin: 2026-10-10T22:45:04+08:00（pve1 的 logger），收集端 `change-mark.sh begin` 22:45:06，Grafana 註記 2，silence 751e35fd 到 23:45
- end: 2026-10-10T22:55:07+08:00（pve1 與收集端），註記 2 補上 timeEnd，silence 失效
- result: 成功。Debian 12.15 升到 13.7，核心 6.1.0-53 到 6.12.111+deb13，corosync-qnetd 3.0.3-1 到 3.0.3-2，openssh-server 9.2p1-2+deb12u10 到 10.0p1-7+deb13u4（sshd_config 保留，root 與密碼登入仍關）。apt 從 22:46:37 跑到 22:53:32。重開機時 QDevice 缺席約 30 秒（pve1 `check.sh` 22:54:07 回 1，22:54:36 回 0），`PveQdeviceAbsent` 的 `for: 2m` 沒到，silence 沒有實際壓下任何告警。`apt modernize-sources` 已做，sources.list 轉成 `sources.list.d/debian.sources`，舊檔留 `.bak`
- evidence: VictoriaLogs `_time:[2026-10-10T22:45:00+08:00, 2026-10-10T23:45:00+08:00) SYSLOG_IDENTIFIER:change "CR-2026-0001"` 四行；不加 `SYSLOG_IDENTIFIER` 是九行，多出收集端 sudo 的三行與 NAS 事件日誌裡 Virtualization Station 建立快照 `CR-2026-0001-before` 的兩行（22:46:30、22:46:44）。Grafana 註記 2（tags change、CR-2026-0001）。VM 上 `/root/cr0001-upgrade.log`。SSH 橫幅 `OpenSSH_10.0p2 Debian-7+deb13u4`
- followups: 2026-10-11 22:55 後刪快照 `CR-2026-0001-before`；onprem-metrics `eol/assets.tsv` 的 qdevice 兩列改成 13，commit 帶本單編號；收回執行用的暫時權限（VM 上 ivan 的第二把公鑰、`/etc/sudoers.d/day26-temp`）；22:31 到 22:35 為了取得 sudo 重設 root 密碼的重開機在窗外、沒有單子，`PveQdeviceAbsent` 22:34:50 到 22:36:35 firing，補 CR-2026-0006（emergency）
