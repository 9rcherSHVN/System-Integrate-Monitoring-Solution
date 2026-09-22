# Update: Disk Drive Space Monitoring & Tracking Added to Pilot Scope

Good addition — and unlike the network/port and DMV pieces, this one is **natively well-supported** by Azure Monitor Agent, so it's low-effort to add but I'll also flag where it needs to be *purpose-built* for your ERP/SQL context rather than left as generic OS monitoring.

---

## PART D — Disk Drive Space Monitoring & Tracking

### D.1 Architectural Fit
This slots directly into the **VM Insights / AMA layer** already defined in Part A's foundation (Arc-enabled server → AMA → Data Collection Rule → LAW) — no new agent or ingestion pipeline required. This is the cleanest, most "native Azure" piece of the entire pilot.

However, **generic disk-% monitoring is insufficient for your workload profile**. Three server roles have very different disk-risk patterns, so the DCR and alert thresholds need to be **role-aware**, not one-size-fits-all:

| Server Role | Disk Risk Profile | What Matters Most |
|---|---|---|
| **SQL Server** | Data/log file growth, tempdb growth, backup file accumulation | Per-**drive/volume** free space AND per-**database file** growth trend (data file, log file, tempdb separately) |
| **ERP App Server** | Log file bloat (app logs, IIS logs), temp file accumulation | Free space on log/app volumes, log directory growth rate |
| **Middleware Server** | Queue/message store growth, transaction logs, cache spillover | Free space on message-store volumes |

### D.2 Data Collection Design

**Standard layer — via AMA Data Collection Rule (built-in, no custom scripting needed):**

Add these **Windows Performance Counters** to the existing DCR's Performance Counter data source:

| Counter | Purpose |
|---|---|
| `LogicalDisk(*)\% Free Space` | Core free-space-remaining metric per volume |
| `LogicalDisk(*)\Free Megabytes` | Absolute free space (more actionable than % on large volumes) |
| `LogicalDisk(*)\Disk Transfers/sec`, `\Avg. Disk sec/Read`, `\Avg. Disk sec/Write` | Disk I/O latency/saturation — correlates with SQL I/O stall data from Part B |
| `LogicalDisk(*)\% Disk Time` | Overall disk load |

Sampling frequency: recommend 5-min interval (aligns with SQL DMV custom collection cadence for correlation purposes).

**Extended layer — custom collection for SQL-specific file growth (ties into Part B's DCR):**

Generic `LogicalDisk` counters tell you a volume is filling up, but not *why* — for SQL Server, you need file-level granularity:

```sql
SELECT 
    GETUTCDATE() AS TimeGenerated,
    DB_NAME(mf.database_id) AS DatabaseName,
    mf.type_desc AS FileType,        -- ROWS, LOG, or TEMP
    mf.physical_name,
    mf.size * 8 / 1024 AS SizeMB,
    mf.growth, mf.is_percent_growth,
    fs.num_of_bytes_read/1024/1024 AS ReadMB,
    fs.num_of_bytes_written/1024/1024 AS WrittenMB
FROM sys.master_files mf
JOIN sys.dm_io_virtual_file_stats(NULL, NULL) fs 
    ON mf.database_id = fs.database_id AND mf.file_id = fs.file_id
```

- Add this as a fourth table alongside the Part B DMV tables: `SQL_FileGrowth_CL`
- Same DCE/DCR/Logs Ingestion API pipeline already built for SQL DMVs — **no new infrastructure**, just an additional query + table in the existing pilot pattern
- This directly answers the architect-level question: *"Volume D:\ is at 90% — is it tempdb runaway growth, transaction log not truncating, or backup file accumulation?"*

**Extended layer — ERP/Middleware log directory tracking (lightweight custom):**

Reuse the same custom-collector pattern from Part A/C (PowerShell via Hybrid Runbook Worker or AMA custom log data source):
```powershell
Get-ChildItem -Path "D:\ERP\Logs" -Recurse -File | 
Measure-Object -Property Length -Sum |
Select-Object @{n='FolderSizeMB';e={$_.Sum/1MB}}
```
Feeds into `ERP_LogFolderGrowth_CL` — useful for catching runaway logging (common ERP issue) before it consumes the whole volume.

### D.3 Alerting Design (role-aware thresholds)

Rather than one generic "disk < 10% free" alert, define **tiered, role-specific alert rules**:

```kql
// Tier 1 - Warning
InsightsMetrics
| where Namespace == "LogicalDisk" and Name == "FreeSpacePercentage"
| where Val < 20
| summarize AggregatedValue = avg(Val) by Computer, bin(TimeGenerated, 5m)

// Tier 2 - Critical (shorter eval window, faster page)
InsightsMetrics
| where Namespace == "LogicalDisk" and Name == "FreeSpacePercentage"
| where Val < 10
```

- **Warning tier (< 20% free)** → Teams notification + ServiceNow low-priority ticket (planning/capacity action)
- **Critical tier (< 10% free)** → ServiceNow high-priority incident + Teams + Outlook page (immediate action, outage risk)
- **SQL-specific rule**: alert on `SQL_FileGrowth_CL` when log file growth rate exceeds a threshold in a short window (catches "log file growing 500MB in 5 minutes" scenarios *before* the volume actually fills — more proactive than free-space % alone)

### D.4 Dashboard Integration
Add a **Disk & Storage** tab to the existing Azure Workbook (from Part A/B design), with:
- Fleet-wide heat map of free space % by server/volume
- SQL database file growth trend lines (data/log/tempdb separated)
- Top 10 fastest-growing volumes/log folders (trend, not just snapshot — catches slow leaks before they become emergencies)

---

## Updated Consolidated Pilot Scope

| Component | Pilot Action |
|---|---|
| Network/Port (continuous) | Connection Monitor, 3 test groups |
| REST API (continuous) | Availability Test, private on-prem test point |
| Network/Port (on-demand) | Custom DCR + PowerShell snapshot, on-demand runbook trigger |
| SQL DMV | Custom DCR: SessionActivity, WaitStats, IOLatency |
| **Disk space (NEW)** | **Standard AMA LogicalDisk counters (all 3 servers) + custom SQL file-growth DCR + ERP log-folder growth collector** |
| Credential mgmt | Windows LAPS pilot + service-account expiry runbook |
| Certificate mgmt | Cert inventory runbook + expiry alerting |
| Alerting/Ticketing | Unified Action Group → ServiceNow ITSM Connector + Teams + Outlook |

This keeps disk monitoring **architecturally consistent** with the rest of the pilot — reusing the same AMA/DCR/LAW pipeline for the standard OS-level metric, and reusing the same custom Logs-Ingestion-API pattern already built for SQL DMVs to get file-level and log-folder-level granularity that generic disk monitoring alone wouldn't surface.

Want me to next produce the **combined Data Collection Rule ARM/JSON template** (covering Performance Counters + all custom log streams: SQL DMVs, file growth, cert inventory, connection state) as a single implementation-ready artifact for your integrators?
