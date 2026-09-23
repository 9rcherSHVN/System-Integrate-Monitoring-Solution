# Phase 0 — Architecture, Identity, and Readiness

## 0.1 Phase Objective

Phase 0 establishes the architectural, security, networking, operational, and governance foundations required before system integrators begin building collectors, deploying scheduled tasks, or provisioning Azure Monitor resources.

The immediate pilot scope is limited to two custom-monitoring use cases:

1. **Certificate expiration monitoring**
   - Windows certificate store: `Cert:\LocalMachine\WebHosting`
   - IIS binding correlation where IIS is installed
   - Alerting for expired and soon-to-expire certificates

2. **ERP log-directory capacity monitoring**
   - Approved ERP log folder paths
   - Folder size, file count, growth, retention-age, and scan health
   - Alerting for rapid growth, retention exceptions, and collection failure

The selected operating model is:

```text
On-premises Windows Server
    │
    ├── Azure Arc Connected Machine agent
    ├── Azure Arc system-assigned managed identity
    ├── Windows Task Scheduler
    └── PowerShell collectors
          ├── Certificate inventory collector
          ├── ERP folder metrics collector
          └── Shared Logs Ingestion API module
                │
                │ Outbound HTTPS / TCP 443
                ▼
Azure Monitor Logs Ingestion API
                │
                ▼
Data Collection Rule
                │
                ▼
Log Analytics Workspace custom tables
                │
                ├── KQL queries
                ├── Azure Workbooks
                └── Scheduled query alerts
                       └── Action Group
                             ├── Microsoft Teams
                             └── Outlook/email
```

> **Design boundary:** Azure Monitor Agent and standard DCRs collect supported native Windows telemetry. The custom PowerShell collectors are reserved for certificate-store inventory and ERP log-directory metrics that require local inspection logic.

---

# 0.2 Architecture Decisions Register

## Confirmed Decisions

| ID | Decision | Pilot design consequence |
|---|---|---|
| AD-01 | Windows servers remain on-premises. | Azure Arc provides Azure resource management; workloads are not migrated to Azure VMs. |
| AD-02 | Public outbound HTTPS is allowed. | Custom collectors use Azure Monitor public ingestion endpoints over TCP 443. Private Link is deferred. |
| AD-03 | Windows Task Scheduler is the custom-collector scheduler. | Every monitored server hosts the appropriate local collector package and scheduled task. |
| AD-04 | Managed identity is required. | Each collector host uses the Azure Arc system-assigned managed identity; no Azure client secrets are stored locally. |
| AD-05 | Bicep is the infrastructure-as-code standard. | Tables, DCRs, role assignments, alert rules, Action Groups, and Workbooks are deployed through Bicep. |
| AD-06 | Pilot alert delivery is Teams and Outlook/email. | ServiceNow is deferred; alert design will retain fields needed for a future ITSM connector/Logic App flow. |
| AD-07 | Certificate pilot scope is `Cert:\LocalMachine\WebHosting`. | The collector will not enumerate all certificate stores by default. |
| AD-08 | Disk/folder scope is ERP logs. | The initial folder collector scans only approved ERP log directories. |
| AD-09 | Custom collectors submit through Azure Monitor Logs Ingestion API. | DCR streams, custom workspace tables, and the Arc identity’s ingestion authorization are required. |

## Decisions Still Required Before Deployment

| ID | Required decision | Recommended pilot position |
|---|---|---|
| PD-01 | Azure region | Use the organization’s approved data-residency region. A Canadian organization would commonly evaluate Canada Central first, subject to enterprise standards. |
| PD-02 | Log Analytics Workspace strategy | Create a dedicated pilot workspace to isolate access, cost, retention, and schema changes. |
| PD-03 | Task Scheduler run-as identity | Use a dedicated local service account; confirm with Windows security team. |
| PD-04 | ERP log paths | Define the exact, approved paths, exclusions, and expected retention period. |
| PD-05 | Teams integration pattern | Confirm whether the organization permits Action Group-to-Logic App-to-Teams or requires a centrally managed Teams notification workflow. |
| PD-06 | Email escalation mailbox | Define the operational mailbox/distribution list and owner. |
| PD-07 | Certificate ownership metadata | Confirm whether an application/service ownership map exists or needs to be maintained as collector configuration. |
| PD-08 | IIS scope | Confirm whether the pilot certificate collector must discover IIS HTTPS bindings. |
| PD-09 | Data retention | Define interactive Log Analytics retention for pilot telemetry, recommended initially at 30–90 days. |

---

# 0.3 Component Responsibility Model

| Component | Responsibility | Does it execute collectors? |
|---|---|---|
| Azure Arc Connected Machine agent | Registers on-premises server in Azure, supports extensions, exposes Azure Arc machine identity capability. | No |
| Azure Arc system-assigned managed identity | Authenticates the machine to Azure without a locally stored secret. | No |
| Azure Monitor Agent | Collects supported native telemetry according to DCR configuration. | No custom PowerShell execution |
| Windows Task Scheduler | Runs the local PowerShell scripts at defined intervals. | Yes |
| PowerShell certificate collector | Reads `Cert:\LocalMachine\WebHosting`, optionally correlates IIS bindings, creates records. | Yes |
| PowerShell ERP folder collector | Calculates folder size, file count, retention-age, and scan state. | Yes |
| Shared ingestion module | Gets an Arc managed-identity token, batches JSON, retries transient requests, posts telemetry. | Yes |
| Data Collection Endpoint (DCE) | Provides custom ingestion/configuration endpoint where required by selected Azure Monitor architecture. | No |
| Data Collection Rule (DCR) | Defines accepted input streams, optional transformations, and Log Analytics destinations. | No |
| Log Analytics Workspace | Stores, indexes, retains, and exposes data to KQL queries. | No |
| Azure Monitor alert rule | Evaluates KQL results periodically and triggers Action Groups. | No |
| Action Group | Invokes Teams/Logic App and email notification mechanisms. | No |
| Azure Workbook | Displays KQL-query results in an operations dashboard. | No |

---

# 0.4 Azure Arc Managed Identity Design

## 0.4.1 Why Azure Arc Identity Is the Correct Choice

Because the pilot uses **Windows Task Scheduler**, the collector runs locally on each on-premises Windows server. It is not executing inside Azure Automation.

The required Azure authentication mechanism is therefore:

> **System-assigned managed identity on the Azure Arc-enabled server.**

The Azure Arc agent provides a local **Hybrid Instance Metadata Service (HIMDS)** endpoint. A local collector can request a Microsoft Entra token from this endpoint and use that token to call Azure services, including the Azure Monitor Logs Ingestion API.

This avoids:

- Stored client secrets.
- Stored service-principal certificates.
- Manual credential rotation on every monitored server.
- A shared high-privilege collector identity across unrelated servers.

## 0.4.2 Authentication Flow

```text
Scheduled task runs locally
        │
        ▼
PowerShell ingestion module
        │
        │ Requests token locally for:
        │ https://monitor.azure.com/
        ▼
Azure Arc HIMDS local endpoint
        │
        │ Uses this server's Arc system-assigned managed identity
        ▼
Microsoft Entra ID
        │
        ▼
Access token returned to local PowerShell process
        │
        ▼
HTTPS POST to Azure Monitor Logs Ingestion API
        │
        ▼
DCR validates stream and routes to Log Analytics Workspace
```

The local HIMDS endpoint is commonly exposed through the `IDENTITY_ENDPOINT` environment variable and uses a local endpoint such as:

```text
http://localhost:40342/metadata/identity/oauth2/token
```

Do not hard-code this URL in the production collector. The collector should use the environment variable when present and validate that it is a loopback endpoint.

## 0.4.3 Arc Managed Identity Enabling Sequence

For each server hosting a custom collector:

1. Install and onboard the Azure Connected Machine agent.
2. Confirm the machine appears as an Azure Arc-enabled server.
3. Enable the server’s system-assigned managed identity.
4. Confirm Microsoft Entra service principal creation for the Arc server identity.
5. Add the appropriate local execution account to the authorized local group, subject to security approval.
6. Assign the Azure Arc machine identity the minimum custom-ingestion role at the applicable DCR scope.
7. Test token acquisition locally.
8. Test a single telemetry submission to a non-production custom stream.

## 0.4.4 Local Access Control

Azure Arc restricts local managed-identity token access. The scheduled-task run-as account must be allowed to access HIMDS.

The intended design is:

```text
Dedicated local collector account
      │
      ├── Local policy: Log on as a batch job
      ├── Member of Hybrid Agent Extension Applications
      ├── Read certificate-store permission as needed
      ├── Read/list access to approved ERP log folders
      ├── Modify access only to collector working/log/spool folders
      └── No interactive logon and no unrestricted local administrator role
```

> Validate access to `Cert:\LocalMachine\WebHosting` in the pilot. Depending on Windows hardening and IIS/binding requirements, read access may require additional controlled permissions. Do not grant local administrator access by default; grant it only if testing proves it is required and security approves it.

## 0.4.5 Token Request Validation Test

The integrator will later include this in the ingestion module. During readiness validation, test the Arc local identity first:

```powershell name=Test-ArcManagedIdentity.ps1
$resource = [uri]::EscapeDataString('https://monitor.azure.com/')
$apiVersion = '2020-06-01'

if ([string]::IsNullOrWhiteSpace($env:IDENTITY_ENDPOINT)) {
    throw 'IDENTITY_ENDPOINT is not available. Confirm Azure Arc managed identity is enabled.'
}

$tokenUri = "$($env:IDENTITY_ENDPOINT)?resource=$resource&api-version=$apiVersion"

$response = Invoke-RestMethod `
    -Method Get `
    -Uri $tokenUri `
    -Headers @{ Metadata = 'True' } `
    -ErrorAction Stop

if ([string]::IsNullOrWhiteSpace($response.access_token)) {
    throw 'No managed identity access token was returned.'
}

Write-Output 'Azure Arc managed identity token acquisition succeeded.'
```

This test proves only local token acquisition. It does **not** prove authorization to ingest telemetry. That is validated after the DCR and role assignment exist.

---

# 0.5 Task Scheduler Execution Model

## 0.5.1 Recommended Job Separation

Use separate scheduled tasks. Do not run all collectors in one monolithic task.

| Scheduled task | Host | Frequency | Purpose |
|---|---|---:|---|
| `Contoso-Monitor-CertificateInventory` | Web/middleware servers with WebHosting certificates | Daily, e.g. 02:15 | Inventory certificate metadata and IIS bindings. |
| `Contoso-Monitor-ErpLogFolderMetrics` | ERP application servers | Every 15–60 minutes | Measure approved ERP log directory size and growth. |
| `Contoso-Monitor-CollectorHealthRetry` | Optional only if local spool/retry is used | Every 15 minutes | Resubmit queued failed telemetry batches. |

Replace `Contoso` with the organization’s approved namespace.

## 0.5.2 Recommended Run-As Account

Use a dedicated local account, for example:

```text
.\svc_AzureMonitorCollector
```

The name is illustrative only. Follow enterprise naming standards.

### Required Windows Rights

| Right/permission | Reason |
|---|---|
| Log on as a batch job | Required by Task Scheduler for non-interactive execution. |
| Read required certificate store | Needed by the certificate collector. |
| Read IIS configuration, if enabled | Needed for IIS binding correlation. |
| Read/list ERP log directories | Needed for folder-metric calculation. |
| Create/write local collector log and spool directory | Needed for diagnostics and optional failed-ingestion retry. |
| Membership in `Hybrid Agent Extension Applications` | Enables local Arc managed-identity token access, subject to tested configuration. |
| No local admin by default | Supports least privilege. |

### Do Not Grant

- Domain Admin.
- Enterprise Admin.
- Local Administrator unless formally required.
- SQL Sysadmin.
- Permission to retrieve LAPS passwords.
- Broad Azure Contributor/Owner roles.
- Read access to unrelated application directories.

## 0.5.3 Task Settings

| Setting | Recommendation |
|---|---|
| Run whether user is logged on or not | Yes |
| Run with highest privileges | No by default; enable only after documented testing/security approval |
| Configure for | Target Windows Server version |
| Start in | Collector root directory |
| Program | `C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe` or approved PowerShell executable |
| Arguments | `-NoLogo -NoProfile -NonInteractive -ExecutionPolicy AllSigned -File "<script path>"` |
| Maximum runtime | Certificate: 15 minutes; folder scan: determine during pilot, initially 30 minutes |
| If task already running | Do not start a new instance |
| Retry after failure | Controlled task retry, such as 3 attempts at 15-minute interval |
| History | Enable Task Scheduler operational history during pilot |
| Conditions | Disable “Start only if on AC power” for servers; avoid idle-only conditions |
| Network condition | Do not make the task dependent solely on a Windows network profile; collector should handle temporary Azure endpoint failure itself |

## 0.5.4 Example Task Invocation

```powershell name=Register-CertificateCollectorTask.ps1
$taskName = 'Contoso-Monitor-CertificateInventory'
$collectorRoot = 'C:\ProgramData\Contoso\AzureMonitorCollectors'
$scriptPath = Join-Path $collectorRoot 'Collectors\Collect-CertificateInventory.ps1'

$action = New-ScheduledTaskAction `
    -Execute 'C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe' `
    -Argument (
        '-NoLogo -NoProfile -NonInteractive -ExecutionPolicy AllSigned ' +
        "-File `"$scriptPath`""
    ) `
    -WorkingDirectory $collectorRoot

$trigger = New-ScheduledTaskTrigger -Daily -At 2:15AM

$settings = New-ScheduledTaskSettingsSet `
    -ExecutionTimeLimit (New-TimeSpan -Minutes 15) `
    -MultipleInstances IgnoreNew `
    -RestartCount 3 `
    -RestartInterval (New-TimeSpan -Minutes 15) `
    -StartWhenAvailable

$principal = New-ScheduledTaskPrincipal `
    -UserId '.\svc_AzureMonitorCollector' `
    -LogonType Password `
    -RunLevel LeastPrivilege

Register-ScheduledTask `
    -TaskName $taskName `
    -Action $action `
    -Trigger $trigger `
    -Settings $settings `
    -Principal $principal `
    -Description 'Collects WebHosting certificate inventory and submits telemetry to Azure Monitor.' `
    -Force
```

This is a design sample. Production task registration must integrate with the organization’s local-account provisioning and password-management standard. If a passwordless local-account execution pattern is mandated, validate its compatibility with Task Scheduler and access to the Azure Arc HIMDS endpoint.

---

# 0.6 Public HTTPS Network and Firewall Readiness

## 0.6.1 Required Traffic Direction

The pilot requires **outbound-only** HTTPS from collector servers. Azure should not initiate inbound monitoring sessions to the on-premises servers.

| Source | Destination | Port | Purpose |
|---|---|---:|---|
| Arc-enabled Windows server | Azure Arc service endpoints | TCP 443 | Azure Arc registration, machine management, identity operations |
| Arc-enabled Windows server | Azure Monitor/Logs ingestion endpoint | TCP 443 | Custom collector JSON ingestion |
| Arc-enabled Windows server | Microsoft Entra ID endpoints | TCP 443 | Managed-identity token acquisition path |
| Administrator workstation/CI pipeline | Azure Resource Manager | TCP 443 | Bicep deployment and configuration |
| Azure Monitor | Teams/email destination | Azure managed | Alert notification through Action Group/Logic App pattern |

The exact Azure endpoint/FQDN allowlist must be taken from Microsoft’s current Azure Arc and Azure Monitor network requirements at implementation time. Do not rely on a static copied list in a script or firewall policy.

## 0.6.2 Proxy Requirements

If servers use an outbound proxy, validate:

- Azure Arc agent proxy support and configuration.
- Azure Monitor Agent proxy support, if AMA is deployed.
- PowerShell `Invoke-RestMethod` proxy behavior.
- Whether proxy authentication is supported for the scheduled-task account.
- TLS inspection behavior.
- Certificate trust chain.
- Whether the proxy bypasses `localhost`, required for HIMDS access.
- Whether Azure Monitor ingestion endpoints are excluded from content modification.

## 0.6.3 Connectivity Acceptance Tests

Before collector deployment, test from every pilot server:

```powershell name=Test-CollectorConnectivity.ps1
$targets = @(
    'https://login.microsoftonline.com/',
    'https://management.azure.com/',
    '<Azure-Monitor-ingestion-endpoint-from-DCR-or-DCE>'
)

foreach ($target in $targets) {
    try {
        $result = Invoke-WebRequest `
            -Uri $target `
            -Method Head `
            -UseBasicParsing `
            -TimeoutSec 30 `
            -ErrorAction Stop

        Write-Output "SUCCESS: $target - HTTP $($result.StatusCode)"
    }
    catch {
        Write-Warning "FAILED: $target - $($_.Exception.Message)"
    }
}
```

An HTTP status such as `401`, `403`, or `404` can still prove network and TLS connectivity; success criteria must distinguish connectivity validation from authorization validation.

---

# 0.7 Azure Region and Workspace Strategy

## 0.7.1 Azure Region Selection Criteria

The region is currently undecided. Select it before Bicep deployment based on:

1. **Data residency and regulatory requirements**
2. **Corporate Azure landing-zone standard**
3. **Availability of Azure Arc, Azure Monitor, Log Analytics, and alerting features**
4. **Network latency from on-premises data centres**
5. **Business continuity and regional resilience policy**
6. **Existing Azure networking, identity, and operations footprint**
7. **Cost and billing alignment**

The DCR, DCE—if used—and Log Analytics Workspace must be designed with regional compatibility in mind. The final implementation will validate current service constraints for the selected region.

## 0.7.2 Workspace Recommendation

For the pilot, create a dedicated workspace:

```text
law-erpmon-pilot-<region>-001
```

Example only:

```text
law-erpmon-pilot-cac-001
```

### Why a dedicated pilot workspace

- Isolates custom schemas from enterprise production telemetry.
- Limits pilot access scope.
- Simplifies ingestion-cost measurement.
- Allows retention testing without affecting existing logs.
- Simplifies cleanup if the pilot does not proceed.
- Provides clear ownership and change-control boundaries.

After acceptance, production can use either:

- A dedicated ERP monitoring workspace, or
- An enterprise central workspace with strict RBAC, table governance, and cost allocation.

This should be an architecture decision at the end of the pilot.

---

# 0.8 Naming and Tagging Standards

## 0.8.1 Azure Resources

| Resource | Pattern | Example |
|---|---|---|
| Resource group | `rg-erpmon-<env>-<region>-001` | `rg-erpmon-pilot-cac-001` |
| Log Analytics Workspace | `law-erpmon-<env>-<region>-001` | `law-erpmon-pilot-cac-001` |
| Data Collection Endpoint | `dce-erpmon-<env>-<region>-001` | `dce-erpmon-pilot-cac-001` |
| Certificate DCR | `dcr-erpmon-cert-<env>-001` | `dcr-erpmon-cert-pilot-001` |
| Folder metrics DCR | `dcr-erpmon-folder-<env>-001` | `dcr-erpmon-folder-pilot-001` |
| Collector health DCR | `dcr-erpmon-health-<env>-001` | `dcr-erpmon-health-pilot-001` |
| Action Group | `ag-erpmon-ops-<env>-001` | `ag-erpmon-ops-pilot-001` |
| Workbook | `wb-erpmon-operations-<env>-001` | `wb-erpmon-operations-pilot-001` |

## 0.8.2 Required Tags

Apply at least:

| Tag | Example value |
|---|---|
| `Application` | `ERP-Monitoring` |
| `Environment` | `Pilot` |
| `Owner` | `InfrastructureOperations` |
| `CostCenter` | `<approved-cost-centre>` |
| `DataClassification` | `Internal` |
| `ManagedBy` | `Bicep` |
| `SupportTeam` | `ERP-Platform-Support` |
| `Criticality` | `High` |

---

# 0.9 Data Classification and Collection Guardrails

## Certificate Inventory

The certificate collector may collect:

- Subject.
- SAN/DNS names.
- Issuer.
- Thumbprint.
- Serial number.
- Validity dates.
- Private-key presence indicator.
- IIS binding association.
- Server name.

The collector must not collect:

- Private keys.
- PFX files.
- PFX passwords.
- Certificate export data.
- Any secret in IIS configuration.
- Authentication tokens.

## ERP Folder Metrics

The folder collector may collect:

- Folder path.
- Total bytes.
- File count.
- Oldest/newest timestamps.
- Files exceeding retention age.
- Access error count.
- Scan duration.
- Server and application identity.

The collector must not collect by default:

- File contents.
- Individual filenames.
- File hashes.
- Customer data.
- ERP transactions.
- Connection strings.
- Credentials.
- Exception output containing sensitive data.

If filenames are later required for troubleshooting, create a separate, explicitly approved diagnostic collector with narrow scope and short retention.

---

# 0.10 Initial Data Retention and Cost Controls

## Pilot Defaults

| Dataset | Collection frequency | Initial interactive retention |
|---|---:|---:|
| Certificate inventory | Daily | 90 days |
| ERP folder metrics | 15–60 minutes | 30–90 days |
| Collector health | Every execution | 90 days |
| Native Windows events/performance | Per baseline DCR | 30–90 days initially |

## Cost Controls

1. Collect certificate data daily, not every minute.
2. Collect ERP folder metrics only for approved paths.
3. Send aggregate folder metrics, not every filename.
4. Use a separate table for collector health.
5. Add `Environment`, `Application`, and `Computer` fields for cost and filtering analysis.
6. Review actual ingestion volume after seven and 30 days.
7. Set an ingestion budget/alert for the pilot workspace.
8. Avoid broad application log ingestion until the vendor-log format and data sensitivity are assessed.

---

# 0.11 Pilot Acceptance Criteria for Phase 0

Phase 0 is complete when the following are documented and approved:

| Area | Acceptance criterion |
|---|---|
| Azure region | Region selected and approved. |
| Workspace | Pilot Log Analytics Workspace strategy approved. |
| Azure Arc | Pilot servers identified and confirmed eligible for Arc onboarding. |
| Managed identity | Azure Arc system-assigned identity model approved. |
| Task Scheduler | Dedicated run-as account model approved. |
| Local permissions | Required local permissions and `Hybrid Agent Extension Applications` membership documented. |
| Network | Outbound HTTPS/proxy requirements approved and test plan agreed. |
| Data scope | WebHosting certificate store and ERP log directories identified. |
| Data governance | Explicit list of collected and prohibited data approved. |
| IaC | Bicep repository, environment parameterization, and deployment ownership agreed. |
| Alerting | Teams/email notification targets, severity model, and operational owner identified. |
| Operations | Initial support/runbook owner identified. |

---

# 0.12 Phase 0 Deliverables

The system-integration team should produce these artifacts:

1. **Pilot Architecture Diagram**
2. **Azure Resource Naming and Tagging Standard**
3. **Azure Arc Managed Identity Design**
4. **Task Scheduler Run-As Account Design**
5. **Network and Proxy Connectivity Matrix**
6. **Certificate and ERP Folder Monitoring Scope Matrix**
7. **Data Classification and Exclusion Register**
8. **RBAC Matrix**
9. **Pilot Resource Bill of Materials**
10. **Phase 0 Acceptance Checklist**
11. **Risk and Assumption Register**
12. **Bicep Repository Bootstrap Specification**

---

# 0.13 Risks to Record Now

| Risk | Impact | Mitigation |
|---|---|---|
| Arc managed identity cannot be accessed by scheduled-task account | Collectors cannot authenticate without a secret | Test HIMDS access early; use approved group membership and least privilege. |
| Certificate store requires elevated rights | Certificate collector cannot read required attributes/bindings | Test on one server; grant only necessary local permissions; document any elevation. |
| ERP log directory has very high file count | Long-running scans can impact server performance | Begin with hourly scans, capture scan duration, enforce timeout, exclude archives/reparse points. |
| Proxy prevents Arc/ingestion calls | No telemetry reaches Azure | Complete proxy validation before rollout. |
| Azure region remains undecided | IaC resource deployment blocked | Escalate region choice as a pilot gating decision. |
| Teams connector/notification pattern not approved | Alerts cannot reach support channel | Agree on Action Group + Logic App or approved enterprise mechanism before Phase 5. |
| Existing enterprise workspace is mandated | Table/RBAC/cost isolation becomes more complex | Adapt Bicep modules to deploy into existing workspace after governance review. |

---

# 0.14 Recommended Next Step

Proceed to **Phase 1 — PowerShell Collector Project and Windows Deployment Foundation**.

Phase 1 will define:

- Git/Bicep/PowerShell repository structure.
- The Windows package directory layout.
- Shared configuration, logging, collector-health, locking, and ingestion modules.
- Local folder and NTFS permission model.
- Script-signing approach.
- Deployment, update, and rollback procedures.
- Task Scheduler installation scripts for certificate and ERP folder collectors.
