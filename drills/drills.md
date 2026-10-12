# Rollback drills

A rollback section that has never been run is a guess. Each row below is
one rollback section of a done record, performed on purpose inside a
drill record's window, timed with [drill.sh](drill.sh) from the moment
the rollback starts to the first passing verification. The format is the
one pve-qdevice-on-qnap's `drills.md` (Day 15) uses: what we expected,
then what was observed, with times. The machine-written line is in
[drills.jsonl](drills.jsonl); corrections made here by hand do not flow
back into it.

Run a drill when a rollback section is written for a mechanism no record
has rolled back yet (snapshot revert, config reversal, file restore plus
container rebuild, ...), and again after the thing it relies on changes
(the snapshot tool, the compose file, the API). `ChangeRollbackUndrilled`
(onprem-metrics) lists done high-risk records with no drill on file;
`ChangeRollbackDrillStale` lists drills older than 90 days.

| # | Rollback of | Mechanism | Drill record | Expected | Observed (date, rto_s, notes) |
|---|---|---|---|---|---|
| 1 | [CR-2026-0001](../changes/CR-2026-0001-qdevice-debian-13.md) QDevice VM Debian 12 to 13 | Virtualization Station snapshot revert | CR-2026-0011 `qdevice-snapshot-revert` | The record says "within a minute". VM off, revert, power on; `check.sh` on pve1 back to 0 and the marker file written after the snapshot is gone. Quorate stays Yes throughout (2 of 3 votes), `PveQdeviceAbsent` may reach its 2 m `for` and is silenced by the window. | 2026-10-12. 【待填：rto_s、VM 關機到開機各時間點、check.sh 1 到 0 的時間、PveQdeviceAbsent 是否 pending/firing、NAS 事件日誌的五行】 |
| 2 | [CR-2026-0007](../changes/CR-2026-0007-fortigate-event-log-audit-trail.md) 60F syslogd3 to the collector's 515 | Config reversal over REST, then forward again | CR-2026-0011 `fgt-syslogd3-reenable` | The disable event is generated after syslogd3 is already off, so it never reaches the collector; the enable event (`0100044546`, `cfgattr="status[disable->enable]"`) arrives within seconds and `audit-report.py` attributes it to CR-2026-0011. `AuditUnattributedFortigateChange` does not fire. | 2026-10-12. 【待填：rto_s、兩筆的 eventtime 與收集端 _time、記憶體日誌裡停用那一筆的 logid 與 cfgtid】 |
| 3 | [CR-2026-0007](../changes/CR-2026-0007-fortigate-event-log-audit-trail.md) VictoriaLogs second listener on 515 | `docker-compose.yml.bak-day27` restored, container rebuilt; then the current file restored, rebuilt again | CR-2026-0011 `vlogs-single-listener-restore`, `vlogs-second-listener-forward` | Each rebuild about 10 s (Day 27). After the rollback 515 is closed and the 60F's reliable syslog retries; events generated in the 10 minute hole stay in the 60F memory log only (to verify: FortiOS does not document buffering for reliable syslog). After the forward step every source seen before the rebuild shows new lines within 5 minutes (Day 27: 2 m 30 s). The hole is backfilled from a memory-log export with `fortigate-export-load.py --field audit=fortigate --devname fgt-edge --source backfill`. | 2026-10-12. 【待填：兩個 rto_s、每個來源重連的時間、洞裡的那一筆 cfgtid 在補載前後的查詢結果、隔天 MANIFEST 的行數】 |

Not drilled this time, on purpose: CR-2026-0004 (switching UTM off on
policy 1 leaves a hole in Day 24's web-filter log that Day 25's weekly
report would show) and CR-2026-0005 (expiring the silence makes twenty
licence alerts fire at once). Both are `risk: high` and both stay on
`ChangeRollbackUndrilled` until a record drills them.
