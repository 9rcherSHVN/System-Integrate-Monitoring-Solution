# Disk and ERP Log Capacity Response Runbook

## Trigger Conditions

- Disk capacity warning or critical alert.
- Sustained disk read/write latency alert.
- ERP log-folder growth alert.
- ERP log retention accumulation alert.
- ERP folder collector failure or missing telemetry alert.

## Initial Triage

1. Open the **ERP Disk and Log Folder Monitoring** Workbook.
2. Identify:
   - Server.
   - Drive letter.
   - Available free percentage and free GB.
   - ERP log folder size.
   - 24-hour folder growth.
   - Old-file count and retention threshold.
   - Disk read/write latency.
3. Confirm whether the affected drive hosts:
   - ERP application logs.
   - IIS logs.
   - Middleware queues.
   - SQL data/log/tempdb.
   - Backup files.
4. Check active incident/change records before removing any files.
5. Confirm whether the growth is expected, such as during batch processing, upgrades, or diagnostics.

## Immediate Action for Critical Disk Capacity

1. Preserve logs required for incident investigation, audit, or legal hold.
2. Stop uncontrolled log generation only through approved application/vendor procedures.
3. Archive or remove only data approved by retention policy.
4. Confirm log rotation and cleanup jobs are operational.
5. Extend storage if approved and necessary.
6. Re-run the folder collector.
7. Verify that the disk-capacity alert resolves.

## High Disk Latency Investigation

1. Correlate with ERP response-time, CPU, memory, backup, antivirus, and storage events.
2. Identify competing processes or backup/maintenance windows.
3. Confirm storage subsystem health with infrastructure/storage support.
4. For SQL-related volumes, correlate with SQL I/O stall and wait statistics.
5. Do not restart ERP or storage services without approved incident/change procedure.

## Evidence Required

- Server and drive.
- Before/after free GB and free percentage.
- ERP folder growth rate.
- Cleanup/archive action.
- Change or incident reference.
- Root cause and prevention action.
