# Execution Plan for the Implementation Tutorial

Yes—this is a substantial implementation guide. I recommend delivering it in **six phased replies** so each phase is reviewable, technically coherent, and can be validated before moving to the next.

The implementation will use two concrete collectors:

1. **Windows Certificate Inventory and Expiration Monitoring**
2. **Windows Disk-Space and ERP Folder-Growth Monitoring**

The architecture is:

```text
On-premises Windows Server
   ├── Azure Arc / Azure Monitor Agent for native telemetry
   └── Custom PowerShell collector for non-native telemetry
            │
            │ Microsoft Entra authentication + outbound HTTPS
            ▼
Azure Monitor Logs Ingestion API
            │
            ▼
Data Collection Endpoint (when required by selected network design)
            │
            ▼
Data Collection Rule
            │
            ▼
Log Analytics Workspace custom tables
            │
            ├── KQL queries
            ├── Azure Workbooks
            └── Azure Monitor scheduled-query alerts
                    │
                    ▼
        ServiceNow / Teams / Outlook via Action Groups and Logic Apps
```

## Phase 0 — Architecture Decisions and Prerequisites

**Purpose:** Establish decisions that must be approved before implementation begins.

Contents:

- Pilot architecture and component responsibility matrix.
- Azure Arc, AMA, DCR, DCE, Log Analytics, and custom collector boundaries.
- Required Azure resources.
- Region, network, proxy, private-link, and firewall decisions.
- Microsoft Entra authentication choice:
  - Managed identity, where feasible.
  - Certificate-based service principal otherwise.
- Azure RBAC/least-privilege matrix.
- Required Windows Server, PowerShell, and module prerequisites.
- Data classification and exclusions.
- Naming, tagging, retention, and environment standards.
- Acceptance criteria and controlled test cases.

**Output:** Approved technical design and readiness checklist.

---

## Phase 1 — PowerShell Project and Windows Deployment Foundation

**Purpose:** Create a maintainable collector codebase and deployment model.

Contents:

- Source-control repository structure.
- Module, collector, configuration, test, deployment, KQL, and IaC folders.
- Shared PowerShell ingestion module.
- Shared logging, configuration, error handling, locking, retry, and health-reporting functions.
- Code-signing strategy.
- Local folder structure on managed servers.
- Windows Task Scheduler deployment model.
- Alternative Azure Automation Hybrid Worker model.
- Execution identity and NTFS permissions.
- Deployment, upgrade, rollback, and validation process.
- Collector configuration-file model.

**Output:** A deployable collector package skeleton and Windows deployment runbook.

---

## Phase 2 — Azure Monitor Custom Ingestion Foundation: Workspace, Tables, DCE, DCR, and RBAC

**Purpose:** Build the Azure-side custom-telemetry data platform.

Contents:

- Log Analytics Workspace setup.
- Custom-table design for:
  - `ServerCertificateInventory_CL`
  - `WindowsDiskMetrics_CL`
  - `ErpFolderMetrics_CL`
  - `CollectorHealth_CL`
- JSON payload and typed schema definitions.
- Data Collection Endpoint design:
  - Public ingestion versus Private Link.
  - When DCE is required.
- Data Collection Rule stream declarations and data flows.
- DCR transformations.
- DCR immutable ID and endpoint handling.
- Custom collector identity and role assignment.
- Infrastructure-as-code pattern using Bicep/ARM structure.
- Direct API test payload validation.

**Output:** Deployed custom tables, DCRs, optional DCE, RBAC assignments, and validated ingestion test.

---

## Phase 3 — Certificate Monitoring Collector: Build, Deploy, Ingest, and Validate

**Purpose:** Implement the first complete reference collector.

Contents:

- Certificate monitoring functional design.
- Target certificate stores and exclusions.
- IIS HTTPS binding association.
- Certificate metadata collection.
- Duplicate and stale certificate handling.
- Private-key presence checks without exporting private keys.
- Certificate collector PowerShell implementation.
- JSON payload generation.
- Logs Ingestion API integration.
- Job scheduling and collector health.
- Local logging, retry, and failure handling.
- Validation cases:
  - Normal certificate.
  - Expired certificate.
  - Near-expiry certificate.
  - Missing private key.
  - Missing store/access denied.
  - Ingestion failure.

**Output:** Operational certificate collector writing to `ServerCertificateInventory_CL`.

---

## Phase 4 — Disk and Folder Monitoring Collector: Build, Deploy, Ingest, and Validate

**Purpose:** Implement the second complete reference collector.

Contents:

- Disk-monitoring functional design.
- Drive capacity, absolute/percentage free space, and volume labels.
- Disk latency and queue indicators.
- ERP/IIS/middleware/SQL/backup folder inventory.
- Directory scanning safeguards:
  - Reparse points.
  - Access denied.
  - Scan duration.
  - High file count.
  - Exclusions.
- Folder-growth and retention-age metrics.
- Disk/folder collector PowerShell implementation.
- Separate disk and folder JSON payloads.
- Logs Ingestion API integration.
- Collection schedule, overlap protection, and health monitoring.
- Test and validation scenarios.

**Output:** Operational disk and folder collectors writing to `WindowsDiskMetrics_CL` and `ErpFolderMetrics_CL`.

---

## Phase 5 — KQL, Workbooks, Alerts, ITSM Integration, and Production Readiness

**Purpose:** Turn stored telemetry into operational monitoring and support workflows.

Contents:

- KQL queries for certificate and disk monitoring.
- Data freshness and collector-health queries.
- Azure Workbook dashboard design:
  - Certificate expiry dashboard.
  - Disk and capacity dashboard.
- Scheduled-query alert-rule design.
- Warning/critical threshold model.
- Common Alert Schema.
- Action Groups.
- ServiceNow incident creation/update through Logic Apps or approved connector.
- Teams and Outlook notification flow.
- Alert correlation, suppression, deduplication, and auto-resolution.
- Controlled fault injection.
- Cost model and retention tuning.
- Production hardening, operational runbooks, and rollout gates.

**Output:** Dashboards, alerts, ITSM notification workflow, test report, and production implementation checklist.

---

# Reference Validation Notes

The tutorial will use the current Azure approach—not the legacy Log Analytics HTTP Data Collector API:

- **Azure Arc-enabled servers** require the Connected Machine agent and applicable outbound HTTPS access; AMA is deployed as an Arc extension and applies DCR-based collection.[[1]](https://learn.microsoft.com/en-us/azure/azure-arc/servers/prerequisites)
- For custom ingestion, the modern model is: **custom table → DCR stream declaration/data flow → optional DCE depending on networking → Logs Ingestion API → DCR immutable ID**, with the collector identity granted the appropriate ingestion permission on the DCR.[[2]](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/tutorial-logs-ingestion-api)[[3]](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/logs-ingestion-api-overview)
- Azure Monitor scheduled-query alerts invoke Action Groups; Action Groups can invoke Logic Apps for ServiceNow and Teams workflows and send email notifications.[[4]](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/action-groups)[[5]](https://learn.microsoft.com/en-us/cli/azure/monitor/scheduled-query?view=azure-cli-latest)

---

# Required Decisions Before Phase 0

I need these answers before producing the detailed implementation:

1. **Azure tenant and region:** Which Azure region will host the Log Analytics Workspace and DCR/DCE resources?  
2. **Networking:** Is outbound public HTTPS access allowed from the on-premises servers, or must Azure Monitor ingestion use Private Link/private endpoints?  
3. **Collector execution model:** For the pilot, do you prefer:
   - Windows Task Scheduler running locally on each monitored server, or
   - Azure Automation Hybrid Runbook Worker, or
   - a combination?  
4. **Authentication:** Is a Microsoft Entra application/service principal with a certificate credential acceptable for custom ingestion, or must the pilot use managed identity through Azure Automation where possible?  
5. **Infrastructure as code:** Should the tutorial use **Bicep**, ARM JSON, Azure CLI/PowerShell, or a portal-first pilot followed by Bicep?  
6. **ServiceNow:** Is the organization using ServiceNow now, or should the initial pilot deliver Teams and Outlook only, with ServiceNow integration designed but not activated?  
7. **Certificate scope:** Should the initial reference collector cover only `Cert:\LocalMachine\My` and `Cert:\LocalMachine\WebHosting`, plus IIS bindings?  
8. **Disk scope:** Which drive and directory categories are initially in scope: ERP logs, IIS logs, middleware queues/archives, SQL data/log/tempdb, and SQL backup directories?  

Once these are answered, I will begin with **Phase 0: Architecture Decisions and Prerequisites**.


---
---


# Confirmed Decisions and Important Technical Adjustment

Your choices define the following pilot baseline:

| Area | Decision |
|---|---|
| Azure region | **Undecided** — must be selected before Azure resources are deployed. |
| Azure connectivity | On-premises servers may use **outbound public HTTPS/TCP 443**. |
| Job scheduler | **Windows Task Scheduler** running locally on each monitored Windows server. |
| Authentication | **Managed identity** is required. |
| Infrastructure as code | Use **Bicep** as the primary Azure IaC language. |
| Alert notification | **Microsoft Teams and Outlook/email** for the initial pilot. ServiceNow is deferred. |
| Certificate scope | `Cert:\LocalMachine\WebHosting` only. |
| Disk/folder scope | ERP log directories only. |

## Important adjustment: Managed Identity + Windows Task Scheduler

Your choices are compatible **if every collector host is Azure Arc-enabled and has its Azure Arc system-assigned managed identity enabled**.

The local Task Scheduler job will run the PowerShell collector. The collector will authenticate by requesting an Azure token from the Azure Arc **Hybrid Instance Metadata Service (HIMDS)** local endpoint, rather than storing a service-principal secret or certificate.

```text
Windows Task Scheduler
       │
       ▼
PowerShell collector
       │
       │ Local-only token request
       ▼
Azure Arc HIMDS endpoint on localhost
       │
       │ Azure Arc system-assigned managed identity
       ▼
Microsoft Entra ID token
       │
       ▼
Azure Monitor Logs Ingestion API
       │
       ▼
DCR → Log Analytics custom table
```

This is not Azure Automation managed identity. Azure Automation Hybrid Runbook Worker would be the natural design if Azure Automation were the chosen scheduler. Since the pilot uses Windows Task Scheduler, we will use the **Azure Arc server identity**.

Microsoft’s Azure Arc managed-identity guidance indicates that Arc-enabled servers can request a token through the local HIMDS endpoint; local access is restricted to local Administrators or the **Hybrid Agent Extension Applications** group.[[1]](https://learn.microsoft.com/en-us/azure/azure-arc/servers/managed-identity-authentication)

### Pilot identity requirement

For each collector host:

1. Onboard the Windows server to Azure Arc.
2. Enable its **system-assigned managed identity**.
3. Configure the scheduled-task run-as account to access the Arc local token endpoint:
   - Prefer a dedicated local service identity.
   - Add it to the local **Hybrid Agent Extension Applications** group, subject to security approval.
   - Do not run the task as an interactive administrator account.
4. Assign only the required Azure role for ingestion at the DCR scope.
5. Confirm that the PowerShell process can obtain a token for:

```text
https://monitor.azure.com/
```

6. Confirm it can submit data only through the required certificate/disk DCR streams.

This eliminates storage of Azure client secrets and private authentication certificates on the ERP and web servers.

---

# Revised Phased Implementation Plan

Given the confirmed pilot decisions, I will use the following sequence.

## Phase 0 — Architecture, Identity, and Readiness

I will cover:

- Pilot component architecture.
- Azure region selection criteria.
- Public outbound HTTPS endpoint and proxy/firewall matrix.
- Azure Arc onboarding requirements.
- Arc system-assigned managed identity configuration.
- Windows Task Scheduler execution identity and local-group requirements.
- Bicep repository and environment strategy.
- Naming, tagging, retention, and data-classification standards.
- Pilot acceptance tests.

## Phase 1 — PowerShell Collector Project and Windows Deployment

I will cover:

- Full repository structure.
- Package structure deployed to each server.
- Shared module design.
- Certificate collector and ERP log-directory collector projects.
- Local configuration model.
- Task Scheduler job definitions.
- Logging, locking, retries, timeouts, and health reporting.
- Code signing and NTFS permissions.
- Deployment and rollback runbook.

## Phase 2 — Azure Infrastructure as Code

I will provide Bicep architecture and implementation for:

- Resource group structure.
- Log Analytics Workspace.
- `ServerCertificateInventory_CL`.
- `WindowsDiskMetrics_CL`.
- `ErpFolderMetrics_CL`.
- `CollectorHealth_CL`.
- DCRs, streams, transformations, and workspace mappings.
- DCE decision: omitted initially for public ingestion unless validation shows it is required by the final API design.
- DCR-scope managed identity role assignments.
- Action Groups for Teams and Outlook/email.
- Scheduled-query alert rules.
- Workbook deployment structure.

## Phase 3 — Certificate Monitoring Reference Implementation

I will provide:

- Certificate collector implementation.
- Azure Arc managed-identity token function.
- Logs Ingestion API sender.
- JSON payload schema.
- Local Task Scheduler deployment.
- Collector health reporting.
- KQL validation, certificate inventory, expiry, and freshness queries.
- Workbook visualization structure.
- Warning/critical alerts and Teams/email notification flow.

## Phase 4 — Disk and ERP Log Directory Monitoring Reference Implementation

I will provide:

- Disk capacity collector.
- ERP log-directory collector.
- Guardrails for reparse points, access failures, scan duration, high file counts, and concurrency.
- JSON payload schemas.
- KQL queries for drive capacity, rapid growth, old files, and collector health.
- Dashboards and alert rules.

## Phase 5 — Testing, Operations, and Production Readiness

I will provide:

- Fault-injection test plan.
- Security hardening.
- Cost and retention model.
- Operational support model.
- Collector and monitoring-platform health.
- Upgrade and schema versioning.
- Release pipeline.
- Production rollout waves and go/no-go gates.

---

# Remaining Decisions Needed Before Azure Deployment

I can begin **Phase 0** now. Before any actual resource deployment, these items must be decided:

1. **Azure region**  
   Select the region based on data residency, latency, organizational standard, Azure Monitor/Arc availability, and disaster-recovery policy. For a Canadian organization, likely candidates may be Canada Central or Canada East, but this needs organization approval.

2. **Log Analytics Workspace ownership**  
   Confirm whether the pilot uses:
   - A new dedicated pilot workspace, recommended; or
   - An existing central enterprise workspace.

3. **Task Scheduler run-as account**  
   Confirm whether the organization permits:
   - A dedicated local service account, or
   - `LocalSystem`.

   My recommendation is a dedicated local service account with only:
   - Read access to `Cert:\LocalMachine\WebHosting`.
   - Read/list access to approved ERP log directories.
   - Write access to the collector’s local log/spool folder.
   - Membership in **Hybrid Agent Extension Applications** to use the Arc managed identity.
   - No local-administrator access unless the certificate-store/IIS-binding access test requires it.

4. **ERP log paths**  
   Provide the pilot log directory path or paths, for example:

```text
D:\ERP\Logs
E:\ERP\ApplicationLogs
```

I will proceed with **Phase 0 — Architecture, Identity, and Readiness** after you confirm the workspace approach and the scheduled-task identity approach.
