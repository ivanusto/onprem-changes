---
id: CR-2026-0016
title: 收集端的 onprem-agent-gate 換成 v0.1.0 tag 版，PENDING 跟著單子的窗與狀態，結案後的等待核准告警自己消失
status: done
type: standard
risk: low
requester: ivan
implementer: ivan
approver: ivan
approval_exception: 單人實驗場域，沒有第二位核准者。以 PR 自審取代，PR 開啟與合併時間在 git log。冷卻未滿 24 小時，由擁有者以管理員略過合併
agent:
assets:
  - collector/Ubuntu
window_start: 2026-10-13T21:30:00+08:00
window_end: 2026-10-13T22:00:00+08:00
silence: alertname=AgentGateApprovalWaiting
tags: agent, gate, followup, day29
---

## 目的

CR-2026-0015 的 followup (1)。演練第 3 步刻意留下的 seq 19 是一個沒有人核准的 PENDING。收集端跑的版本（CR-2026-0015 部署的那一版，內容等於 repo 的 f8857ed）的 `pending()` 只看有沒有被核准、不看單子，單子結案之後 `agent_gate_pending_approvals` 仍是 1，`AgentGateApprovalWaiting` 從 2026-10-13 20:03 起一直 firing。onprem-agent-gate v0.1.0（d123e11）改成：單子不是 in_progress、不在窗內或沒有 begin 標記時，PENDING 不計入、也不能被核准。

## 影響

只換收集端 `/opt/onprem-agent-gate` 的 `gate/core.py` 與 `bin/gatectl`。`tools.json`、`config.json`、金鑰、sudoers、cron、稽核鏈都不動。`agent:` 不寫，本單沒有任何代理程式可以呼叫閘道（CR-2026-0015 已結案，閘道目前對所有呼叫都是 DENY，因為沒有任何一張 live 的單子寫 `agent:`）。

換檔期間沒有人在用閘道。`AgentGateApprovalWaiting` 預期在下一次 textfile（每 5 分鐘）之後轉為 inactive，silence 只是保險。

## 步驟

在工作站：

```sh
git -C ~/onprem-agent-gate archive v0.1.0 bin gate | ssh ivan@192.168.2.49 'mkdir -p ~/cr0016 && tar -x -C ~/cr0016'
```

在收集端：

```sh
sudo -u metrics /opt/onprem-changes/scripts/change-mark.sh begin CR-2026-0016
sudo cp -n -p /opt/onprem-agent-gate/gate/core.py /opt/onprem-agent-gate/gate/core.py.bak-cr0016
sudo cp -n -p /opt/onprem-agent-gate/bin/gatectl /opt/onprem-agent-gate/bin/gatectl.bak-cr0016
diff -r ~/cr0016/gate /opt/onprem-agent-gate/gate; diff ~/cr0016/bin/gatectl /opt/onprem-agent-gate/bin/gatectl   # 只有 pending 與 approve 的差異
sudo install -m 0644 ~/cr0016/gate/core.py /opt/onprem-agent-gate/gate/core.py
sudo install -m 0755 ~/cr0016/bin/gatectl /opt/onprem-agent-gate/bin/gatectl
sudo -u agent-gate gatectl pending
sudo -u agent-gate gatectl textfile --out /var/lib/node_exporter/textfile/agent-gate.prom
sudo -u metrics /opt/onprem-changes/scripts/change-mark.sh end CR-2026-0016
```

## 驗證

- `gatectl pending` 回 `no pending requests`（seq 19 屬於已結案的 CR-2026-0015）。
- `agent-gate.prom` 的 `agent_gate_pending_approvals 0`，Prometheus 的 `AgentGateApprovalWaiting` 在 5 分鐘內轉為 inactive。
- `gatectl verify --vl http://127.0.0.1:9428` 仍是 `chain: ok`，筆數不變（換程式不寫稽核）。

## 回退

`.bak-cr0016` 兩個檔覆回即可，沒有資料流受影響。回退後 `AgentGateApprovalWaiting` 會回到 firing，直到另一個修正。

## 紀錄

- begin: 2026-10-13T21:30:02+08:00（Grafana 註記 10，silence 41d281f7 到 22:00）
- end: 2026-10-13T21:31:23+08:00，註記 10 補上 timeEnd，silence 失效
- result: 完成。`gate/core.py` 與 `bin/gatectl` 換成 v0.1.0（d123e11），舊檔 `*.bak-cr0016`；diff 只有 `pending()`、`approve()` 與 `open_for_changes()` 以及 `gatectl pending` 的提示
- evidence: 21:30:12 `gatectl pending` 回 `no pending requests`，`agent_gate_pending_approvals 0`；`gatectl verify --vl` `chain: ok (27 records)`，VictoriaLogs 27 行，0 筆缺行、0 筆 hash 不符；`AgentGateApprovalWaiting` 21:31:20 起不再出現在 Prometheus 的 alerts（20:03 起 firing）
- followups: 無
