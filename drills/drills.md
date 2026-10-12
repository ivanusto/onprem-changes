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
| 1 | [CR-2026-0001](../changes/CR-2026-0001-qdevice-debian-13.md) QDevice VM Debian 12 to 13 | Virtualization Station snapshot revert | CR-2026-0011 `qdevice-snapshot-revert` | The record says "within a minute". VM off, revert, power on; `check.sh` on pve1 back to 0 and the marker file written after the snapshot is gone. Quorate stays Yes throughout (2 of 3 votes), `PveQdeviceAbsent` may reach its 2 m `for` and is silenced by the window. | 2026-10-12. rto_s 19 (6 checks). Snapshot `CR-2026-0011-before` taken with the VM running, marker written 19:38:29, VM off; `check.sh` on pve1 went to 1 at 19:39:21 (Quorate Yes, 2/3), drill started 19:39:22, passed 19:39:41 after revert and power on. Much faster than expected because a running-VM snapshot includes memory: the VM resumed instead of booting (`uptime -s` still 19:19:11) and its clock came back 3 min 31 s behind. `check.sh` passed before both qdevice clients reconnected: `Connect timeout` on both nodes until 19:40:34 and 19:40:42, `pve_quorum_qdevice_votes` 0 for one sample (19:40:30 to 19:41:30), `PveQdeviceAbsent` pending at 19:40:45, never firing. Debian 13.7, marker gone. |
| 2 | [CR-2026-0007](../changes/CR-2026-0007-fortigate-event-log-audit-trail.md) 60F syslogd3 to the collector's 515 | Config reversal over REST, then forward again | CR-2026-0011 `fgt-syslogd3-reenable` | The disable event is generated after syslogd3 is already off, so it never reaches the collector; the enable event (`0100044546`, `cfgattr="status[disable->enable]"`) arrives within seconds and `audit-report.py` attributes it to CR-2026-0011. `AuditUnattributedFortigateChange` does not fire. | 2026-10-12. rto_s 8 (5 checks). Disable PUT 19:45:26, enable PUT 19:46:29. The expectation was wrong: the disable event (cfgtid 10289368, `status[enable->disable]`, 19:45:29) did reach the collector, as did the enable event (cfgtid 10289369, 19:46:31), plus `0100022905` Log device status has changed before each. In the 60 s off, the 60F memory log has only four wireless events, which syslogd3's filter excludes anyway, so nothing was lost. Setting read back equal to the backup. |
| 3 | [CR-2026-0007](../changes/CR-2026-0007-fortigate-event-log-audit-trail.md) VictoriaLogs second listener on 515 | `docker-compose.yml.bak-day27` restored, container rebuilt; then the current file restored, rebuilt again | CR-2026-0011 `vlogs-single-listener-restore`, `vlogs-second-listener-forward` | Each rebuild about 10 s (Day 27). After the rollback 515 is closed and the 60F's reliable syslog retries; events generated in the 10 minute hole stay in the 60F memory log only (to verify: FortiOS does not document buffering for reliable syslog). After the forward step every source seen before the rebuild shows new lines within 5 minutes (Day 27: 2 m 30 s). The hole is backfilled from a memory-log export with `fortigate-export-load.py --field audit=fortigate --devname fgt-edge --source backfill`. | 2026-10-12. Rollback rto_s 4 (3 checks), 19:47:14 to 19:47:18; forward rto_s 14 (8 checks), 19:57:22 to 19:57:36. 515 closed for 10 min 18 s; 514 stayed up, so the traffic log kept arriving. During the hole the 60F logged two config changes (policy 4 `comments` set 19:47:33 cfgtid 10289370, cleared 19:47:49 cfgtid 10289371) and two performance statistics events. After 515 came back the collector had all of them except the first one: reliable syslog re-sent what it generated after it noticed the broken connection and lost the one written into it. That one line was backfilled from a memory-log export with `fortigate-export-load.py --field audit=fortigate --devname <devname> --source backfill` (1 line, `source:=backfill` total 1). Every source seen before the rebuild had new lines within 70 s of the forward step; journald sources (journal-upload with a cursor) have the same per-minute counts as the nodes' own journals. Next day's MANIFEST: to check. |

Not drilled this time, on purpose: CR-2026-0004 (switching UTM off on
policy 1 leaves a hole in Day 24's web-filter log that Day 25's weekly
report would show) and CR-2026-0005 (expiring the silence makes twenty
licence alerts fire at once). Both are `risk: high` and both stay on
`ChangeRollbackUndrilled` until a record drills them.
