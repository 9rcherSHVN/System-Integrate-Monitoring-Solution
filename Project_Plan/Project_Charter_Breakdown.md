# Project Charter  
## Hybrid Azure Monitor Pilot for On-Premises Windows ERP Platform

**Project name:** Hybrid ERP Infrastructure and Application Monitoring Pilot  
**Project sponsor:** To be assigned  
**Project manager:** To be assigned  
**Technical lead:** System Integration Lead  
**Document status:** Draft for approval  
**Target delivery:** Pilot timeline to be confirmed after Azure region, resource ownership, and pilot-server availability are approved.

---

## 1. Project Purpose

Implement and validate a hybrid, cloud-managed monitoring solution for on-premises Windows Server workloads hosting third-party ERP application services, web/middleware services, and supporting infrastructure.

The solution will use Azure Arc, Azure Monitor, Log Analytics, Data Collection Rules, Azure Monitor alerts, Workbooks, Microsoft Teams, and Outlook/email. Windows workloads remain on-premises; this project does not migrate applications or servers to Azure virtual machines.

The initial pilot focuses on:

1. **Windows certificate inventory and expiration monitoring**
2. **Windows disk capacity monitoring**
3. **ERP log-folder capacity, growth, retention-age, and scan-health monitoring**
4. **Custom PowerShell data collection with Azure Arc managed identity**
5. **Azure Monitor dashboards, KQL, alerts, email, and Teams notifications**

---

## 2. Business Problem

IT support currently requires a practical, integrated capability to monitor and troubleshoot the runtime health of on-premises ERP-related Windows servers.

The identified operational gaps include:

- Limited centralized visibility of Windows server health.
- Risk of certificate expiration causing ERP/API outage.
- Risk of disk exhaustion caused by ERP log growth.
- Insufficient historical tracking of disk capacity and log-folder growth.
- Limited cloud-based alerting and dashboard visibility.
- Manual troubleshooting across Windows, IIS, certificates, file systems, and application support teams.
- Lack of a reusable, secure custom telemetry pattern for data that cannot be collected natively.

---

## 3. Project Objectives

| ID | Objective |
|---|---|
| OBJ-01 | Validate Azure Arc and Azure Monitor as a feasible hybrid monitoring platform for on-premises Windows ERP workloads. |
| OBJ-02 | Deploy secure custom PowerShell collectors using Azure Arc system-assigned managed identities. |
| OBJ-03 | Monitor certificates stored in `Cert:\LocalMachine\WebHosting`, including expiration and IIS-binding context. |
| OBJ-04 | Monitor Windows logical-disk capacity and performance using AMA where approved, or custom PowerShell collection where AMA is unavailable. |
| OBJ-05 | Monitor ERP log-folder size, file count, growth, retention-age, and collection quality. |
| OBJ-06 | Centralize telemetry in Azure Log Analytics Workspace custom tables. |
| OBJ-07 | Provide KQL queries and Azure Workbooks for operational troubleshooting. |
| OBJ-08 | Generate actionable Azure Monitor alerts routed to Outlook/email and Microsoft Teams. |
| OBJ-09 | Validate collector health, retry, spool, alert resolution, rollback, and operational recovery. |
| OBJ-10 | Produce a production-readiness recommendation and reusable implementation pattern. |

---

## 4. Scope

## 4.1 In Scope

| Area | Included capability |
|---|---|
| Hybrid management | Azure Arc onboarding and system-assigned managed identity for pilot servers |
| Azure monitoring | Log Analytics Workspace, DCE, DCRs, custom tables, KQL, alerts, Workbooks |
| Certificate monitoring | `Cert:\LocalMachine\WebHosting`, expiry, private-key status, IIS binding correlation |
| Disk monitoring | Logical-disk free space, used/free capacity, latency, queue length, activity rate |
| ERP folder monitoring | Approved ERP log-directory size, growth, file count, retention-age, access errors, scan timeout |
| Collector platform | Scheduled PowerShell collectors, local locking, retry, spool, health telemetry |
| Alerting | Azure Monitor scheduled query rules, Action Groups, Outlook/email |
| Teams integration | Action Group to Logic App to approved Teams channel |
| Security | Managed identity, DCR-scoped RBAC, script signing, NTFS ACLs, data classification |
| Validation | Fault injection, recovery testing, cost/performance assessment, production-readiness assessment |
| Documentation | Architecture, runbooks, operational handover, deployment/rollback procedures |

## 4.2 Out of Scope

| Area | Exclusion |
|---|---|
| Server migration | Moving on-premises servers to Azure VMs |
| ERP replacement or modernization | Changes to third-party ERP software architecture |
| Full SQL monitoring | SQL DMV, backup, job, wait-stat, and blocking monitoring are future phases |
| Full APM | Source-code instrumentation or distributed tracing inside third-party ERP applications |
| Packet capture | Deep network packet capture or inspection |
| ServiceNow production integration | Deferred; only future design compatibility is required |
| Enterprise SIEM rollout | Microsoft Sentinel/SOC implementation |
| Certificate authority redesign | PKI replacement or enterprise certificate automation redesign |
| Service-account password remediation | gMSA migration or broad identity redesign |
| Broad application-log ingestion | ERP/IIS/middleware log collection beyond approved future scope |

---

## 5. Solution Overview

```text
On-Premises Windows Server
    │
    ├── Azure Arc Connected Machine Agent
    ├── Azure Arc System-Assigned Managed Identity
    ├── Azure Monitor Agent, where approved
    ├── Windows Task Scheduler
    └── Signed PowerShell Collectors
           │
           ├── Certificate inventory
           ├── ERP log-folder metrics
           └── Custom logical-disk metrics, if AMA is not approved
           │
           ▼
Azure Monitor Logs Ingestion API
           │
           ▼
Data Collection Endpoint / Data Collection Rules
           │
           ▼
Log Analytics Workspace
    ├── ServerCertificateInventory_CL
    ├── ErpFolderMetrics_CL
    ├── WindowsDiskMetrics_CL
    └── CollectorHealth_CL
           │
           ├── KQL
           ├── Azure Monitor Workbooks
           └── Scheduled Query Alerts
                    │
                    ▼
Action Group
    ├── Outlook/email
    └── Logic App → Microsoft Teams
```

---

## 6. Deliverables

| ID | Deliverable | Acceptance owner |
|---|---|---|
| DEL-01 | Approved architecture and security design | System Solution Architect |
| DEL-02 | Azure resource naming/tagging/RBAC standard | Azure Monitoring Lead |
| DEL-03 | Azure Arc onboarding and managed-identity validation | Windows Infrastructure Lead |
| DEL-04 | Signed PowerShell collector package | System Integration Lead |
| DEL-05 | Scheduled Task deployment and rollback scripts | Windows Infrastructure Lead |
| DEL-06 | Bicep infrastructure-as-code package | System Integration Lead |
| DEL-07 | Log Analytics Workspace, DCE, DCRs, and custom tables | Azure Monitoring Lead |
| DEL-08 | Certificate monitoring collector and custom table | ERP/Windows Support |
| DEL-09 | Disk and ERP log-folder collector and custom tables | ERP/Windows Support |
| DEL-10 | KQL query catalogue | Azure Monitoring Lead |
| DEL-11 | Certificate Workbook | Operations Support Lead |
| DEL-12 | Disk and ERP log-folder Workbook | Operations Support Lead |
| DEL-13 | Alert rules, email Action Group, and Teams Logic App | Azure Monitoring Lead |
| DEL-14 | Fault-injection test report | System Integration Lead |
| DEL-15 | Operational runbooks | Operations Support Lead |
| DEL-16 | Cost/performance assessment | Project Sponsor / Monitoring Lead |
| DEL-17 | Production-readiness assessment | System Solution Architect |

---

# 7. Work-Breakdown Structure

## 7.1 WBS Summary

```text
1.0 Project Governance and Planning
2.0 Architecture, Security, and Readiness
3.0 Azure Foundation and Infrastructure as Code
4.0 Windows Collector Platform
5.0 Certificate Monitoring Implementation
6.0 Disk and ERP Folder Monitoring Implementation
7.0 Dashboards, Alerts, and Notification Integration
8.0 Testing and Fault Injection
9.0 Operations Handover and Production Readiness
10.0 Pilot Closure and Rollout Recommendation
```

---

## 7.2 Detailed WBS

### 1.0 Project Governance and Planning

| WBS | Work package | Key activities | Deliverable |
|---|---|---|---|
| 1.1 | Project initiation | Confirm sponsor, technical lead, stakeholders, pilot goals | Approved project charter |
| 1.2 | Scope management | Confirm in-scope servers, certificate store, ERP log paths, notification channels | Scope register |
| 1.3 | Project controls | Establish risks, issues, change control, status reporting | RAID log and reporting process |
| 1.4 | Stakeholder engagement | Schedule architecture, security, operations, ERP, and network workshops | Stakeholder plan |

### 2.0 Architecture, Security, and Readiness

| WBS | Work package | Key activities | Deliverable |
|---|---|---|---|
| 2.1 | Pilot architecture | Define Arc, DCE, DCR, LAW, collectors, alerts, Workbooks | Architecture diagram |
| 2.2 | Server/dependency discovery | Identify pilot servers, roles, owners, certificates, log paths, disk volumes | Asset/dependency inventory |
| 2.3 | Azure region decision | Confirm region, data residency, workspace ownership | Region decision record |
| 2.4 | Network readiness | Validate outbound HTTPS, proxy, DNS, firewall, TLS inspection | Network connectivity matrix |
| 2.5 | Identity design | Define Azure Arc system identity, local collector account, DCR RBAC | Identity/RBAC matrix |
| 2.6 | Security review | Confirm data classification, prohibited fields, logging controls, script signing | Security assessment |
| 2.7 | Operations readiness | Define Tier 1/Tier 2 ownership and escalation | Operations model |

### 3.0 Azure Foundation and Infrastructure as Code

| WBS | Work package | Key activities | Deliverable |
|---|---|---|---|
| 3.1 | Bicep repository | Create repository, module structure, parameter files, deployment scripts | IaC repository |
| 3.2 | Resource group/workspace | Deploy pilot RG and Log Analytics Workspace | Pilot workspace |
| 3.3 | Custom tables | Deploy certificate, folder, disk, and collector-health tables | Custom tables |
| 3.4 | DCE | Deploy Data Collection Endpoint | DCE endpoint |
| 3.5 | DCRs | Deploy certificate, folder, disk, and health DCRs | DCR resources |
| 3.6 | DCR RBAC | Assign DCR-scoped monitoring publisher permissions | Role assignments |
| 3.7 | Deployment validation | Execute Bicep build, validate, what-if, deployment | Deployment evidence |

### 4.0 Windows Collector Platform

| WBS | Work package | Key activities | Deliverable |
|---|---|---|---|
| 4.1 | Collector code foundation | Shared logging, locking, configuration, HIMDS identity, ingestion module | Shared PowerShell module |
| 4.2 | Package management | Build signed package, release manifest, installer, rollback, uninstall | Collector package |
| 4.3 | Local environment | Create folder structure, NTFS ACLs, log/spool/lock folders | Deployment standard |
| 4.4 | Collector account | Provision task account, batch-job right, HIMDS authorization | Validated account |
| 4.5 | Task Scheduler | Register collector and spool-replay tasks | Scheduled tasks |
| 4.6 | Health monitoring | Implement collector-health records and missing-data model | Health telemetry |

### 5.0 Certificate Monitoring Implementation

| WBS | Work package | Key activities | Deliverable |
|---|---|---|---|
| 5.1 | Certificate collector | Implement WebHosting store enumeration and metadata extraction | Signed certificate collector |
| 5.2 | IIS correlation | Implement optional HTTPS binding correlation | Binding data |
| 5.3 | Ingestion test | Validate Arc identity, DCE, DCR, payload, table routing | Test record evidence |
| 5.4 | KQL | Build inventory, expiry, private-key, stale-data, health queries | KQL catalogue |
| 5.5 | Alerts | Deploy expiry and collector-health rules | Certificate alert rules |
| 5.6 | Workbook | Deploy certificate dashboard | Certificate Workbook |
| 5.7 | Runbook | Define expiry/replacement/binding response | Certificate response runbook |

### 6.0 Disk and ERP Folder Monitoring Implementation

| WBS | Work package | Key activities | Deliverable |
|---|---|---|---|
| 6.1 | Disk collection decision | Confirm AMA or custom collector approach | Decision record |
| 6.2 | AMA DCR, if used | Configure logical-disk counters and Arc association | AMA disk DCR |
| 6.3 | Custom disk collector, if used | Implement capacity/performance collector | Signed disk collector |
| 6.4 | ERP folder collector | Implement aggregate folder size/growth/retention scan | Signed folder collector |
| 6.5 | Ingestion validation | Validate disk/folder table records | Test evidence |
| 6.6 | KQL | Build capacity, latency, growth, retention, health queries | KQL catalogue |
| 6.7 | Alerts | Deploy capacity, growth, retention, and health alerts | Disk/folder alerts |
| 6.8 | Workbook | Deploy disk/log-folder dashboard | Disk/folder Workbook |
| 6.9 | Runbook | Define capacity, cleanup, retention, and escalation procedures | Capacity response runbook |

### 7.0 Dashboards, Alerts, and Notification Integration

| WBS | Work package | Key activities | Deliverable |
|---|---|---|---|
| 7.1 | Email Action Group | Configure approved distribution-list notification | Email notification test |
| 7.2 | Teams Logic App | Create Common Alert Schema workflow and Adaptive Card | Teams workflow |
| 7.3 | Alert correlation | Configure mute periods, severities, auto-mitigation | Alert design register |
| 7.4 | Workbook usability | Validate dashboard with support teams | Usability feedback |
| 7.5 | Notification acceptance | Test activation and resolution notification | Notification test report |

### 8.0 Testing and Fault Injection

| WBS | Work package | Key activities | Deliverable |
|---|---|---|---|
| 8.1 | Test plan | Define functional, security, fault, and recovery tests | Approved test plan |
| 8.2 | Certificate tests | Expiry, expired, private key, collector failure, stale data | Test evidence |
| 8.3 | Disk/folder tests | Capacity, growth, retention, timeout, access error | Test evidence |
| 8.4 | Ingestion tests | Network interruption, RBAC failure, HIMDS failure, invalid payload | Test evidence |
| 8.5 | Alert tests | Email, Teams, suppression, auto-resolution | Test evidence |
| 8.6 | Rollback tests | Collector and Bicep rollback | Recovery evidence |
| 8.7 | Performance/cost tests | Runtime, CPU, memory, ingestion volume, spool size | Assessment report |

### 9.0 Operations Handover and Production Readiness

| WBS | Work package | Key activities | Deliverable |
|---|---|---|---|
| 9.1 | Runbook review | Tier 1/Tier 2 review and tabletop exercises | Approved runbooks |
| 9.2 | Support training | Train operations team on Workbooks, alerts, and escalation | Training record |
| 9.3 | Cost review | Measure 7-day and 30-day cost/ingestion | Cost report |
| 9.4 | Security review | Final review of RBAC, signing, accounts, data, spool | Security sign-off |
| 9.5 | Production readiness | Complete assessment and remaining risks | Readiness report |

### 10.0 Pilot Closure and Rollout Recommendation

| WBS | Work package | Key activities | Deliverable |
|---|---|---|---|
| 10.1 | Pilot evaluation | Compare objectives to actual results | Pilot outcome report |
| 10.2 | Backlog | Document enhancements, gaps, and deferred use cases | Phase-two backlog |
| 10.3 | Rollout recommendation | Define rollout waves and go/no-go decision | Rollout plan |
| 10.4 | Closure | Obtain acceptance/sign-off | Closure approval |

---

# 8. Milestone Plan

The schedule below is an indicative **10-week pilot**. Adjust after confirming resource availability, security approvals, Azure region, and ERP maintenance windows.

| Milestone | Target week | Entry criteria | Exit criteria |
|---|---:|---|---|
| M0 — Project Charter Approved | Week 1 | Sponsor and stakeholders identified | Scope, objectives, roles, and governance approved |
| M1 — Architecture and Readiness Approved | Week 2 | Pilot servers and owners identified | Region, network, identity, security, and workspace strategy approved |
| M2 — Azure Foundation Deployed | Week 3 | Bicep repository available | Workspace, tables, DCE, DCRs, RBAC, Action Group deployed |
| M3 — Collector Platform Installed | Week 4 | Arc identity and local account approved | Signed package, NTFS ACLs, Scheduled Tasks, token test complete |
| M4 — Certificate Monitoring Operational | Week 5 | Certificate DCR/table/collector deployed | Inventory, KQL, Workbook, alerts, email/Teams test successful |
| M5 — Disk and ERP Folder Monitoring Operational | Week 6 | Disk/folder design approved | Capacity, growth, retention, dashboard, alerts functioning |
| M6 — Fault Injection Complete | Week 7 | Monitoring features deployed | Required fault, recovery, spool, RBAC, and notification tests passed |
| M7 — Operations Handover Complete | Week 8 | Runbooks and dashboards available | Tier 1/Tier 2 training and tabletop exercises complete |
| M8 — Cost and Performance Review Complete | Week 9 | At least one week of telemetry collected | Cost forecast and performance impact accepted |
| M9 — Production Readiness Decision | Week 10 | Test and operational evidence complete | Go/no-go decision, risks, rollout waves, backlog approved |

---

# 9. Critical Path

```text
Azure region/workspace decision
    ↓
Azure Arc identity and network readiness
    ↓
Bicep deployment of workspace/DCE/DCR/tables/RBAC
    ↓
Collector configuration with DCR outputs
    ↓
Scheduled Task account and HIMDS validation
    ↓
Certificate collector and ingestion validation
    ↓
Disk/folder collector and ingestion validation
    ↓
Alerts, email, Teams Logic App, Workbooks
    ↓
Fault injection and recovery testing
    ↓
Operations handover and cost review
    ↓
Production readiness decision
```

The main schedule risks are:

- Azure region approval delay.
- Network/proxy endpoint approval delay.
- Azure Arc managed identity/local HIMDS access issues.
- Service-account approval and local rights delay.
- ERP log-directory ownership/retention policy uncertainty.
- Teams Logic App/connector approval delay.
- Lack of isolated non-production fault-test environment.

---

# 10. RACI Matrix

## Role Key

| Code | Role |
|---|---|
| SA | System Solution Architect |
| SI | System Integration Lead |
| PM | Project Manager |
| WIN | Windows Infrastructure Lead |
| ERP | ERP Application Support Lead |
| AZM | Azure Monitoring Lead |
| SEC | Security/IAM Lead |
| NET | Network Lead |
| OPS | Operations Support Lead |
| DB | Database Lead |
| SPO | Project Sponsor / Budget Owner |

**R = Responsible**  
**A = Accountable**  
**C = Consulted**  
**I = Informed**

| Activity | SA | SI | PM | WIN | ERP | AZM | SEC | NET | OPS | DB | SPO |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| Approve project charter | C | C | R | I | I | I | I | I | I | I | A |
| Manage project plan, issues, and status | I | C | A/R | I | I | I | I | I | I | I | I |
| Define pilot architecture | A | R | C | C | C | C | C | C | C | C | I |
| Approve technical architecture | A | R | C | C | C | C | C | C | C | C | I |
| Confirm pilot servers and ERP log paths | C | C | I | C | A/R | I | I | I | I | C | I |
| Select Azure region and workspace strategy | A | R | C | I | I | C | C | I | I | I | C |
| Validate network/proxy/firewall requirements | C | C | I | C | I | C | I | A/R | I | I | I |
| Azure Arc onboarding | I | C | I | A/R | I | C | C | C | I | I | I |
| Enable Arc system-assigned identity | I | C | I | A/R | I | C | C | I | I | I | I |
| Provision Scheduled Task service account | I | C | I | A/R | I | I | C | I | I | I | I |
| Approve local account rights/HIMDS access | I | C | I | R | I | I | A | I | I | I | I |
| Design Azure RBAC | C | R | I | C | I | C | A | I | I | I | I |
| Deploy Bicep Azure resources | I | R | I | I | I | A | C | I | I | I | I |
| Create/maintain DCRs and tables | C | R | I | I | I | A | C | I | I | I | I |
| Develop collector package | C | A/R | I | C | C | C | C | I | I | I | I |
| Code-sign collector package | I | R | I | C | I | I | A | I | I | I | I |
| Deploy collector package to Windows servers | I | R | I | A | I | C | C | I | I | I | I |
| Register Scheduled Tasks | I | C | I | A/R | I | I | C | I | I | I | I |
| Validate certificate collector | C | R | I | C | A | C | I | I | C | I | I |
| Validate disk/folder collector | C | R | I | C | A | C | I | I | C | I | I |
| Build KQL queries | C | R | I | I | C | A | I | I | C | C | I |
| Build Azure Workbooks | C | R | I | I | C | A | I | I | C | C | I |
| Configure alerts | C | R | I | I | C | A | I | I | C | I | I |
| Configure email notification | I | C | I | I | I | A/R | I | I | C | I | I |
| Configure Teams Logic App | I | R | I | I | I | A | C | I | C | I | I |
| Conduct fault injection | C | A/R | I | C | C | C | C | C | C | C | I |
| Approve testing outcomes | A | R | C | C | C | C | C | C | C | C | I |
| Create operational runbooks | C | R | I | C | C | C | I | I | A | C | I |
| Train operations team | I | R | C | C | C | C | I | I | A | I | I |
| Review cost and ingestion | C | R | I | I | I | A | I | I | C | I | C |
| Approve production readiness | A | R | C | C | C | C | C | C | C | C | I |
| Approve production funding/rollout | C | C | R | I | I | I | I | I | I | I | A |

---

# 11. RAID Register Template

## 11.1 Risks

| ID | Risk | Impact | Probability | Owner | Mitigation |
|---|---|---|---|---|---|
| R-01 | Azure region decision delayed | Delays all Azure resource deployment | Medium | SA / SPO | Escalate as Week 1 gating decision |
| R-02 | Arc HIMDS unavailable to task account | Custom collectors cannot authenticate | Medium | WIN / SEC | Validate local group membership and token request early |
| R-03 | Proxy blocks DCE/Entra/Azure Arc traffic | No telemetry ingestion | Medium | NET | Pre-approve endpoint matrix and test from pilot hosts |
| R-04 | ERP log directory has excessive file count | Long-running scans/server impact | Medium | ERP / SI | Start hourly, enforce timeout, exclude archive/reparse paths |
| R-05 | Alert storm from thresholds | Operational fatigue | Medium | AZM / OPS | Use suppression, auto-mitigation, staged thresholds |
| R-06 | Sensitive information in telemetry | Security/compliance impact | Low/High | SEC / SI | Data classification, source filtering, no file content/query text/secrets |
| R-07 | Teams integration approval delayed | Teams notification delayed | Medium | AZM / OPS | Use email as pilot baseline; implement Teams Logic App in parallel |
| R-08 | Script-signing process not ready | Production deployment blocked | Medium | SEC / SI | Establish signing process in Week 2–3 |
| R-09 | Workspace cost exceeds forecast | Budget risk | Low/Medium | AZM / SPO | Restrict collection scope and measure weekly |
| R-10 | No isolated test environment | Unsafe fault injection | Medium | PM / ERP | Obtain controlled test host/volume before test phase |

## 11.2 Assumptions

| ID | Assumption | Validation |
|---|---|---|
| A-01 | Outbound HTTPS/TCP 443 from pilot servers is permitted. | Network test. |
| A-02 | Pilot servers can be Azure Arc-enabled. | Arc onboarding test. |
| A-03 | System-assigned Arc identity is approved. | Security/IAM approval. |
| A-04 | ERP support can identify approved log paths and retention requirements. | Discovery workshop. |
| A-05 | Certificate monitoring store is `LocalMachine\WebHosting`. | Windows/IIS validation. |
| A-06 | Email notification distribution list is available. | Operations confirmation. |
| A-07 | Teams Logic App integration is approved. | Collaboration/platform approval. |
| A-08 | Azure Monitor and Log Analytics are approved cloud services. | Architecture/security approval. |

## 11.3 Dependencies

| ID | Dependency | Owner |
|---|---|---|
| D-01 | Azure subscription and resource group access | SPO / AZM |
| D-02 | Azure region decision | SA / SPO |
| D-03 | Azure Arc onboarding permissions | WIN / SEC |
| D-04 | Network allowlist/proxy configuration | NET |
| D-05 | Local collector account provisioning | WIN / SEC |
| D-06 | Code-signing certificate/process | SEC |
| D-07 | ERP log-path and retention definition | ERP |
| D-08 | Email mailbox and Teams channel | OPS |
| D-09 | Test certificate/test disk-volume approval | WIN / ERP |
| D-10 | Logic App/Teams connector approval | AZM / OPS |

---

# 12. Reporting Cadence

| Meeting/report | Frequency | Participants | Purpose |
|---|---:|---|---|
| Project status update | Weekly | PM, SA, SI, leads | Progress, risks, blockers, decisions |
| Technical design review | Weekly during build | SA, SI, WIN, AZM, SEC, NET | Validate architecture and implementation decisions |
| Security review | At design completion and before production | SEC, SA, SI, WIN, AZM | Review identities, RBAC, signing, data handling |
| Operations readiness review | Before fault testing and handover | OPS, SI, WIN, ERP, AZM | Validate runbooks, alerts, Workbooks |
| Cost review | After 7 days and 30 days of telemetry | AZM, SPO, SI | Validate forecast and retention |
| Go/no-go board | End of pilot | Sponsor and all leads | Approve rollout, extend pilot, or remediate |

---

# 13. Formal Success Measures

| Measure | Target |
|---|---|
| Certificate inventory freshness | Certificate hosts submit usable inventory at least every 30 hours |
| Certificate expiry detection | Test certificate within 7 days creates alert and notification |
| Disk-capacity detection | Test capacity threshold creates alert within expected collection/evaluation window |
| ERP folder growth visibility | Growth appears in KQL and Workbook after scheduled collection |
| Collector reliability | Successful collector-health record rate ≥ 99% during pilot observation period |
| Alert notification | Email delivery demonstrated for all alert categories |
| Teams notification | Teams delivery demonstrated for certificate and disk test alerts |
| Ingestion recovery | Spool replay demonstrated after restored endpoint/RBAC availability |
| Rollback | Previous signed collector package successfully restored |
| Security | No secrets/private keys/passwords found in telemetry or local logs |
| Cost | Measured cost accepted by budget owner |
| Operations | Tier 1/Tier 2 can execute runbooks successfully in tabletop or controlled test |

---

# 14. Project Closure Criteria

The project can close after:

1. Pilot acceptance criteria are met.
2. Fault-injection evidence is complete.
3. Operational runbooks are approved.
4. Production-readiness assessment is signed.
5. Cost and retention review is completed.
6. Production rollout recommendation is documented.
7. Future backlog is prioritized.
8. Sponsor approves one of:
   - Production rollout;
   - Extended pilot;
   - Remediation before rollout;
   - Solution redesign.

### Want me to…
<!-- copilot:suggestions -->
1. Produce a detailed 10-week implementation schedule with weekly activities, dependencies, and staffing estimates.
2. Add a formal risk-management plan, communication plan, and change-control procedure to this project package.
3. Hand this project-charter package off to the coding agent to open a pull request in a repository you provide.
