# Consolidated Implementation Runbook  
## Hybrid Azure Monitor Pilot for On-Premises Windows ERP Servers

**Document status:** Draft for pilot execution  
**Audience:** System solution architects, system integrators, Windows infrastructure engineers, ERP support, database support, monitoring operations, security/IAM teams  
**Scope:** Certificate monitoring, disk capacity monitoring, ERP log-folder monitoring, Azure Monitor ingestion, dashboards, alerts, Teams/email notification, pilot validation, and production readiness

---

## 1. Purpose

This runbook defines the controlled implementation of a hybrid monitoring solution for on-premises Windows Servers hosting third-party ERP application servers, web/middleware servers, and related services.

The pilot keeps all workloads on-premises. Azure is used as the cloud monitoring, alerting, dashboard, and telemetry platform.

The pilot establishes a reusable monitoring pattern:

```text
On-premises Windows Server
    │
    ├── Azure Arc
    ├── Azure Arc system-assigned managed identity
    ├── Azure Monitor Agent where approved
    ├── Windows Task Scheduler
    └── Signed PowerShell custom collectors
           │
           ▼
Azure Monitor Logs Ingestion API
           │
           ▼
Data Collection Endpoint / Data Collection Rule
           │
           ▼
Log Analytics Workspace
           │
           ├── KQL queries
           ├── Azure Monitor Workbooks
           ├── Scheduled query alerts
           └── Action Groups
                  ├── Outlook/email
                  └── Logic App → Microsoft Teams
```

---

## 2. Pilot Scope

### 2.1 Included Use Cases

| Use case | Collection approach | Destination |
|---|---|---|
| Windows WebHosting certificate inventory | Custom PowerShell collector | `ServerCertificateInventory_CL` |
| Certificate expiry tracking | KQL, Workbook, scheduled-query alerts | Log Analytics / Azure Monitor |
| IIS binding correlation | Custom PowerShell collector with `WebAdministration` module | `ServerCertificateInventory_CL` |
| Windows logical-disk capacity | AMA counters where approved, or custom PowerShell collector | `Perf` or `WindowsDiskMetrics_CL` |
| Disk latency/queue metrics | AMA counters where approved, or custom PowerShell collector | `Perf` or `WindowsDiskMetrics_CL` |
| ERP log-directory size/growth | Custom PowerShell collector | `ErpFolderMetrics_CL` |
| ERP log retention-age tracking | Custom PowerShell collector | `ErpFolderMetrics_CL` |
| Collector execution health | Shared collector module | `CollectorHealth_CL` |
| Email notification | Azure Monitor Action Group | Outlook/email |
| Teams notification | Action Group → Logic App → Teams | Teams channel |

### 2.2 Deferred Use Cases

The following are future phases, not part of the initial certificate/disk pilot:

- SQL Server DMV monitoring.
- SQL Agent job and backup monitoring.
- REST API synthetic monitoring.
- TCP connection and port diagnostics.
- Service-account password expiry tracking.
- Windows LAPS/Entra LAPS metadata tracking.
- ERP-specific application log parsing.
- ServiceNow incident integration.

---

# 3. Architecture Decisions

| Decision | Pilot implementation |
|---|---|
| Server location | Windows servers remain on-premises. |
| Azure connectivity | Outbound public HTTPS/TCP 443 is permitted. |
| Hybrid management | Azure Arc-enabled servers. |
| Collector scheduler | Windows Task Scheduler. |
| Authentication | Azure Arc system-assigned managed identity. |
| Secrets | No workspace keys, service-principal secrets, or private keys on collector servers. |
| Azure IaC | Bicep. |
| Certificate store | `Cert:\LocalMachine\WebHosting`. |
| Folder scope | Approved ERP log directories only. |
| Alerts | Azure Monitor scheduled query alerts. |
| Notification | Outlook/email and Teams through Logic App. |
| Data store | Dedicated pilot Log Analytics Workspace. |
| Custom ingestion | Logs Ingestion API through DCE/DCR. |
| Script policy | Signed PowerShell scripts; `AllSigned` in production. |

---

# 4. Roles and Responsibilities

| Team | Responsibilities |
|---|---|
| System Solution Architect | Approves architecture, scope, security boundaries, rollout gates. |
| System Integrator | Builds Bicep, collectors, DCRs, tables, dashboards, alerts, and deployment automation. |
| Windows Infrastructure Team | Azure Arc onboarding, local task account, NTFS permissions, Task Scheduler, IIS validation. |
| ERP Application Support | Confirms ERP log locations, log retention policy, application ownership, safe cleanup procedures. |
| Azure Monitoring Team | Workspace, DCE, DCR, KQL, alerts, Workbooks, retention, costs. |
| Security/IAM Team | Managed identity, Azure RBAC, local account rights, code signing, data-classification approval. |
| Network Team | Proxy, DNS, firewall allowlists, outbound HTTPS validation. |
| Operations Support | Alert triage, Workbook usage, runbook execution, escalation, evidence capture. |

---

# 5. Phase 0 — Readiness and Prerequisites

## 5.1 Required Decisions

Before deployment, confirm:

1. Azure subscription and Azure region.
2. Pilot resource group name.
3. Dedicated pilot Log Analytics Workspace approval.
4. Azure Arc onboarding status for pilot servers.
5. System-assigned managed identity enabled for Arc machines.
6. Collector Scheduled Task run-as account.
7. ERP log directory path or paths.
8. Email distribution list.
9. Teams channel and Teams ownership.
10. Certificate renewal owner and escalation path.
11. Data-retention period.
12. Script-signing certificate and release process.

## 5.2 Pilot Server Set

| Server | Role | Required collectors |
|---|---|---|
| `ERP-APP-01` | ERP application server | ERP log-folder collector; logical-disk collector if AMA is not used. |
| `ERP-MW-01` | Middleware/IIS server | Certificate inventory collector. |
| Optional SQL server | SQL/data tier | Future phase only unless disk monitoring is included. |

## 5.3 Required Local Permissions

The scheduled-task account requires:

| Permission | Purpose |
|---|---|
| Log on as a batch job | Required by Windows Task Scheduler. |
| Membership in `Hybrid Agent Extension Applications` | Access Azure Arc HIMDS managed-identity endpoint. |
| Read access to `Cert:\LocalMachine\WebHosting` | Certificate inventory. |
| Read access to ERP log directories | Folder metrics. |
| Read access to IIS configuration, if enabled | IIS HTTPS binding correlation. |
| Modify access to collector `Logs`, `Spool`, and `Locks` folders | Operational logging, queueing, concurrency control. |
| Read/execute access to signed collector code | Collector execution. |

Do not grant local administrator permissions unless specifically required, tested, and approved.

---

# 6. Phase 1 — Collector Package Deployment

## 6.1 Deployment Location

```text
C:\ProgramData\Contoso\AzureMonitorCollectors\
├── Current\
│   ├── Modules\
│   ├── Collectors\
│   ├── Config\
│   └── Version\
├── Previous\
├── Logs\
├── Spool\
└── Locks\
```

## 6.2 Package Components

```text
AzureMonitorCollectors/
├── Install-CollectorPackage.ps1
├── Register-CollectorScheduledTasks.ps1
├── Rollback-CollectorPackage.ps1
├── Uninstall-CollectorPackage.ps1
├── Test-CollectorPrerequisites.ps1
│
├── Modules/
│   └── AzureMonitorCollector.Common.psm1
│
├── Collectors/
│   ├── Collect-CertificateInventory.ps1
│   ├── Collect-ErpFolderMetrics.ps1
│   ├── Collect-WindowsDiskMetrics.ps1
│   └── Submit-CollectorSpool.ps1
│
├── Config/
│   ├── CollectorSettings.json
│   ├── CertificateInventory.config.json
│   ├── ErpFolderMetrics.config.json
│   └── WindowsDiskMetrics.config.json
│
└── Version/
    └── release.json
```

## 6.3 Collector Runtime Controls

Every collector must:

- Create a unique execution ID.
- Acquire a lock file before execution.
- Prevent overlapping task runs.
- Log locally using sanitized JSON logs.
- Remove expired local logs/spool items.
- Replay old spool batches before new collection.
- Obtain token through Azure Arc HIMDS.
- Submit payloads in batches.
- Retry transient errors.
- Spool failed payloads where enabled.
- Submit a collector-health record.
- Exit with a non-zero status on unrecoverable failure.

## 6.4 Scheduled Tasks

| Task name | Frequency | Maximum runtime |
|---|---:|---:|
| `Contoso-Monitor-CertificateInventory` | Daily | 15 minutes |
| `Contoso-Monitor-ErpFolderMetrics` | Hourly initially | 30 minutes |
| `Contoso-Monitor-WindowsDiskMetrics` | Every 15 minutes, if custom disk collector enabled | 15 minutes |
| `Contoso-Monitor-*-SpoolReplay` | Every 15 minutes | 15 minutes |

Task setting:

```text
If the task is already running: Do not start a new instance.
```

---

# 7. Phase 2 — Azure Infrastructure Deployment

## 7.1 Azure Resources

| Resource | Purpose |
|---|---|
| Resource group | Groups pilot monitoring resources. |
| Log Analytics Workspace | Stores and queries telemetry. |
| Data Collection Endpoint | Logs Ingestion API endpoint. |
| Certificate DCR | Defines certificate JSON stream and routing. |
| ERP folder DCR | Defines folder-metrics JSON stream and routing. |
| Windows disk DCR | Defines custom logical-disk JSON stream and routing. |
| Collector health DCR | Defines collector-health JSON stream and routing. |
| Custom tables | Store the custom records. |
| Action Group | Sends email and invokes Teams Logic App. |
| Workbooks | Operations dashboards. |
| Scheduled query rules | Certificate/disk/folder/collector alerts. |

## 7.2 Custom Tables

| Table | Primary purpose |
|---|---|
| `ServerCertificateInventory_CL` | Certificate inventory, expiry, private-key status, IIS binding. |
| `ErpFolderMetrics_CL` | ERP folder size, growth, old files, errors, scan duration. |
| `WindowsDiskMetrics_CL` | Custom logical-disk capacity/latency metrics, if AMA is not used. |
| `CollectorHealth_CL` | Collector result, duration, counts, errors, and version. |

## 7.3 Custom DCR Streams

| Input stream | Output table stream |
|---|---|
| `Custom-ServerCertificateInventoryRaw` | `Custom-ServerCertificateInventory_CL` |
| `Custom-ErpFolderMetricsRaw` | `Custom-ErpFolderMetrics_CL` |
| `Custom-WindowsDiskMetricsRaw` | `Custom-WindowsDiskMetrics_CL` |
| `Custom-CollectorHealthRaw` | `Custom-CollectorHealth_CL` |

## 7.4 RBAC

Grant each Arc server’s system-assigned managed identity:

```text
Monitoring Metrics Publisher
```

Scope the role to only the required DCR.

Example:

| Server | DCR role scopes |
|---|---|
| `ERP-MW-01` | Certificate DCR + Collector Health DCR |
| `ERP-APP-01` | ERP Folder DCR + Windows Disk DCR, if used + Collector Health DCR |

---

# 8. Phase 3 — Certificate Monitoring Implementation

## 8.1 Collection Scope

```text
Cert:\LocalMachine\WebHosting
```

Collected fields include:

- Subject.
- SAN/DNS names.
- Issuer.
- Thumbprint.
- Serial number.
- Validity dates.
- Days until expiry.
- Private-key presence.
- Enhanced key usage.
- IIS HTTPS binding, if enabled.
- Collection status.
- Collector version.

Do not collect:

- Certificate private keys.
- PFX exports.
- Certificate passwords.
- Tokens.
- Secrets.

## 8.2 Certificate Alert Thresholds

| Alert | Severity | Threshold |
|---|---:|---|
| Expired certificate | Sev 1 | `DaysUntilExpiry < 0` |
| Expiry within 7 days | Sev 2 | `0–7 days` |
| Expiry within 14 days | Sev 2 | `8–14 days` |
| Expiry within 30 days | Sev 3 | `15–30 days` |
| IIS-bound certificate without private key | Sev 2 | `HasPrivateKey == false` and IIS binding exists |
| Certificate collector failed | Sev 2 | Latest health status not `Success` |
| Certificate inventory stale | Sev 2 | No inventory within 30 hours |

## 8.3 Core Certificate KQL

```kusto name=kql/certificate/CurrentCertificateInventory.kql
ServerCertificateInventory_CL
| where TimeGenerated > ago(7d)
| summarize arg_max(TimeGenerated, *) by Computer, StoreLocation, StoreName, Thumbprint
| project
    TimeGenerated,
    Computer,
    Application,
    StoreName,
    Subject,
    DnsNames,
    Issuer,
    Thumbprint,
    NotAfter,
    DaysUntilExpiry,
    HasPrivateKey,
    IisBindings,
    CollectionStatus
| order by DaysUntilExpiry asc
```

```kusto name=kql/certificate/ExpiredCertificates.kql
ServerCertificateInventory_CL
| where TimeGenerated > ago(30h)
| summarize arg_max(TimeGenerated, *) by Computer, StoreLocation, StoreName, Thumbprint
| where CollectionStatus == "Success"
| where DaysUntilExpiry < 0 or NotAfter < now()
| project
    Computer,
    StoreName,
    Subject,
    DnsNames,
    NotAfter,
    DaysUntilExpiry,
    IisBindings,
    Thumbprint
| order by DaysUntilExpiry asc
```

```kusto name=kql/certificate/CertificatesExpiring7Days.kql
ServerCertificateInventory_CL
| where TimeGenerated > ago(30h)
| summarize arg_max(TimeGenerated, *) by Computer, StoreLocation, StoreName, Thumbprint
| where CollectionStatus == "Success"
| where DaysUntilExpiry between (0 .. 7)
| project
    Computer,
    StoreName,
    Subject,
    DnsNames,
    NotAfter,
    DaysUntilExpiry,
    IisBindings,
    Thumbprint
| order by DaysUntilExpiry asc
```

## 8.4 Certificate Workbook Tabs

1. Certificate health summary.
2. Expired certificates.
3. Certificates expiring within 7 days.
4. Certificates expiring within 14 days.
5. Certificates expiring within 30 days.
6. IIS binding/private-key risk.
7. Certificate collector health.
8. Missing certificate inventory.

## 8.5 Certificate Response Procedure

1. Open the Certificate Monitoring Workbook.
2. Confirm affected server, subject, thumbprint, expiry, and IIS binding.
3. Confirm application/certificate owner.
4. Obtain replacement certificate through approved PKI process.
5. Import into `LocalMachine\WebHosting`.
6. Confirm private key exists.
7. Update IIS binding where needed.
8. Perform approved endpoint test.
9. Run collector manually.
10. Confirm replacement certificate is visible.
11. Confirm alert auto-resolves.
12. Close incident/change with evidence.

---

# 9. Phase 4 — Disk and ERP Log-Folder Monitoring

## 9.1 Collection Decision

Use this decision path:

```text
Is Azure Monitor Agent approved for performance counters?
    │
    ├── Yes:
    │     Use AMA performance counters for disk capacity/latency.
    │     Use PowerShell only for ERP folder metrics.
    │
    └── No:
          Use PowerShell logical-disk collector
          → WindowsDiskMetrics_CL.
```

Do not operate AMA and custom logical-disk alerts simultaneously for the same drives after production stabilization.

## 9.2 Disk Monitoring Thresholds

| Condition | Warning | Critical |
|---|---:|---:|
| Free percentage | Under 20% | Under 10% |
| Absolute free capacity | Under 50 GB | Under 20 GB |
| Read/write latency | Over 25 ms, after baseline validation | Define per application/storage tier |
| ERP folder growth | Over 5 GB/24 hours, initial threshold | Tune by workload |
| Retention accumulation | Files exceed approved retention | Planned remediation |

## 9.3 Core Custom Disk KQL

```kusto name=kql/disk-custom/CurrentDiskCapacity.kql
WindowsDiskMetrics_CL
| where TimeGenerated > ago(45m)
| summarize arg_max(TimeGenerated, *) by Computer, Drive
| where CollectionStatus in ("Success", "CounterUnavailable")
| project
    TimeGenerated,
    Computer,
    Application,
    Drive,
    VolumeLabel,
    TotalGB,
    FreeGB,
    UsedGB,
    FreePercent,
    UsedPercent,
    ReadLatencyMs,
    WriteLatencyMs,
    DiskQueueLength,
    CollectionStatus
| order by FreePercent asc, FreeGB asc
```

```kusto name=kql/disk-custom/DiskCapacityCritical.kql
WindowsDiskMetrics_CL
| where TimeGenerated > ago(45m)
| summarize arg_max(TimeGenerated, *) by Computer, Drive
| where CollectionStatus in ("Success", "CounterUnavailable")
| where FreePercent < 10 or FreeGB < 20
| project
    Computer,
    Application,
    Drive,
    VolumeLabel,
    TotalGB,
    FreeGB,
    FreePercent,
    CollectionStatus
| order by FreePercent asc, FreeGB asc
```

## 9.4 ERP Folder KQL

```kusto name=kql/folder/CurrentErpFolderMetrics.kql
ErpFolderMetrics_CL
| where TimeGenerated > ago(6h)
| summarize arg_max(TimeGenerated, *) by Computer, Application, FolderPath
| project
    TimeGenerated,
    Computer,
    Application,
    FolderPath,
    TotalGB,
    FileCount,
    OldestFileUtc,
    NewestFileUtc,
    FilesOlderThanRetention,
    RetentionDays,
    AccessErrorCount,
    ScanDurationSeconds,
    CollectionStatus
| order by TotalGB desc
```

```kusto name=kql/folder/ErpFolderGrowth24Hours.kql
let CurrentMetrics =
    ErpFolderMetrics_CL
    | where TimeGenerated > ago(2h)
    | summarize arg_max(TimeGenerated, TotalBytes, TotalGB, FileCount, CollectionStatus)
        by Computer, Application, FolderPath;
let PreviousMetrics =
    ErpFolderMetrics_CL
    | where TimeGenerated between (ago(26h) .. ago(22h))
    | summarize arg_max(TimeGenerated, TotalBytes, TotalGB, FileCount)
        by Computer, Application, FolderPath;
CurrentMetrics
| join kind=leftouter PreviousMetrics
    on Computer, Application, FolderPath
| extend GrowthGB24h = round(
    todouble(TotalBytes - TotalBytes1) / 1024 / 1024 / 1024,
    2
)
| project
    Computer,
    Application,
    FolderPath,
    TotalGB,
    GrowthGB24h,
    FileCount,
    CollectionStatus
| order by GrowthGB24h desc
```

## 9.5 Disk/Folder Workbook Tabs

1. Current disk capacity.
2. Critical capacity risk.
3. Disk latency and queue length.
4. Current ERP folder size.
5. ERP folder growth in 24 hours.
6. ERP retention accumulation.
7. Scan errors and timeouts.
8. Disk collector health.
9. ERP folder collector health.

## 9.6 Capacity Response Procedure

1. Open Disk and ERP Folder Monitoring Workbook.
2. Confirm server, drive, free GB, free percentage, and affected ERP folder.
3. Check abnormal folder growth, retention accumulation, and current incidents.
4. Preserve logs required for audit, legal hold, or incident investigation.
5. Use only approved ERP/vendor cleanup procedures.
6. Confirm log rotation/archival jobs are operating.
7. Extend capacity if approved and necessary.
8. Re-run collector.
9. Confirm capacity has recovered and alert resolves.
10. Record cause, action, and preventive control.

---

# 10. Alerting and Notification

## 10.1 Azure Monitor Alert Standards

Every alert must define:

- Display name.
- Description.
- Severity.
- KQL query.
- Evaluation frequency.
- Time window.
- Threshold.
- Failing period.
- Auto-mitigation.
- Mute duration.
- Action Group.
- Custom properties.
- Linked operational runbook.

## 10.2 Notification Flow

```text
Scheduled query alert
    │
    ▼
Action Group
    ├── Outlook/email
    └── Logic App action
           │
           ▼
       Teams Adaptive Card
```

## 10.3 Teams Requirements

- Use Common Alert Schema.
- Use Logic App, not ungoverned legacy webhook patterns.
- Post to a dedicated operational channel.
- Include severity, rule name, affected server, key results, runbook link, Workbook link, and alert portal link.
- Ensure Logic App failure is itself monitored.

---

# 11. Fault Injection and Test Execution

## 11.1 Test Categories

| Category | Examples |
|---|---|
| Certificate | Expiring test certificate, expired certificate, missing private key, missing store. |
| Disk/folder | Fill test volume, create old log file, increase folder size, deny folder access. |
| Collector | Disable task, force timeout, lock file conflict, invalid config. |
| Identity | Remove DCR role, remove HIMDS group membership, disable Arc identity. |
| Network | Block DCE endpoint, proxy failure, temporary outbound HTTPS interruption. |
| Ingestion | Invalid DCR stream, invalid field type, schema mismatch. |
| Notification | Email delivery, Teams delivery, alert resolution, duplicate suppression. |
| Recovery | Spool replay, collector rollback, Bicep redeployment. |

## 11.2 Mandatory Tests

| Test | Required proof |
|---|---|
| Certificate expiry | Alert fires, email/Teams sent, alert resolves after remediation. |
| Disk critical | Alert fires, capacity shown in Workbook, alert resolves after recovery. |
| ERP folder growth | Growth visible and alertable. |
| Access error | Folder collector reports `PartialSuccess`. |
| Ingestion authorization failure | Collector logs/spools safely; no credentials exposed. |
| Spool replay | Data reaches Log Analytics after connectivity restoration. |
| Missing collector data | Missing-data alert activates. |
| Task restart | Collector resumes after Windows restart. |
| Rollback | Previous signed package restores successful collection. |

---

# 12. Security Controls

## 12.1 Mandatory Controls

| Control | Requirement |
|---|---|
| Authentication | Azure Arc system-assigned managed identity. |
| Ingestion permission | `Monitoring Metrics Publisher`, scoped to required DCR only. |
| Secrets | No client secret, workspace key, password, token, or private key stored in collectors. |
| Script execution | Signed scripts/modules only in production. |
| Execution policy | `AllSigned`. |
| Local account | Dedicated service account; no interactive use. |
| NTFS | Collector account cannot modify collector code. |
| Spool files | Restricted ACLs and short retention. |
| Logging | Sanitize errors; never log tokens or secrets. |
| Workspace access | Azure RBAC and periodic access review. |
| Data classification | Review before any application-log, SQL, or sensitive business-data ingestion. |
| Change control | Peer-reviewed source changes and Bicep deployments. |

---

# 13. Cost Governance

## 13.1 Main Cost Drivers

| Component | Cost consideration |
|---|---|
| Log Analytics | Data ingestion and retention. |
| Alerts | Query frequency and number of alert rules. |
| Logic Apps | Workflow/connector executions. |
| Azure Arc | Validate current licensing/pricing in organization agreement. |
| Operations | Collector support, release management, and troubleshooting. |

## 13.2 Cost Controls

1. Certificate scans run daily.
2. Folder collection begins hourly.
3. Custom disk collection runs every 15 minutes only when required.
4. Aggregate metrics only; do not ingest file names or file contents.
5. Use 30–90-day pilot retention.
6. Use DCR transformations to drop unneeded fields.
7. Review ingestion at seven and 30 days.
8. Set resource-group and subscription budgets.
9. Do not ingest broad IIS, ERP, SQL query, or Windows event logs without scope and data-classification approval.

---

# 14. Monitoring the Monitoring Platform

## 14.1 Health Signals

| Component | Required check |
|---|---|
| Azure Arc | Server connected and identity enabled. |
| HIMDS | Scheduled-task account can obtain Azure token. |
| Scheduled Task | Last run result, next run, duration. |
| Collector | Latest `CollectorHealth_CL` record is successful. |
| Spool | File count, size, and oldest queue age. |
| DCR | Correct configuration, destination, and RBAC. |
| Log Analytics | Fresh records appear within expected delay. |
| Alerts | Enabled, evaluated, and notifying. |
| Action Group | Email receiver valid. |
| Logic App | Successful trigger and Teams post. |

## 14.2 Missing Data Thresholds

| Collector | Expected interval | Missing/stale alert threshold |
|---|---:|---:|
| Certificate inventory | Daily | 30 hours |
| ERP folder metrics | Hourly | 2 hours |
| Custom disk metrics | Every 15 minutes | 45–60 minutes |
| Collector health | Per execution | Same threshold as collector type |

---

# 15. Deployment, Change, and Rollback

## 15.1 Deployment Order

1. Approve architecture and security design.
2. Deploy Azure resources using Bicep.
3. Confirm DCR outputs, DCE endpoint, and DCR immutable IDs.
4. Apply generated collector configuration.
5. Deploy signed collector package.
6. Apply NTFS permissions.
7. Validate prerequisites under the Scheduled Task account.
8. Register scheduled tasks.
9. Run direct-ingestion validation.
10. Run collectors manually.
11. Validate KQL, Workbooks, alerts, email, and Teams.
12. Execute fault-injection tests.
13. Complete production-readiness report.

## 15.2 Rollback

```text
Collector release issue
    → disable task
    → preserve logs/spool
    → run Rollback-CollectorPackage.ps1
    → validate signature
    → run prerequisite test
    → run collector manually
    → validate Log Analytics and health record
```

For Azure resources:

```text
Approved prior Git tag
    → Bicep what-if
    → deploy prior release
    → validate tables/DCRs/alerts/workbooks
```

Avoid deleting tables or DCRs as a first response to an incident.

---

# 16. Production Rollout Waves

| Wave | Scope | Exit criterion |
|---|---|---|
| 0 | Lab/non-production | Collection, ingestion, alerting, and rollback validated. |
| 1 | Pilot ERP app and middleware servers | Certificate and ERP-folder use cases accepted. |
| 2 | Wider non-production ERP estate | Thresholds, scale, cost, and support model proven. |
| 3 | Low-criticality production servers | Production proxy/network/identity behavior validated. |
| 4 | Critical production ERP, middleware, and SQL-adjacent servers | Formal approval and change window. |
| 5 | Other business application platforms | Reuse validated architecture and release process. |

---

# 17. Production Readiness Gate

Production rollout is approved only when all conditions are met:

- Azure region and workspace strategy approved.
- Arc identity and DCR RBAC validated.
- Scripts signed and deployed through controlled process.
- Certificate, disk, folder, and collector-health monitoring works.
- Email and Teams notifications tested.
- Alert suppression and auto-resolution validated.
- Runbooks tested by operations.
- Cost measured for at least 30 days or an approved equivalent pilot period.
- No secrets/private keys/passwords appear in telemetry.
- Rollback procedure tested.
- Outstanding high-risk findings closed or formally accepted.
- Monitoring, ERP, Windows, security, and operations owners sign off.

---

# 18. Production Readiness Assessment Template

```markdown name=docs/operations/ProductionReadinessAssessment.md
# ERP Hybrid Monitoring Production Readiness Assessment

## Deployment Summary

- Pilot start date:
- Pilot end date:
- Azure region:
- Monitoring resource group:
- Log Analytics Workspace:
- Data Collection Endpoint:
- DCRs:
- Collector release:
- Servers monitored:
- Certificate stores:
- ERP log directories:
- Notification channels:

## Functional Acceptance

| Capability | Result | Evidence | Owner |
|---|---|---|---|
| Certificate inventory | Pass / Fail | | |
| Certificate expiry alert | Pass / Fail | | |
| IIS binding correlation | Pass / Fail | | |
| Disk capacity alert | Pass / Fail | | |
| ERP log-folder growth alert | Pass / Fail | | |
| Collector health | Pass / Fail | | |
| Missing telemetry detection | Pass / Fail | | |
| Email notification | Pass / Fail | | |
| Teams notification | Pass / Fail | | |
| Spool replay | Pass / Fail | | |
| Collector rollback | Pass / Fail | | |

## Security Acceptance

- Arc system-assigned managed identity:
- DCR-scope RBAC:
- Script-signing result:
- Task account review:
- NTFS ACL review:
- Data classification review:
- Secret scanning result:

## Cost and Performance

- Daily ingestion:
- Monthly cost estimate:
- Workspace retention:
- Logic App execution estimate:
- Maximum collector CPU:
- Maximum collector memory:
- Maximum folder scan duration:
- Maximum spool size:

## Open Risks

| Risk | Severity | Owner | Target date | Accepted |
|---|---:|---|---|---|

## Recommendation

- [ ] Approve production rollout.
- [ ] Extend pilot.
- [ ] Remediate gaps before production.
- [ ] Redesign selected components.

## Approvals

| Role | Name | Date | Approval |
|---|---|---|---|
| System Solution Architect | | | |
| System Integration Lead | | | |
| Windows Infrastructure Lead | | | |
| ERP Application Owner | | | |
| Security/IAM Lead | | | |
| Monitoring Operations Lead | | | |
| Budget Owner | | | |
```

---

# 19. Future Expansion Roadmap

After this pilot is accepted, reuse the same architecture for:

1. SQL DMV and wait-stat monitoring.
2. SQL backup and SQL Agent job monitoring.
3. REST API synthetic monitoring.
4. TCP dependency and port diagnostics.
5. Windows services and IIS application-pool health.
6. Service-account lifecycle and gMSA validation.
7. Windows LAPS/Entra LAPS metadata monitoring.
8. ERP/middleware application-log ingestion and error-pattern alerts.
9. ServiceNow incident creation/update integration.
10. Cross-tier ERP → middleware → SQL correlation Workbook.

---

# 20. Final Operational Principle

> Use Azure Arc and Azure Monitor for secure, cloud-managed observability of on-premises Windows ERP workloads; use AMA for standard telemetry where approved; use narrowly scoped, signed, managed-identity PowerShell collectors only where application-specific data is not available natively; and treat dashboards, alerts, runbooks, cost control, and monitoring-health checks as equal parts of the production solution.

### Want me to…
<!-- copilot:suggestions -->
1. Add Pester tests and a CI/CD pipeline design for the consolidated collector and Bicep solution.
2. Convert this runbook into a formal project charter, work-breakdown structure, milestone plan, and RACI matrix.
3. Hand this implementation documentation off to the coding agent to open a pull request in a repository you provide.
