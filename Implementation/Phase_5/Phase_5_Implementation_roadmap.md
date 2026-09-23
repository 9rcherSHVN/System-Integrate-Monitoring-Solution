# Phase 5 — Fault Injection, Production Hardening, Operations Handover, Cost Governance, and Rollout Gates

Phase 5 validates that the monitoring solution is not only technically functional, but also secure, supportable, cost-controlled, and ready to operate in production.

The goal is to prove this full chain:

```text
Windows server condition or collector fault
    → data collection
    → Azure Monitor ingestion
    → Log Analytics table
    → KQL alert
    → Action Group
    → Outlook/email
    → Logic App
    → Teams
    → operator triage
    → documented recovery
    → alert auto-resolution
```

---

# 1. Pilot Exit Criteria

The pilot should not be considered complete merely because data appears in Log Analytics. It should meet the following operational acceptance criteria.

| Domain | Required pilot proof |
|---|---|
| Azure Arc | Pilot servers are onboarded, healthy, and use system-assigned managed identities. |
| Windows scheduling | Collector tasks execute under the approved local service account, do not overlap, and recover after restart. |
| Authentication | Collectors obtain tokens through Azure Arc HIMDS; no client secrets or workspace keys are stored on servers. |
| Certificate monitoring | Inventory, expiry, IIS binding, and collector health are visible and alertable. |
| Disk monitoring | Logical-disk capacity/performance and ERP log-folder growth are visible and alertable. |
| Data ingestion | DCR schemas, DCE endpoint, DCR RBAC, batching, retry, and spool replay are tested. |
| Alerting | Critical, warning, collector failure, and missing-data alerts activate and resolve. |
| Notification | Outlook/email and Teams notifications are received with usable operational context. |
| Operations | Tier 1/Tier 2 runbooks are approved and tested. |
| Security | Least privilege, RBAC, script signing, data classification, and access controls are approved. |
| Cost | Ingestion, retention, alerting, Logic App, and operational costs are measured and accepted. |
| Recovery | Deployment rollback, collector recovery, spool replay, and configuration restore are tested. |
| Governance | Ownership, support hours, escalation, change control, and review cadence are defined. |

---

# 2. Fault-Injection Test Plan

Conduct controlled fault injection only in a non-production or formally approved pilot environment. Every test requires:

- Change or test approval.
- Defined start and end time.
- Named test owner.
- Rollback plan.
- Expected alert.
- Expected notification recipient.
- Evidence capture.
- Post-test cleanup.

Create a test register:

```text
docs/testing/PilotFaultInjectionRegister.xlsx
```

Recommended fields:

| Field | Example |
|---|---|
| Test ID | `FI-CERT-001` |
| Test category | Certificate / Disk / Collector / Ingestion / Alerting |
| Description | Create an expiring test certificate |
| Target server | `ERP-MW-01` |
| Test owner | ERP platform support |
| Approval reference | Change request ID |
| Expected KQL result | Record returned by `CertificatesExpiring7Days.kql` |
| Expected alert | `ERP Certificate Expires Within 7 Days` |
| Expected notification | Email + Teams |
| Start/end time | UTC |
| Actual result | Pass / fail |
| Evidence links | Screenshot, alert ID, task history |
| Remediation action | Required if failed |

---

## 2.1 Certificate Monitoring Fault Tests

| ID | Test | Injection method | Expected outcome |
|---|---|---|---|
| FI-CERT-001 | Near-expiry certificate | Import an isolated test certificate expiring within seven days. | Certificate inventory record appears; Sev 2 seven-day alert activates; email/Teams sent. |
| FI-CERT-002 | Expired certificate | Use an expired test certificate in an isolated store or test server. | Sev 1 expired alert activates. |
| FI-CERT-003 | Missing private key | Import public-only test certificate; associate with safe test binding only if approved. | Missing-private-key query identifies it; Sev 2 alert activates if it has IIS binding. |
| FI-CERT-004 | IIS binding mismatch | Change a test IIS binding to a different certificate. | Workbook shows changed thumbprint/binding; endpoint validation should be added in future phase. |
| FI-CERT-005 | Missing certificate store | Change test collector configuration to an invalid certificate store path. | Record shows `StoreNotFound`; health record becomes `PartialSuccess`. |
| FI-CERT-006 | Certificate collector disabled | Disable task in isolated pilot host. | Missing-data alert triggers after threshold. |
| FI-CERT-007 | Invalid DCR immutable ID | Temporarily alter test collector configuration. | Ingestion fails; payload spools; local log and health condition identify failure. |
| FI-CERT-008 | DCR authorization removal | Remove test Arc identity DCR role assignment. | HTTP 403; collector records/spools failure; restoration enables replay. |
| FI-CERT-009 | IAM recovery | Restore DCR role assignment. | Spool replay succeeds; records become visible. |

---

## 2.2 Disk and ERP Folder Fault Tests

| ID | Test | Injection method | Expected outcome |
|---|---|---|---|
| FI-DISK-001 | Disk warning | Fill an isolated test volume to cross warning threshold. | Warning alert activates; Workbook shows reduced free capacity. |
| FI-DISK-002 | Disk critical | Fill test volume to critical threshold only with approved test data. | Critical alert activates; escalation delivered. |
| FI-DISK-003 | ERP log growth | Add controlled test files to approved ERP test log path. | Folder growth query identifies 24-hour growth after test window. |
| FI-DISK-004 | Retention accumulation | Create a harmless test file and set old last-write timestamp. | `FilesOlderThanRetention` increases; retention alert activates. |
| FI-DISK-005 | Access denied | Restrict one test subfolder from the collector account. | `AccessErrorCount > 0`, `PartialSuccess`, health signal visible. |
| FI-DISK-006 | Scan timeout | Set a short timeout and scan a controlled large test directory. | `ScanTimedOut` status; health alert as designed. |
| FI-DISK-007 | Counter unavailable | Temporarily use invalid or unsupported counter path in isolated test configuration. | Capacity still collected; status becomes `CounterUnavailable`. |
| FI-DISK-008 | Drive missing | Add an unavailable test drive letter to configuration. | Record has `DriveNotFound`; collector returns `PartialSuccess`. |
| FI-DISK-009 | Disk collector disabled | Disable Scheduled Task on pilot host. | Missing-health/missing-metrics detection activates. |

---

## 2.3 Ingestion, Network, and Identity Fault Tests

| ID | Test | Injection method | Expected outcome |
|---|---|---|---|
| FI-ING-001 | Azure endpoint blocked | Temporarily block outbound HTTPS to DCE/ingestion endpoint. | Collector retries, spools payload, logs failure; no data loss within spool retention. |
| FI-ING-002 | Proxy failure | Apply invalid proxy configuration in non-production. | Token/ingestion fails predictably; error captured without exposing secrets. |
| FI-ING-003 | HIMDS access denied | Remove test task account from `Hybrid Agent Extension Applications`. | Managed-identity token acquisition fails; no Azure credential fallback occurs. |
| FI-ING-004 | Arc identity disabled | Disable system-assigned identity on test Arc server. | Collector cannot obtain token; health monitoring identifies failure. |
| FI-ING-005 | Invalid payload | Submit a test payload with schema/type mismatch. | Logs Ingestion API rejects it; no malformed record enters table. |
| FI-ING-006 | Azure throttling | Use controlled burst submission against a test DCR only. | Backoff/retry behavior is observed; no uncontrolled retry loop. |
| FI-ING-007 | Server restart | Restart test server while tasks are scheduled. | Task resumes at next schedule; no lock-file deadlock. |
| FI-ING-008 | Stale spool | Leave test spool batch past retention threshold. | Cleanup removes expired spool files; operational alert/report is generated if required. |

---

## 2.4 Alerting and Notification Fault Tests

| ID | Test | Injection method | Expected outcome |
|---|---|---|---|
| FI-ALERT-001 | Email delivery | Trigger test certificate warning. | Email arrives at approved operations mailbox. |
| FI-ALERT-002 | Teams delivery | Trigger test certificate warning. | Logic App posts Common Alert Schema-based card to Teams channel. |
| FI-ALERT-003 | Alert resolution | Correct the test condition and submit fresh data. | Alert auto-resolves; resolution notification is delivered. |
| FI-ALERT-004 | Duplicate suppression | Keep a condition active across multiple evaluation periods. | Mute duration prevents notification storm. |
| FI-ALERT-005 | Logic App failure | Disable Teams action in isolated Logic App test workflow. | Email still arrives; Logic App operational failure is logged/alerted. |
| FI-ALERT-006 | Action Group misconfiguration | Use a non-production test Action Group. | Deployment validation and notification test identify configuration issue before production use. |

---

# 3. Fault-Injection Evidence Collection

For every test, capture:

1. Scheduled Task result code.
2. Windows Task Scheduler Operational event log entry.
3. Local collector JSON log.
4. Spool file state before and after recovery, if applicable.
5. Azure Monitor alert ID.
6. Alert firing and resolution timestamp.
7. Email evidence.
8. Teams message evidence.
9. KQL output.
10. Workbook screenshot or exported result.
11. Root-cause explanation if actual behavior differs from expected behavior.
12. Corrective action, owner, and target completion date.

---

# 4. Production Hardening

## 4.1 Source Control and Release Engineering

Maintain all artifacts in source control:

```text
erp-hybrid-monitoring/
├── azure/
│   ├── main.bicep
│   ├── modules/
│   └── parameters/
├── collectors/
│   ├── Collectors/
│   ├── Modules/
│   ├── Config/
│   └── deployment/
├── kql/
├── workbooks/
├── docs/
├── tests/
└── pipeline/
```

Required branch model:

```text
feature/* → develop → release/* → main
```

Recommended controls:

- Pull-request review by system integrator and security/infrastructure owner.
- Protected `main` branch.
- Mandatory Bicep build/validation.
- Mandatory PowerShell linting.
- Mandatory Pester unit tests.
- Signed release artifacts only.
- Versioned Bicep parameters by environment.
- No production values embedded in source code.

---

## 4.2 PowerShell Quality Controls

Use PSScriptAnalyzer and Pester in the build pipeline.

### PSScriptAnalyzer Example

```powershell name=pipeline/Invoke-PowerShellStaticAnalysis.ps1
$scriptRoot = Split-Path -Path $PSScriptRoot -Parent
$collectorPath = Join-Path $scriptRoot 'collectors'

$results = Invoke-ScriptAnalyzer `
    -Path $collectorPath `
    -Recurse `
    -Severity Error, Warning

if ($results) {
    $results | Format-Table -AutoSize | Out-String | Write-Error
    throw 'PowerShell static analysis failed.'
}

Write-Output 'PowerShell static analysis passed.'
```

### Required Test Coverage

| Component | Minimum tests |
|---|---|
| Configuration | Missing file, invalid JSON, missing required properties. |
| Arc identity | HIMDS missing, invalid challenge, secret file absent, successful token parse. |
| Ingestion | Batch splitting, payload size limit, retry, spool creation, spool replay. |
| Certificate collector | Empty store, certificate with/without private key, IIS mapping, expired certificate. |
| Folder collector | Empty folder, inaccessible subfolder, reparse-point exclusion, timeout, growth calculation. |
| Disk collector | Capacity calculation, missing drive, counter unavailable, localized counter path handling. |
| Locking | Concurrent execution prevention and cleanup after failure. |
| Security | No token or secret written to local logs. |

---

## 4.3 Script Signing

Production policy:

```text
ExecutionPolicy = AllSigned
```

Controls:

1. Sign `.ps1` and `.psm1` files in a protected CI/CD process.
2. Use an approved enterprise code-signing certificate.
3. Ensure target Windows servers trust the issuing CA.
4. Verify signatures before deployment.
5. Block installation of unsigned releases.
6. Monitor code-signing certificate expiry.
7. Maintain a documented emergency process for signing-certificate replacement.

Do not place the code-signing private key on application servers.

---

## 4.4 Configuration Security

Configuration files may contain:

- DCE ingestion endpoint.
- DCR immutable ID.
- Stream name.
- Log path.
- Collection cadence.
- Threshold values.

These are not secret values, but must still be protected from unauthorized modification.

| File/folder | Collector account | Deployment account | Administrators |
|---|---:|---:|---:|
| `Current\Collectors` | Read/Execute | Modify | Full control |
| `Current\Modules` | Read/Execute | Modify | Full control |
| `Current\Config` | Read | Modify | Full control |
| `Logs` | Modify | Modify | Full control |
| `Spool` | Modify | Modify | Full control |
| `Locks` | Modify | Modify | Full control |

Configuration integrity is as important as code integrity. A modified DCR ID or endpoint can silently redirect or disable telemetry.

---

## 4.5 Azure RBAC Separation

Use separate Azure roles and identities.

| Function | Identity | Minimum authorization |
|---|---|---|
| Collector submission | Arc server system-assigned managed identity | `Monitoring Metrics Publisher` at required DCR scope |
| Azure infrastructure deployment | CI/CD deployment identity | Contributor or scoped deployment permissions on monitoring resource group; User Access Administrator only where role assignments are deployed |
| Monitoring operations | IT support users/group | Log Analytics Reader / Monitoring Reader |
| Workbook management | Monitoring engineering group | Workbook/monitoring contributor permissions |
| Alert administration | Monitoring engineering group | Monitoring Contributor or scoped equivalent |
| Security audit | Security team | Reader + Log Analytics Reader, according to data classification |

Do not grant collector identities:

- Owner.
- Contributor.
- Log Analytics Reader.
- Resource Group Reader, unless separately justified.
- Access to Azure Key Vault, unless a future use case requires it.

---

# 5. Operations Handover Model

## 5.1 Tiered Support Responsibilities

| Team | Responsibilities |
|---|---|
| Tier 1 Service Desk / Operations Centre | Receive alerts, acknowledge, open/run runbook, verify dashboard, perform initial triage, escalate. |
| Tier 2 Windows/Infrastructure | Azure Arc, Task Scheduler, local account, certificate store, IIS binding, disk/storage capacity, proxy/firewall. |
| Tier 2 ERP Application Support | ERP logging behavior, log retention, app configuration, vendor escalation, safe log cleanup. |
| Tier 2 Database Team | SQL log/data disk correlation, backup growth, SQL-related capacity impact. |
| Azure Monitoring Team | Log Analytics, DCE, DCR, RBAC, alert rules, Workbook, Action Group, cost. |
| Security/IAM Team | Script signing, managed identity governance, certificate policies, local service-account approval. |
| Network Team | Proxy, firewall, Azure endpoint allowlisting, DNS and TLS inspection issues. |

---

## 5.2 Required Operations Runbooks

| Runbook | Trigger |
|---|---|
| Certificate expiry response | Expired or expiring certificate alert. |
| IIS binding/private key response | IIS-bound certificate missing private key. |
| Disk capacity critical response | Free space below critical threshold. |
| ERP log growth response | Rapid folder growth or retention accumulation. |
| Disk latency response | High read/write latency. |
| Collector failure response | `CollectorHealth_CL` failure. |
| Missing telemetry response | No collector heartbeat/data in expected period. |
| Ingestion/DCR authorization response | HTTP 401/403/404/400 ingestion error. |
| Arc managed identity response | HIMDS/token failure. |
| Spool replay response | Payloads accumulating locally. |
| Deployment rollback response | Release failure after collector update. |

Each runbook should include:

1. Scope.
2. Alert trigger.
3. Ownership.
4. Prerequisites.
5. Immediate risk assessment.
6. Step-by-step investigation.
7. Approved remediation.
8. Escalation path.
9. Evidence requirements.
10. Resolution and closure criteria.

---

## 5.3 Daily Operations Checklist

| Check | Owner | Frequency |
|---|---|---:|
| Review active Sev 1/Sev 2 alerts | Operations | Daily |
| Review certificates expiring within 30 days | Certificate/application owner | Daily or weekly |
| Review disk/free-space trend | Infrastructure/ERP support | Daily |
| Review ERP log growth and retention accumulation | ERP support | Daily |
| Check collector health | Monitoring operations | Daily |
| Check spool folder size and age | Windows operations | Daily |
| Review ingestion/cost anomalies | Azure monitoring owner | Weekly |
| Review failed scheduled tasks | Windows operations | Daily |
| Review Azure Arc connection/identity health | Infrastructure | Weekly |
| Review alert false positives | Monitoring/app owners | Weekly during pilot |

---

# 6. Cost Governance

## 6.1 Cost Drivers

| Azure component | Primary cost driver |
|---|---|
| Log Analytics Workspace | Ingested GB and retention beyond included period. |
| Azure Monitor scheduled query rules | Rule frequency, query scope, and alert evaluations. |
| Action Groups | Usually low, but notification mechanisms may have service-specific costs. |
| Logic Apps | Trigger/action execution count, connector usage, plan type. |
| Azure Arc | Review current Arc-enabled server pricing/licensing for managed capabilities. |
| Outbound network | Usually organization-specific internet/proxy/telecom impact. |
| Custom collectors | Operational maintenance cost and local CPU/disk impact, not just Azure billing. |

## 6.2 Pilot Ingestion Estimate

Use this formula:

```text
Daily GB =
  average record size in bytes
  × records per collection
  × collections per day
  × monitored servers
  ÷ 1,073,741,824
```

### Example: Certificate Collector

```text
20 certificates/server
× 1 KB average record
× 1 collection/day
× 20 servers
≈ 0.0004 GB/day
```

Certificate inventory is generally negligible.

### Example: ERP Folder Metrics

```text
5 monitored folders/server
× 1 KB record
× 24 collections/day (hourly)
× 20 servers
≈ 0.0022 GB/day
```

Also negligible.

### Example: Custom Disk Collector

```text
4 drives/server
× 1 KB record
× 96 collections/day (every 15 minutes)
× 20 servers
≈ 0.0072 GB/day
```

Still low at pilot scale.

The major ingestion cost risk is not these aggregate collectors. It is broad ingestion of:

- IIS logs.
- ERP application logs.
- Windows Event Logs without filtering.
- SQL query text.
- High-frequency connection snapshots.
- Large error payloads.

---

## 6.3 Cost-Control Rules

1. Store aggregate folder metrics; do not ingest individual filenames.
2. Collect certificate inventory daily, not continuously.
3. Start custom disk collection at 15-minute intervals.
4. Restrict custom disk collection to named drives.
5. Use 30–90-day initial retention.
6. Do not collect raw SQL statement text by default.
7. Do not ingest full error-stack traces without data classification review.
8. Apply DCR transformations to remove unneeded fields before storage.
9. Set a budget and cost alert for the pilot resource group.
10. Review daily ingestion after one week and after one month.

---

# 7. Monitoring the Monitoring Platform

The monitoring platform must have its own health controls.

## 7.1 Required Health Signals

| Component | Health signal |
|---|---|
| Azure Arc agent | Connected status and last heartbeat. |
| Azure Arc managed identity | HIMDS token request works. |
| Scheduled task | Last run result, duration, and next run. |
| Collector process | `CollectorHealth_CL` success/partial/failure. |
| Custom ingestion | Submitted count versus collected count. |
| Spool | File count, total size, oldest queued item. |
| DCR | Current configuration/version and authorization. |
| Workspace | Ingestion latency and retention availability. |
| Alert rule | Enabled state and recent evaluation history. |
| Action Group | Email delivery / Logic App execution state. |
| Logic App | Trigger success, Teams connector outcome, failures. |

## 7.2 Spool Directory Monitoring Script

Add a local scheduled check or include this logic in the collector-health module.

```powershell name=Collectors/Test-CollectorSpoolHealth.ps1
[CmdletBinding()]
param(
    [string] $CollectorRoot = 'C:\ProgramData\Contoso\AzureMonitorCollectors',

    [int] $WarningFileCount = 10,

    [int] $CriticalFileCount = 100,

    [int] $MaximumOldestFileAgeMinutes = 60
)

Set-StrictMode -Version Latest
$ErrorActionPreference = 'Stop'

$spoolRoot = Join-Path $CollectorRoot 'Spool'

if (-not (Test-Path -LiteralPath $spoolRoot)) {
    Write-Output 'Spool directory does not exist; no queued batches found.'
    exit 0
}

$files = Get-ChildItem -LiteralPath $spoolRoot -Recurse -File -Filter '*.json'

$count = $files.Count
$oldestFile = $files | Sort-Object LastWriteTimeUtc | Select-Object -First 1

$oldestAgeMinutes = if ($oldestFile) {
    [Math]::Round(
        ((Get-Date).ToUniversalTime() - $oldestFile.LastWriteTimeUtc).TotalMinutes,
        2
    )
}
else {
    0
}

$result = [pscustomobject]@{
    TimeGenerated        = (Get-Date).ToUniversalTime().ToString('o')
    Computer             = $env:COMPUTERNAME
    SpoolFileCount       = $count
    OldestFileAgeMinutes = $oldestAgeMinutes
    Status               = if ($count -ge $CriticalFileCount -or
                               $oldestAgeMinutes -ge $MaximumOldestFileAgeMinutes) {
        'Critical'
    }
    elseif ($count -ge $WarningFileCount) {
        'Warning'
    }
    else {
        'Success'
    }
}

$result | ConvertTo-Json -Compress
```

For production, send this output into `CollectorHealth_CL` or a dedicated `CollectorSpoolHealth_CL` table.

---

# 8. Backup, Recovery, and Rollback

## 8.1 What Must Be Recoverable

| Asset | Recovery method |
|---|---|
| Bicep templates | Source-control repository and protected release tags. |
| DCR/table/workbook/alert definitions | Redeploy Bicep from approved release. |
| Collector scripts/modules | Signed package release artifact. |
| Local configurations | Deployment-management source plus encrypted/controlled backup where required. |
| Scheduled Task definitions | Registration scripts or exported XML definitions. |
| Task account membership | IAM/local-group provisioning documentation and automation. |
| Logic App workflow | IaC/template export and source control. |
| KQL/runbooks | Version-controlled documentation. |
| Local spool data | Temporary operational data only; replay before cleanup where possible. |

## 8.2 Rollback Procedure

1. Disable affected scheduled tasks if a collector release is causing operational impact.
2. Preserve current local logs and spool files.
3. Execute `Rollback-CollectorPackage.ps1`.
4. Confirm previous signed version becomes active.
5. Re-register tasks only if task definitions changed.
6. Run prerequisite validation.
7. Run collector manually.
8. Verify data and health record arrive.
9. Close incident/change only after alert/workbook validation.

## 8.3 Azure Configuration Rollback

Do not manually delete production DCRs, tables, or alert rules during an incident unless required for safety.

Preferred approach:

```text
Git release tag
    → Bicep what-if
    → deploy previous approved Bicep release
    → validate resources and alerts
```

For destructive schema changes, create new table/version rather than deleting an existing production table.

---

# 9. Lifecycle Management

## 9.1 Update Cadence

| Component | Review cadence |
|---|---:|
| Collector code | Monthly or after incident/change. |
| Azure Arc agent | Per enterprise patch/agent policy. |
| PowerShell runtime | Per Windows patching policy. |
| DCR schemas | Controlled change only. |
| KQL and alert thresholds | Monthly during pilot; quarterly after production stabilization. |
| Workbook | Monthly during pilot; quarterly after stabilization. |
| Certificates | Daily monitoring, monthly ownership review. |
| Disk thresholds | After baseline collection and major workload/storage changes. |
| RBAC assignments | Quarterly access review. |
| Action Group recipients | Quarterly or personnel-change event. |
| Logic App/Teams connector | Quarterly and after Microsoft platform changes. |

## 9.2 Schema Versioning

Use additive changes only when possible.

Example:

```text
WindowsDiskMetrics_CL v1
    ├── existing fields retained
    └── new optional field added: VolumeRole

Future breaking change
    └── create WindowsDiskMetricsV2_CL
```

Every record includes:

```text
CollectorVersion = "1.0.0"
```

Recommended versioning:

```text
MAJOR.MINOR.PATCH

1.0.0 — initial supported release
1.1.0 — add non-breaking optional metric
1.1.1 — defect correction
2.0.0 — incompatible schema or behavioral change
```

---

# 10. Security Hardening Checklist

| Control | Production requirement |
|---|---|
| Azure Arc system-assigned identity | Enabled only for approved collector hosts. |
| DCR RBAC | `Monitoring Metrics Publisher` scoped per DCR. |
| Local token access | Scheduled task account in required Arc local group only. |
| Task identity | Dedicated local service account; no interactive use. |
| Script signing | Mandatory, trusted enterprise signing chain. |
| Execution policy | `AllSigned`. |
| NTFS ACLs | Collector cannot modify its own signed code. |
| DCR endpoint | HTTPS only; validate proxy/TLS behavior. |
| Secrets | No client secret, workspace key, or certificate private key in scripts/config/logs. |
| Logging | Sanitize errors; suppress tokens and raw payloads. |
| Spool | Restrictive ACLs, short retention, no secrets. |
| Workspace access | Azure RBAC; least privilege; periodic review. |
| Alert channels | Restricted Teams channel and owned email distribution list. |
| Data classification | Approved before collecting new application logs or SQL telemetry. |
| Change control | Peer review, tested deployment, rollback plan. |

---

# 11. Production Rollout Waves

Use progressive rollout rather than deploying to all servers simultaneously.

| Wave | Scope | Objective |
|---|---|---|
| Wave 0 | Lab/non-production server | Validate scripts, DCRs, RBAC, endpoint access, and alert flow. |
| Wave 1 | Pilot ERP app + middleware + one SQL-adjacent server | Validate realistic application paths and operational support. |
| Wave 2 | Non-production ERP estate | Validate scale, schemas, thresholds, cost, and release process. |
| Wave 3 | Low-criticality production servers | Validate production network/proxy, schedule, and support workflow. |
| Wave 4 | Business-critical production ERP/middleware/SQL ecosystem | Roll out only after pilot acceptance and change approval. |
| Wave 5 | Broader Windows application platform | Reuse proven collector/IaC framework for other business applications. |

## Per-Wave Go/No-Go Questions

1. Did all collectors report successful health telemetry?
2. Did all intended tables receive records?
3. Are alert thresholds producing manageable signal quality?
4. Are Teams and email notifications usable?
5. Is the support team using the Workbook/runbook successfully?
6. Is ingestion cost within forecast?
7. Are spool queues stable and empty under normal conditions?
8. Are no secrets or sensitive payloads appearing in logs?
9. Is CPU/memory impact on monitored servers acceptable?
10. Has rollback been tested?
11. Are all owners and escalation paths confirmed?

---

# 12. Production Go/No-Go Gate

Proceed from pilot to production only if all mandatory gates pass.

| Gate | Mandatory condition |
|---|---|
| Architecture | Azure region, workspace strategy, DCE/DCR design, identity model approved. |
| Security | RBAC, script signing, local task account, data-classification review approved. |
| Platform | Arc, HIMDS, DCR, ingestion, tables, and alerts operational. |
| Functional | Certificate, disk, and ERP folder use cases meet success criteria. |
| Notification | Email and Teams notifications tested for firing and resolution. |
| Operations | Runbooks tested by Tier 1 and Tier 2 support. |
| Resilience | Retry, spool replay, missing-data alert, rollback, and recovery tested. |
| Cost | 30-day measured cost accepted by budget owner. |
| Performance | Collector runtime and resource usage accepted by server owners. |
| Documentation | Architecture, configuration, support, deployment, and recovery documents complete. |
| Governance | Change-management process and ongoing ownership established. |
| Risk | Outstanding high-severity pilot risks resolved or formally accepted. |

---

# 13. Production Acceptance Report Template

Create:

```text
docs/operations/ProductionReadinessAssessment.md
```

```markdown name=docs/operations/ProductionReadinessAssessment.md
# ERP Hybrid Monitoring Production Readiness Assessment

## Deployment Summary

- Pilot start date:
- Pilot end date:
- Azure region:
- Log Analytics Workspace:
- DCE:
- DCRs:
- Collector version:
- Servers monitored:
- Certificate stores monitored:
- ERP log directories monitored:

## Functional Results

| Capability | Result | Evidence | Owner |
|---|---|---|---|
| Certificate inventory | Pass / Fail | KQL / Workbook link | |
| Certificate expiry alert | Pass / Fail | Alert ID | |
| IIS binding correlation | Pass / Fail | Workbook screenshot | |
| Disk capacity alert | Pass / Fail | Alert ID | |
| ERP folder growth alert | Pass / Fail | Alert ID | |
| Collector health alert | Pass / Fail | Alert ID | |
| Email notification | Pass / Fail | Evidence | |
| Teams notification | Pass / Fail | Evidence | |
| Spool replay | Pass / Fail | Evidence | |
| Rollback | Pass / Fail | Evidence | |

## Security Results

- Script-signing validation:
- Arc managed-identity validation:
- DCR-scope RBAC validation:
- Local task-account permissions validation:
- Data classification review:
- Secrets scanning result:

## Cost and Performance Results

- Daily ingestion:
- Monthly forecast:
- Alert query cost/volume:
- Logic App execution forecast:
- Collector CPU impact:
- Collector memory impact:
- Maximum folder scan duration:
- Maximum spool size:

## Outstanding Risks

| Risk | Severity | Owner | Target date | Accepted? |
|---|---:|---|---|---|

## Recommendation

- [ ] Approve production Wave 3 rollout.
- [ ] Extend pilot.
- [ ] Remediate identified gaps before proceeding.
- [ ] Reject current architecture and redesign.

## Approval

| Role | Name | Date | Approval |
|---|---|---|---|
| System Solution Architect | | | |
| System Integration Lead | | | |
| Windows Infrastructure Lead | | | |
| ERP Application Owner | | | |
| Security/IAM Lead | | | |
| Operations Support Lead | | | |
| Budget Owner | | | |
```

---

# 14. Final System Integrator Competencies

System integrators supporting this design must be competent in:

## Azure Platform

- Azure Arc-enabled servers.
- Azure Arc system-assigned managed identities.
- Azure Monitor Logs Ingestion API.
- DCE and DCR architecture.
- Log Analytics Workspace table schema.
- Azure RBAC.
- Bicep and Azure CLI deployment.
- Azure Monitor Workbooks.
- Scheduled query alerts and Action Groups.
- Logic Apps and Common Alert Schema.

## Windows Platform

- Task Scheduler.
- Windows service accounts and local rights.
- PowerShell security and signing.
- Certificate stores and IIS bindings.
- Logical disks and Windows performance counters.
- File-system ACLs, reparse points, and long-running scans.
- Windows event/task diagnostics.

## Operations

- KQL.
- Incident response.
- Alert suppression and correlation.
- Capacity planning.
- Certificate lifecycle management.
- Data-retention and cost governance.
- Release/change/rollback process.
- Documentation and runbook discipline.

---

# Final Pilot Outcome

At the completion of Phase 5, the organization will have a validated hybrid monitoring pattern for on-premises Windows ERP workloads:

```text
On-premises Windows servers
    → Azure Arc managed identity
    → secure custom PowerShell collectors
    → Azure Monitor ingestion / DCR
    → Log Analytics
    → KQL / Workbooks
    → alerts
    → email and Teams
    → documented Tier 1 / Tier 2 response
```

The certificate and disk/ERP-log monitoring implementations become reusable reference patterns for future expansion into:

- SQL Server DMV monitoring.
- SQL backup/job monitoring.
- REST API synthetic monitoring.
- TCP connection and port diagnostics.
- Service-account and Windows LAPS metadata tracking.
- IIS and third-party middleware log monitoring.
- ERP application-specific health endpoints.

### Want me to…
<!-- copilot:suggestions -->
1. Add Pester test suites and a CI/CD pipeline design for the collectors and Bicep deployment.
2. Produce a consolidated implementation runbook that merges Phases 0–5 into one controlled project document.
3. Hand this implementation off to the coding agent to open a pull request in a repository you provide.
