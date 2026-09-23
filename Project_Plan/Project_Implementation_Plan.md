# Detailed 10-Week Implementation Schedule  
## Hybrid Azure Monitor Pilot for On-Premises Windows ERP Servers

This schedule assumes:

- One ERP application server, one middleware/IIS server, and optional SQL-adjacent server in the pilot.
- Azure Arc onboarding is technically possible.
- Outbound HTTPS is approved.
- A dedicated pilot Azure resource group and Log Analytics Workspace will be used.
- Certificate monitoring, disk capacity, ERP log-folder monitoring, alerts, Workbooks, email, and Teams are in scope.
- ServiceNow, SQL DMV, REST API synthetic monitoring, and network port diagnostics remain future phases.

---

## 1. Staffing Estimate

## 1.1 Core Team

| Role | Typical effort | Primary contribution |
|---|---:|---|
| Project Manager | 0.1–0.2 FTE | Schedule, RAID log, status, approvals, stakeholder management |
| System Solution Architect | 0.1–0.2 FTE | Architecture decisions, security and production-readiness approval |
| System Integration Lead | 0.5–0.8 FTE during build | Bicep, collectors, DCR/DCE, KQL, alerts, Workbooks, testing |
| Azure Monitoring Engineer | 0.2–0.4 FTE | Log Analytics, Azure Monitor, alerts, Action Groups, Workbooks |
| Windows Infrastructure Engineer | 0.2–0.4 FTE | Azure Arc, collector account, Task Scheduler, NTFS, IIS access |
| ERP Application Support Lead | 0.1–0.25 FTE | ERP log-path discovery, retention policy, safe cleanup procedures |
| Security/IAM Engineer | 0.05–0.2 FTE | Arc identity, RBAC, local-account rights, code-signing review |
| Network Engineer | 0.05–0.15 FTE | Proxy, DNS, firewall, outbound endpoint validation |
| Operations Support Lead | 0.1–0.25 FTE | Runbooks, alert routing, operational acceptance, support training |
| Teams/Logic App Engineer | 0.05–0.15 FTE | Teams notification workflow, Adaptive Card, connector governance |

## 1.2 Estimated Total Pilot Effort

| Workstream | Estimated effort |
|---|---:|
| Architecture, requirements, and governance | 80–120 hours |
| Azure Bicep and monitoring foundation | 80–120 hours |
| Windows collector package and deployment | 100–160 hours |
| Certificate monitoring use case | 50–80 hours |
| Disk and ERP log-folder monitoring use case | 70–110 hours |
| Alerting, Teams/email, and Workbooks | 60–100 hours |
| Testing, fault injection, and remediation | 80–140 hours |
| Operations handover and production readiness | 50–90 hours |
| **Total estimated pilot effort** | **570–920 hours** |

This is a realistic multi-team pilot. The range depends heavily on existing Azure Arc readiness, network approvals, code-signing maturity, and Teams/Logic App approval.

---

# 2. Milestone Summary

| Milestone | Target week | Approval or evidence |
|---|---:|---|
| M0 — Charter and scope approved | Week 1 | Project sponsor and technical leads approve scope |
| M1 — Architecture and readiness approved | Week 2 | Region, network, identity, security, and pilot servers confirmed |
| M2 — Azure foundation deployed | Week 3 | Workspace, DCE, DCRs, tables, and RBAC validated |
| M3 — Collector platform deployed | Week 4 | Package, Task Scheduler, Arc HIMDS token access validated |
| M4 — Certificate monitoring operational | Week 5 | Certificate data, alerts, email, and dashboard validated |
| M5 — Disk and ERP folder monitoring operational | Week 6 | Disk/folder data, alerts, and dashboard validated |
| M6 — Teams and operational alerting complete | Week 7 | Teams Logic App and resolution workflow validated |
| M7 — Fault injection and recovery complete | Week 8 | Test register completed; remediation gaps identified |
| M8 — Handover, cost, and performance review complete | Week 9 | Runbooks, training, cost baseline, support acceptance complete |
| M9 — Production-readiness decision | Week 10 | Go/no-go decision and production rollout plan approved |

---

# 3. Week-by-Week Implementation Plan

## Week 1 — Project Initiation, Scope, and Discovery

### Objectives

- Approve pilot charter.
- Confirm pilot use cases and pilot server population.
- Begin technical discovery.
- Identify owners and approval dependencies.

### Activities

| Activity | Owner | Support | Deliverable |
|---|---|---|---|
| Conduct project kickoff | PM | SA, SI, WIN, ERP, AZM, SEC, NET, OPS | Kickoff record |
| Approve project charter | Sponsor | PM, SA, SI | Approved charter |
| Confirm in-scope servers | WIN / ERP | SI, AZM | Pilot server inventory |
| Confirm certificate scope | WIN / ERP | SI | `WebHosting` store and IIS scope confirmed |
| Identify ERP log directories | ERP | WIN, SI | Log-path inventory |
| Confirm disk volumes | WIN / ERP | SI | Drive/volume inventory |
| Confirm support email distribution list | OPS | PM | Email recipient list |
| Confirm Teams channel owner | OPS | Teams/Logic App engineer | Teams channel decision |
| Create RAID log | PM | All leads | Initial RAID register |
| Identify Azure subscription/resource-group owner | Sponsor / AZM | PM | Azure ownership record |

### Technical Discovery Checklist

- Hostname and Azure Arc status.
- Windows Server version.
- IIS installation status.
- Current certificate count in `Cert:\LocalMachine\WebHosting`.
- ERP log-directory path.
- Log-folder size and approximate file count.
- Disk volumes that contain OS, ERP application, and ERP logs.
- Existing proxy settings.
- Existing Azure subscription and landing-zone requirements.
- Current local service-account management policy.
- Existing PowerShell execution policy.
- Code-signing certificate/process availability.

### Dependencies

- Project sponsor participation.
- ERP application owner availability.
- Windows server inventory availability.
- Azure subscription confirmation.

### Week 1 Exit Criteria

- Charter approved.
- Pilot servers selected.
- ERP log directories identified.
- Certificate-monitoring scope confirmed.
- Open decision log includes Azure region, workspace ownership, and local collector account model.

### Estimated Effort

| Role | Estimated hours |
|---|---:|
| PM | 12 |
| SA | 12 |
| SI | 20 |
| WIN | 12 |
| ERP | 12 |
| AZM | 8 |
| SEC | 4 |
| NET | 4 |
| OPS | 4 |
| **Total** | **88 hours** |

---

## Week 2 — Architecture, Security, Network, and Readiness Approval

### Objectives

- Finalize core technical design.
- Confirm Azure region, workspace strategy, identity design, and connectivity.
- Resolve blockers before build work starts.

### Activities

| Activity | Owner | Support | Deliverable |
|---|---|---|---|
| Approve Azure region | SA / Sponsor | AZM, SEC | Architecture decision record |
| Confirm pilot workspace strategy | SA / AZM | SEC, Sponsor | Workspace decision |
| Finalize architecture diagram | SI | SA, AZM, WIN | Architecture diagram |
| Define DCE/DCR/table model | SI | AZM | Data model and DCR design |
| Define Azure Arc identity model | WIN / SEC | SI, AZM | Identity design |
| Define local task account model | WIN / SEC | SI | Local account and rights design |
| Confirm Azure RBAC matrix | SEC | AZM, SI | RBAC matrix |
| Confirm public endpoint/proxy path | NET | WIN, AZM | Network connectivity matrix |
| Define data classification | SEC | ERP, SI | Data collection/exclusion register |
| Define naming/tagging standards | AZM | SI, SA | Naming/tagging standard |
| Define code-signing process | SEC | SI, WIN | Script-signing process |
| Define alert severity model | OPS / AZM | SI, ERP | Alert design register |

### Key Decisions Required

1. Azure region.
2. Resource-group and workspace names.
3. Whether AMA is approved for disk performance counters.
4. Whether custom disk collector is required.
5. Dedicated local collector account versus approved alternative.
6. Email distribution list and Teams channel.
7. ERP retention policy for monitored log directories.
8. Code-signing authority and signing workflow.

### Dependencies

- Security/IAM approval.
- Network/proxy confirmation.
- Azure landing-zone standards.
- Operations support ownership confirmation.

### Week 2 Exit Criteria

- Architecture baseline approved.
- Security controls agreed.
- Network path validated conceptually.
- Azure region selected.
- Identity and RBAC model approved.
- Bicep and collector development can begin.

### Estimated Effort

| Role | Estimated hours |
|---|---:|
| PM | 8 |
| SA | 16 |
| SI | 28 |
| WIN | 16 |
| AZM | 16 |
| SEC | 16 |
| NET | 12 |
| ERP | 8 |
| OPS | 8 |
| **Total** | **128 hours** |

---

## Week 3 — Azure Foundation and Infrastructure as Code

### Objectives

- Build and deploy Azure Monitor infrastructure.
- Create custom tables, DCE, DCRs, RBAC, and email Action Group.
- Validate the Azure deployment before deploying production-like collectors.

### Activities

| Activity | Owner | Support | Deliverable |
|---|---|---|---|
| Create Bicep repository structure | SI | AZM | IaC repository |
| Build Bicep modules | SI | AZM | Bicep source |
| Define Log Analytics Workspace | AZM | SI | Workspace module |
| Define custom tables | SI | AZM | Table schemas |
| Define DCE | SI | AZM, NET | DCE module |
| Define DCR streams and data flows | SI | AZM | DCR modules |
| Define DCR RBAC assignments | SI | SEC, AZM | RBAC module |
| Define email Action Group | AZM | OPS | Action Group module |
| Build parameter files | SI | AZM | Pilot parameters |
| Run Bicep build/validate/what-if | SI | AZM | Validation evidence |
| Deploy pilot Azure resources | AZM / SI | SEC | Deployment outputs |
| Validate Arc principal IDs | WIN | SI, AZM | Principal ID register |
| Test DCR role assignments | SI | WIN | Authorization validation |

### Dependencies

- Azure region/resource group approved.
- Arc machine system-assigned managed identities enabled.
- Azure deployment permissions available.
- Principal IDs retrievable.

### Week 3 Exit Criteria

- Dedicated pilot workspace deployed.
- DCE deployed.
- Custom tables exist:
  - `ServerCertificateInventory_CL`
  - `ErpFolderMetrics_CL`
  - `WindowsDiskMetrics_CL`, if required
  - `CollectorHealth_CL`
- DCRs deployed.
- Arc identities have DCR-scoped ingestion roles.
- Email Action Group deployed.
- Bicep deployment artifacts and outputs saved.

### Estimated Effort

| Role | Estimated hours |
|---|---:|
| SI | 36 |
| AZM | 28 |
| WIN | 8 |
| SEC | 8 |
| NET | 4 |
| PM | 4 |
| **Total** | **88 hours** |

---

## Week 4 — Windows Collector Platform Deployment

### Objectives

- Deploy the signed PowerShell collector framework.
- Configure local accounts, permissions, Scheduled Tasks, and Azure Arc HIMDS token access.
- Validate end-to-end direct ingestion.

### Activities

| Activity | Owner | Support | Deliverable |
|---|---|---|---|
| Complete shared collector module | SI | WIN, AZM | Shared module |
| Build release package | SI | SEC | Collector package |
| Sign scripts and modules | SEC / SI | WIN | Signed package |
| Create collector service account | WIN | SEC | Local account |
| Apply task-account rights | WIN | SEC | Rights assignment evidence |
| Add HIMDS local access | WIN | SEC | Group membership evidence |
| Create collector folders/ACLs | WIN | SI | Local deployment structure |
| Install collector package | WIN / SI | SEC | Installed package |
| Apply configuration files | SI | ERP, AZM | Server configuration |
| Register Scheduled Tasks | WIN | SI | Scheduled tasks |
| Run prerequisite tests as task account | WIN | SI | Validation report |
| Test Arc HIMDS token acquisition | SI | WIN, SEC | Token test evidence |
| Run direct-ingestion test | SI | AZM | Validation record in LAW |
| Validate spool/retry baseline | SI | WIN | Initial spool test |

### Dependencies

- Week 3 deployment outputs.
- Task account approved.
- Script-signing process available.
- Required server access approved.
- ERP log path confirmed.

### Week 4 Exit Criteria

- Collector framework deployed to pilot hosts.
- Task account can obtain Arc managed-identity token.
- Scheduled tasks exist and run under approved account.
- Certificate, folder, and disk collector configs contain valid DCE/DCR values.
- Direct-ingestion validation record appears in Log Analytics.
- No secrets are present in scripts/configuration/local logs.

### Estimated Effort

| Role | Estimated hours |
|---|---:|
| SI | 40 |
| WIN | 32 |
| SEC | 12 |
| AZM | 12 |
| ERP | 8 |
| PM | 4 |
| **Total** | **108 hours** |

---

## Week 5 — Certificate Monitoring Implementation

### Objectives

- Make certificate inventory operational.
- Build certificate KQL, dashboard, alerts, and email notification.
- Validate IIS binding correlation where applicable.

### Activities

| Activity | Owner | Support | Deliverable |
|---|---|---|---|
| Validate WebHosting certificate enumeration | WIN / SI | ERP | Certificate collection evidence |
| Validate IIS binding discovery | WIN | SI, ERP | IIS correlation evidence |
| Run scheduled certificate collector | WIN / SI | AZM | Certificate records |
| Validate `ServerCertificateInventory_CL` schema | AZM | SI | Schema validation |
| Build certificate KQL catalogue | SI | AZM | KQL files |
| Create certificate Workbook | SI | AZM, OPS | Workbook |
| Create certificate alert Bicep module | SI | AZM | Alert IaC |
| Deploy expired/7/14/30-day alerts | AZM / SI | OPS | Alert rules |
| Configure email notification | AZM | OPS | Email validation |
| Validate collector-health data | SI | AZM | Health query result |
| Draft certificate response runbook | OPS | WIN, ERP, SI | Runbook |
| Conduct certificate alert test | SI | WIN, OPS | Test evidence |

### Dependencies

- Certificate collector deployed.
- Certificate DCR and table operational.
- Test certificate procedure approved.
- Email distribution list operational.

### Week 5 Exit Criteria

- Certificate inventory visible in Log Analytics.
- Certificate expiry dashboard operational.
- Certificate alerts deployed and enabled.
- Email alert tested.
- Certificate collector failure/stale-data checks operating.
- Certificate response runbook reviewed.

### Estimated Effort

| Role | Estimated hours |
|---|---:|
| SI | 32 |
| AZM | 24 |
| WIN | 16 |
| ERP | 8 |
| OPS | 12 |
| SEC | 4 |
| PM | 4 |
| **Total** | **100 hours** |

---

## Week 6 — Disk and ERP Log-Folder Monitoring Implementation

### Objectives

- Deploy and validate disk capacity/performance monitoring.
- Deploy ERP folder-size, growth, and retention monitoring.
- Build KQL, alerts, and Workbook.

### Activities

| Activity | Owner | Support | Deliverable |
|---|---|---|---|
| Confirm AMA versus custom disk collector path | SA / SI | WIN, AZM | Design decision |
| Deploy AMA disk DCR, if approved | AZM / WIN | SI | AMA DCR association |
| Deploy custom disk collector, if required | SI / WIN | AZM | Disk collector |
| Validate capacity and latency records | SI | AZM, WIN | Disk telemetry evidence |
| Configure ERP folder collector | SI | ERP, WIN | Folder configuration |
| Validate folder sizing and scan duration | SI | ERP, WIN | Folder telemetry evidence |
| Build disk/folder KQL catalogue | SI | AZM | KQL files |
| Build disk/folder Workbook | SI | AZM, OPS | Workbook |
| Deploy capacity, latency, growth, retention alerts | AZM / SI | OPS, ERP | Alert rules |
| Draft capacity/log response runbook | OPS | ERP, WIN, SI | Runbook |
| Test warning/critical thresholds in test environment | SI | WIN, ERP, OPS | Test evidence |

### Dependencies

- ERP log folder and retention policy confirmed.
- Test volume or safe test-data path available.
- Disk collection approach approved.
- Folder collector deployed.

### Week 6 Exit Criteria

- Disk capacity data is visible.
- ERP folder size and growth are visible.
- Disk/folder Workbook operational.
- Capacity and folder-growth alerts deployed.
- Retention accumulation is visible.
- Disk/folder operational runbook drafted and reviewed.

### Estimated Effort

| Role | Estimated hours |
|---|---:|
| SI | 40 |
| AZM | 20 |
| WIN | 24 |
| ERP | 16 |
| OPS | 12 |
| SA | 4 |
| PM | 4 |
| **Total** | **120 hours** |

---

## Week 7 — Teams Integration, Alert Tuning, and Operational Workflow

### Objectives

- Implement Teams notifications through Logic App.
- Validate Common Alert Schema.
- Tune alert severity, suppression, and auto-resolution.
- Prepare operations for incident workflow.

### Activities

| Activity | Owner | Support | Deliverable |
|---|---|---|---|
| Design Teams Logic App workflow | AZM / Teams engineer | OPS, SI | Logic App design |
| Create HTTP trigger and Common Alert Schema handling | Teams engineer | AZM | Workflow |
| Create Adaptive Card format | Teams engineer | OPS, SI | Adaptive Card |
| Connect Logic App to Action Group | AZM | Teams engineer | Action Group integration |
| Test certificate alert to Teams | SI | OPS | Teams evidence |
| Test disk alert to Teams | SI | OPS | Teams evidence |
| Configure alert mute durations | AZM | OPS, ERP | Alert tuning |
| Validate auto-mitigation/resolution | SI | AZM, OPS | Resolution evidence |
| Define alert ownership/escalation | OPS | ERP, WIN, AZM | Escalation matrix |
| Conduct Tier 1 dashboard/alert walkthrough | OPS | SI, AZM | Operations feedback |

### Dependencies

- Teams channel and connector approval.
- Logic App resource access.
- Certificate and disk alerts deployed.
- Operations ownership established.

### Week 7 Exit Criteria

- Teams receives approved alert cards.
- Email and Teams activation/resolution notifications tested.
- Alert suppression prevents notification storms.
- Escalation ownership is documented.
- Operations team can identify the correct Workbook and runbook.

### Estimated Effort

| Role | Estimated hours |
|---|---:|
| SI | 20 |
| AZM | 24 |
| Teams/Logic App engineer | 20 |
| OPS | 16 |
| WIN | 4 |
| ERP | 4 |
| PM | 4 |
| **Total** | **92 hours** |

---

## Week 8 — Fault Injection, Recovery, and Resilience Testing

### Objectives

- Validate expected failure detection.
- Confirm spool/retry, identity, RBAC, alert, and rollback behavior.
- Document and remediate gaps.

### Activities

| Activity | Owner | Support | Deliverable |
|---|---|---|---|
| Approve fault-injection schedule | PM | SA, WIN, ERP, OPS | Test approval |
| Execute certificate expiry test | SI | WIN, OPS | Test evidence |
| Execute certificate private-key test | SI | WIN, ERP | Test evidence |
| Execute disk warning/critical test | SI | WIN, ERP, OPS | Test evidence |
| Execute ERP folder growth test | SI | ERP, WIN | Test evidence |
| Execute retention-age test | SI | ERP | Test evidence |
| Execute access-denied/timeout test | SI | WIN | Test evidence |
| Execute DCR RBAC removal/restoration test | SI | SEC, AZM | Test evidence |
| Execute endpoint interruption/spool replay test | SI | NET, WIN | Test evidence |
| Execute HIMDS access failure test | WIN / SI | SEC | Test evidence |
| Execute alert suppression/resolution test | AZM | OPS | Test evidence |
| Execute collector rollback test | SI / WIN | OPS | Rollback evidence |
| Record defects and remediation plan | PM / SI | All leads | Updated RAID log |

### Dependencies

- Controlled test environment.
- Approved test certificates/test data.
- Change authorization.
- Operations and network support availability.

### Week 8 Exit Criteria

- Mandatory fault tests completed.
- Spool replay demonstrated.
- Collector rollback demonstrated.
- Alert activation and auto-resolution demonstrated.
- High-severity defects either fixed or formally accepted.
- Test report drafted.

### Estimated Effort

| Role | Estimated hours |
|---|---:|
| SI | 44 |
| WIN | 24 |
| AZM | 20 |
| ERP | 16 |
| SEC | 12 |
| NET | 8 |
| OPS | 16 |
| PM | 8 |
| SA | 4 |
| **Total** | **152 hours** |

---

## Week 9 — Operations Handover, Cost Review, and Production Readiness Assessment

### Objectives

- Complete documentation and support handover.
- Measure ingestion, operational impact, and cost.
- Review security and support readiness.

### Activities

| Activity | Owner | Support | Deliverable |
|---|---|---|---|
| Finalize all runbooks | OPS | SI, WIN, ERP, AZM | Approved runbooks |
| Conduct Tier 1/Tier 2 training | OPS | SI, WIN, AZM | Training evidence |
| Review collector CPU/memory/runtime | WIN / SI | ERP | Performance report |
| Review folder scan duration and access errors | ERP / SI | WIN | Folder performance report |
| Review workspace ingestion volume | AZM | SI | Ingestion report |
| Calculate monthly cost forecast | AZM | Sponsor, PM | Cost forecast |
| Review alert quality/false positives | OPS | ERP, WIN, AZM | Alert-tuning register |
| Review Arc/DCR/RBAC security posture | SEC | WIN, AZM, SI | Security acceptance |
| Update production-readiness assessment | SI | SA, PM, leads | Draft readiness report |
| Create production rollout backlog | SI | SA, WIN, ERP, AZM | Rollout backlog |

### Dependencies

- At least one week of stable telemetry after implementation.
- Fault testing completed.
- Operations staff available for training.
- Cost data accessible.

### Week 9 Exit Criteria

- Operations handover complete.
- Runbooks approved.
- Cost forecast available.
- Security review complete.
- Production-readiness report drafted.
- Open risks assigned owners and due dates.

### Estimated Effort

| Role | Estimated hours |
|---|---:|
| SI | 28 |
| OPS | 28 |
| AZM | 20 |
| WIN | 16 |
| ERP | 12 |
| SEC | 12 |
| SA | 8 |
| PM | 8 |
| Sponsor | 4 |
| **Total** | **136 hours** |

---

## Week 10 — Production Readiness Decision and Pilot Closure

### Objectives

- Review pilot outcomes against objectives.
- Obtain formal go/no-go decision.
- Define rollout waves and future backlog.

### Activities

| Activity | Owner | Support | Deliverable |
|---|---|---|---|
| Finalize production-readiness assessment | SI | SA, PM, all leads | Final readiness report |
| Conduct pilot outcome review | PM / SA | All leads | Pilot review minutes |
| Review unresolved risks | SA | SEC, WIN, ERP, AZM | Risk acceptance decisions |
| Approve production rollout waves | Sponsor / SA | PM, leads | Rollout plan |
| Approve Wave 3 production candidates | WIN / ERP | SA, OPS | Production candidate list |
| Confirm production change process | PM | WIN, AZM, OPS | Change plan |
| Define post-pilot support ownership | OPS | Sponsor, SA | Operating model |
| Prioritize phase-two backlog | SA / SI | DB, ERP, AZM | Enhancement backlog |
| Close or extend pilot | Sponsor | PM, SA | Formal decision |

### Recommended Phase-Two Backlog

1. SQL Server DMV, blocking, I/O, backup, and job monitoring.
2. REST API synthetic monitoring.
3. TCP connection and port diagnostics.
4. Service-account and gMSA lifecycle monitoring.
5. Entra-backed Windows LAPS metadata tracking.
6. ServiceNow incident integration.
7. Vendor ERP application-log patterns.
8. Cross-tier ERP/middleware/SQL operational Workbook.

### Week 10 Exit Criteria

- Sponsor issues go/no-go decision.
- Production rollout plan exists.
- Support ownership accepted.
- Open risks have approved disposition.
- Pilot closure report is completed.
- Phase-two backlog is prioritized.

### Estimated Effort

| Role | Estimated hours |
|---|---:|
| PM | 12 |
| SA | 12 |
| SI | 20 |
| WIN | 8 |
| ERP | 8 |
| AZM | 8 |
| SEC | 4 |
| OPS | 8 |
| Sponsor | 8 |
| **Total** | **88 hours** |

---

# 4. Consolidated Weekly Staffing Plan

| Role | W1 | W2 | W3 | W4 | W5 | W6 | W7 | W8 | W9 | W10 | Total |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| Project Manager | 12 | 8 | 4 | 4 | 4 | 4 | 4 | 8 | 8 | 12 | 68 |
| Solution Architect | 12 | 16 | 0 | 0 | 0 | 4 | 0 | 4 | 8 | 12 | 56 |
| System Integrator | 20 | 28 | 36 | 40 | 32 | 40 | 20 | 44 | 28 | 20 | 308 |
| Azure Monitoring | 8 | 16 | 28 | 12 | 24 | 20 | 24 | 20 | 20 | 8 | 180 |
| Windows Infrastructure | 12 | 16 | 8 | 32 | 16 | 24 | 4 | 24 | 16 | 8 | 160 |
| ERP Support | 12 | 8 | 0 | 8 | 8 | 16 | 4 | 16 | 12 | 8 | 92 |
| Security/IAM | 4 | 16 | 8 | 12 | 4 | 0 | 0 | 12 | 12 | 4 | 72 |
| Network | 4 | 12 | 4 | 0 | 0 | 0 | 0 | 8 | 0 | 0 | 28 |
| Operations Support | 4 | 8 | 0 | 0 | 12 | 12 | 16 | 16 | 28 | 8 | 104 |
| Teams/Logic App Engineer | 0 | 0 | 0 | 0 | 0 | 0 | 20 | 0 | 0 | 0 | 20 |
| Sponsor/Budget Owner | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 4 | 8 | 12 |
| **Total effort** | **88** | **128** | **88** | **108** | **100** | **120** | **92** | **152** | **136** | **88** | **1,100** |

The 1,100-hour total is the high end because it includes structured testing, governance, training, fault injection, and cross-functional involvement. A streamlined pilot may use 570–920 hours if existing Azure Arc, signing, networking, and Teams foundations are already available.

---

# 5. Dependency Map

```text
Week 1 discovery
    ↓
Week 2 architecture/security/network approvals
    ↓
Week 3 Azure infrastructure deployment
    ↓
Week 4 local collector package and managed identity validation
    ├── Week 5 certificate monitoring
    └── Week 6 disk/folder monitoring
              ↓
Week 7 email/Teams workflow and alert tuning
              ↓
Week 8 fault injection and recovery tests
              ↓
Week 9 operations handover, security review, cost review
              ↓
Week 10 go/no-go decision and rollout plan
```

---

# 6. Schedule Compression Options

If the project must complete faster than 10 weeks, these workstreams can run in parallel after Week 2 approval.

| Parallel workstreams | Constraint |
|---|---|
| Week 3 Bicep build + Week 3 collector code finalization | Requires agreed schemas and DCR stream names. |
| Certificate monitoring + disk/folder monitoring | Requires the shared collector framework to be stable. |
| Teams Logic App design + Workbook design | Requires alert payload and ownership model, but not final production data. |
| Runbook drafting + technical build | Requires operational owners to participate early. |
| Cost model + collector deployment | Initial estimate can be prepared before actual ingestion data. |

Do not compress:

- Security/identity approval.
- Network/proxy validation.
- DCR RBAC validation.
- Fault injection.
- Rollback testing.
- Operational acceptance.

---

# 7. Schedule Risks and Contingency

| Risk | Potential delay | Contingency |
|---|---|---|
| Azure region undecided | 1–2 weeks | Escalate as Week 1 decision gate; prepare Bicep region parameterization in parallel. |
| Arc managed identity issue | 1 week | Use dedicated troubleshooting workstream with Windows and Security teams before collector deployment. |
| Proxy/firewall approval delay | 1–3 weeks | Test in an already-approved network segment; prepare endpoint evidence for network team. |
| Script-signing process unavailable | 1–2 weeks | Use controlled non-production exception only for pilot proof, then sign before production readiness gate. |
| No safe test volume/certificate | 1 week | Build an isolated test VM/volume; do not inject faults into production. |
| Teams connector approval delay | 1–2 weeks | Use email as baseline notification; complete Teams workflow as conditional pilot deliverable. |
| ERP log path/retention undefined | 1–2 weeks | Monitor capacity only initially; delay growth/retention alerting until ERP owner approves policy. |
| High folder scan runtime | 1 week | Reduce scan frequency, constrain paths, exclude archive directories, or use storage metrics. |

### Want me to…
<!-- copilot:suggestions -->
1. Add a detailed Gantt-style dependency schedule with named work packages, predecessors, and completion criteria.
2. Produce a formal communication plan, change-control procedure, and risk-management plan for this pilot.
3. Hand this project schedule and planning package off to the coding agent to open a pull request in a repository you provide.
