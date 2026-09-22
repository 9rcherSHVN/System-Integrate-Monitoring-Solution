# Updated Pilot Scope Plan — Hybrid End-to-End Monitoring for On-Prem Windows ERP Platform

The pilot should validate a **cloud-managed, hybrid observability platform** for on-premises Windows ERP, middleware, and SQL Server workloads. The recommended foundation remains:

> **Azure Arc-enabled Servers + Azure Monitor Agent (AMA) + Data Collection Rules (DCR) + Log Analytics Workspace + Azure Monitor alerts + Azure Workbooks + ServiceNow/Teams/Outlook integration.**

The pilot must prove both **continuous proactive monitoring** and **on-demand incident diagnostics**. It should not be treated as only an infrastructure dashboard project.

---

## 1. Pilot Objectives and Success Criteria

### 1.1 Objectives

Validate that IT support can centrally monitor and troubleshoot:

1. Windows runtime and server health.
2. Disk capacity, disk performance, and application/SQL disk growth.
3. TCP connectivity and port availability between ERP, middleware, and SQL tiers.
4. REST API availability and response performance.
5. SQL Server resource use, sessions, blocking, waits, I/O, and database-file growth.
6. Certificate expiry and service-account/password-expiry exposure.
7. Alert-to-ticket workflow through ServiceNow, Teams, and Outlook.
8. Historical trends and cross-tier diagnosis through dashboards and KQL queries.

### 1.2 Pilot Success Criteria

The pilot is successful if it demonstrates all of the following:

| Area | Success criterion |
|---|---|
| Hybrid onboarding | Three representative on-prem Windows servers are successfully onboarded into Azure Arc without inbound internet exposure. |
| Infrastructure telemetry | CPU, memory, disk capacity, disk latency, Windows services, and selected Windows event logs are visible in Azure Monitor. |
| Network dependency checks | Defined ERP → middleware and application/middleware → SQL TCP checks detect failure and latency degradation. |
| REST health | A representative ERP REST endpoint is checked for HTTP status, response time, and functional response validation. |
| SQL visibility | SQL session origin, blocking, waits, I/O latency, database-file size, and growth are collected and queryable. |
| Security operations | Certificate expiry and service-account expiry/compliance data are visible and alertable. Actual passwords are never collected. |
| Operations workflow | A critical alert creates or updates a ServiceNow incident and notifies the designated Teams channel/email group. |
| Troubleshooting | Support personnel can use a workbook and saved KQL queries to identify whether an incident originates in app, network, SQL, disk, certificate, or credentials. |
| Governance/cost | Data ingestion volume, retention, Azure role model, outbound firewall requirements, and operational ownership are documented. |

---

# 2. Pilot Reference Architecture

```text
                         Azure Tenant / Management Plane
┌───────────────────────────────────────────────────────────────────────┐
│ Azure Arc                                                            │
│ Azure Monitor Agent (AMA) + Data Collection Rules                    │
│ Data Collection Endpoint + Logs Ingestion API                        │
│ Log Analytics Workspace                                              │
│ Azure Monitor Alerts + Action Groups                                 │
│ Azure Workbooks / optional Managed Grafana                           │
│ Logic Apps / ITSM connector → ServiceNow / Teams / Outlook           │
└───────────────────────────────────────────────────────────────────────┘
                                  ▲
                                  │ Outbound HTTPS only, typically TCP 443
                                  │
┌─────────────────────────────────┴─────────────────────────────────────┐
│                         On-Premises Environment                       │
│                                                                       │
│ ERP application server ── TCP/HTTPS ──> Middleware/Web server         │
│       │                                               │               │
│       └──────────────── TCP/SQL ──────────────────────┼───────> SQL  │
│                                                       │      Server   │
│                                                                       │
│ Azure Arc agent + AMA on all pilot servers                           │
│ SQL/PowerShell collectors only where custom data is required         │
│ Automation Hybrid Worker or controlled scheduled-task runner         │
└───────────────────────────────────────────────────────────────────────┘
```

---

# 3. Recommended Pilot Population

Use a small but representative dependency chain rather than three arbitrary servers.

| Pilot node | Role | Minimum purpose |
|---|---|---|
| `ERP-APP-01` | ERP business application / REST API server | Hosts a representative ERP API and application service. |
| `ERP-MW-01` | Middleware, web, proxy, integration, or message-processing server | Represents the middle-tier dependency and API routing path. |
| `ERP-SQL-01` | SQL Server | Hosts a non-production but representative ERP database. |
| Optional `MON-TOOLS-01` | Management utility server | Hosts Azure Automation Hybrid Worker or controlled custom collectors if security policy does not permit them directly on production-like servers. |

Use **non-production or production-like pre-production workloads** first. The pilot should include realistic traffic, service accounts, certificates, SQL workload patterns, and disk layouts.

---

# 4. Workstream 1 — Azure Foundation, Connectivity, and Governance

## 4.1 Activities

1. Create or designate:
   - Azure subscription/resource group.
   - Log Analytics Workspace.
   - Data Collection Endpoint, if custom ingestion is used.
   - Azure Monitor Action Groups.
   - Azure Key Vault for collector credentials, API secrets, and certificate-management secrets where applicable.
2. Onboard pilot Windows servers to Azure Arc.
3. Deploy Azure Monitor Agent through Azure Arc.
4. Define Azure RBAC model:
   - Monitoring administrators.
   - Support/read-only analysts.
   - Automation operators.
   - ServiceNow integration account owners.
5. Confirm outbound proxy, DNS, TLS inspection, firewall, and private-network routing requirements.
6. Configure tagging standards:
   - `Environment`
   - `Application`
   - `ApplicationOwner`
   - `ServerRole`
   - `Criticality`
   - `SupportTeam`
   - `DataClassification`

## 4.2 Architecture Decisions to Confirm

- One Log Analytics Workspace for the pilot, preferably separated by tags and resource groups.
- Production workspace topology decision deferred until after pilot cost and access-control assessment.
- Data retention period, such as 30–90 days for high-volume operational logs and longer retention only where justified.
- Whether Azure services are accessed through public endpoints with outbound-only firewall rules or through private-link-enabled architecture where available and required by policy.

---

# 5. Workstream 2 — Windows Server Runtime, Service, Event, and Disk Monitoring

This is the core infrastructure baseline and should be implemented through **AMA and DCR-based performance counters/event-log collection**.

## 5.1 Required Runtime Signals

| Domain | Signals |
|---|---|
| CPU | Processor utilization, processor queue length, sustained high CPU. |
| Memory | Available memory, committed bytes, paging indicators, memory pressure. |
| Operating system | Restart events, unexpected shutdown, critical Windows events, time synchronization health. |
| Windows services | ERP services, middleware services, SQL Server services, SQL Agent, IIS/WAS where applicable. |
| IIS/application pools | Application pool stopped/recycled state, worker-process failures, HTTP errors, IIS log exceptions. |
| Event logs | System, Application, selected Security events, ERP/middleware vendor event channels where available. |
| Network interface | Bytes sent/received, errors/discards, connection resets where meaningful. |
| Scheduled tasks/jobs | Critical ERP batch jobs, backup jobs, and integration jobs: success/failure state where technically available. |

## 5.2 Disk Capacity and Disk Performance

### Standard Counters

Collect at five-minute intervals initially:

- `LogicalDisk(*)\% Free Space`
- `LogicalDisk(*)\Free Megabytes`
- `LogicalDisk(*)\Avg. Disk sec/Read`
- `LogicalDisk(*)\Avg. Disk sec/Write`
- `LogicalDisk(*)\Disk Transfers/sec`
- `LogicalDisk(*)\Current Disk Queue Length`
- `LogicalDisk(*)\% Disk Time` where useful

### Role-Aware Alert Thresholds

| Condition | Suggested initial action |
|---|---|
| Free capacity under 20% | Warning; Teams notification and capacity-planning ticket. |
| Free capacity under 10% | Critical; ServiceNow incident, Teams, and email escalation. |
| SQL log/data/tempdb volume rapidly declining | Critical or high-priority alert based on growth rate and predicted exhaustion. |
| Sustained disk latency above agreed baseline | Performance alert; correlate against SQL waits and I/O statistics before escalation. |

Thresholds must be tuned per drive size and workload. A fixed “10% free” rule is insufficient for a 50 GB OS disk versus a 10 TB SQL data volume. Add **absolute-free-space thresholds** as well, for example, warning below 50 GB and critical below 20 GB on designated SQL volumes.

## 5.3 Additional Disk Scope

Add these targeted checks:

- ERP and middleware log-directory size and growth rate.
- IIS log-directory growth.
- SQL backup directory capacity and cleanup status.
- SQL transaction-log volume capacity.
- `tempdb` volume capacity.
- Windows event-log size/retention where applications generate high event volume.

---

# 6. Workstream 3 — Network, TCP Port, REST API, and Runtime Connection Monitoring

This workstream needs multiple methods because no single product gives full TCP, HTTP, process-level, and live-connection visibility.

## 6.1 Continuous Dependency Monitoring

Define an application dependency inventory before configuring tests.

| Source | Destination | Protocol/Port | Purpose | Business criticality |
|---|---|---:|---|---|
| ERP app | Middleware | HTTPS/443 or vendor port | ERP API/integration calls | High |
| ERP app | SQL | TCP/1433 or named-instance port | Database connectivity | Critical |
| Middleware | SQL | TCP/1433 or named-instance port | Message/integration persistence | Critical |
| Client/internal monitor | ERP API | HTTPS/443 | API availability | High |

For each dependency, document:
- DNS name and IP.
- Protocol and fixed port.
- Firewall ownership.
- Expected TLS certificate.
- Authentication method.
- Normal latency baseline.
- Responsible support team.

Use Azure-native connection monitoring where it supports the target topology and required source/destination arrangement. Where it does not meet internal-path requirements, use a controlled synthetic probe from a monitored on-prem server.

## 6.2 REST API Monitoring

Monitor at the **application layer**, not merely the TCP layer.

A REST health check should validate:

1. DNS resolution.
2. TCP connection.
3. TLS handshake and certificate validity.
4. HTTP response status code.
5. Response time.
6. Optional response-body validation, such as expected JSON health status.
7. Authentication behavior using a non-privileged monitoring identity, if a health endpoint cannot be anonymous.

### Important Design Constraint

Public cloud availability tests do not prove that an internal ERP client can access a private on-prem endpoint. For internal-only APIs, use a controlled **on-premises synthetic probe**—for example, a secured scheduled PowerShell/.NET collector or automation worker—to execute the request from the same network zone as the real caller and publish the result to Log Analytics.

Do not use a production business transaction that writes, posts, or changes ERP data. Ask the ERP vendor/application owner to provide a safe `health`, `status`, `version`, or read-only validation endpoint.

## 6.3 On-Demand Connection Diagnostics

For incident support, collect point-in-time TCP connection state using a controlled PowerShell diagnostic action:

- `Get-NetTCPConnection`
- Process/PID enrichment
- Service/process name
- Local and remote address/port
- TCP state: `Established`, `Listen`, `TimeWait`, `CloseWait`, etc.
- Optional process command line, subject to security review

This capability should run:
- On demand through an Azure Automation Hybrid Worker or approved orchestration method.
- With least privilege.
- With audit logging.
- Without collecting sensitive payload data.

The collector should publish to a custom Log Analytics table through the **Logs Ingestion API**. A DCR maps the incoming custom stream to that table; the DCR itself does **not** execute PowerShell scripts.

### Diagnostic Use Case

During an ERP timeout, support should be able to answer:

- Is the ERP process listening on its expected API port?
- Does the ERP process have established SQL connections?
- Is the middleware server connected to SQL?
- Are there excessive `TIME_WAIT` or `CLOSE_WAIT` connections?
- Has the client connected to the wrong destination IP due to DNS or load-balancer behavior?
- Is the affected process different from the expected ERP service process?

---

# 7. Workstream 4 — SQL Server Monitoring and DMV Collection

## 7.1 Baseline SQL Monitoring

Use Azure Arc-enabled SQL Server capabilities where supported in the tenant/region and validated during the pilot. Regardless of that evaluation, capture the minimum operational signals below.

| SQL domain | Required indicators |
|---|---|
| SQL service health | SQL Server service, SQL Agent, listener/port availability. |
| CPU and OS resource | CPU, memory, disk latency, throughput, available disk. |
| Sessions/connections | Active user sessions, client host, application/program name, login, client address. |
| Blocking | Blocking session ID, blocked duration, wait type, blocking chain. |
| Waits | Top waits by delta over collection interval, not just cumulative totals. |
| I/O | File read/write latency, reads/writes, throughput, database-file location. |
| Database files | Data, log, and `tempdb` size; autogrowth events; available disk capacity. |
| Errors | SQL error log signals: deadlocks, severity 17+, login failures, I/O errors, backup failures. |
| SQL jobs | SQL Agent job failures and abnormal duration, where SQL Agent is used. |
| Backup status | Most recent successful full/differential/log backup and backup-job failure status. |

## 7.2 Custom DMV Collection Pattern

Where native SQL monitoring does not provide required details, use a controlled collector:

```text
Scheduled SQL Agent job, secure collector service, or Hybrid Worker
       ↓
Read-only SQL account executes approved DMV queries
       ↓
Collector converts result set to JSON
       ↓
Logs Ingestion API
       ↓
Data Collection Endpoint + Data Collection Rule
       ↓
Custom Log Analytics tables
```

Suggested custom tables:

- `SQLSessionActivity_CL`
- `SQLBlocking_CL`
- `SQLWaitStatsDelta_CL`
- `SQLFileIo_CL`
- `SQLDatabaseFileGrowth_CL`
- `SQLAgentJobStatus_CL`
- `SQLBackupStatus_CL`

## 7.3 Critical Security Requirements for DMV Collection

- Use a dedicated **read-only monitoring identity**.
- Grant only minimum SQL permissions required for selected DMVs, often `VIEW SERVER STATE`; validate per query and SQL version.
- Do not ingest full SQL statement text by default.
- Query text may contain personal, financial, customer, or proprietary values.
- If top-query analysis is included, store query hash, database name, duration, CPU, execution count, and query-plan metadata first; enable sanitized query-text collection only after data-classification approval.
- Store Azure authentication material in Key Vault or use managed identity where the execution model supports it.
- Restrict custom-table read access through Azure RBAC and workspace design.

## 7.4 SQL Collection Frequencies

| Dataset | Initial frequency |
|---|---:|
| Active sessions and blocking | 1–5 minutes |
| Wait-stat deltas | 5 minutes |
| File I/O stats | 5 minutes |
| File size/autogrowth state | 5–15 minutes |
| Backup status | 15–60 minutes |
| SQL Agent job status | 5–15 minutes |
| Top expensive query summary | 15–60 minutes |

Do not collect high-cardinality session and query data indefinitely at a one-minute frequency without a cost and usefulness review.

---

# 8. Workstream 5 — Certificates, Windows Accounts, and Service-Account Exposure

## 8.1 Certificate Monitoring

Discover and track certificates used by:

- IIS HTTPS bindings.
- ERP and middleware services.
- Reverse proxies/load balancers where accessible.
- SQL Server TLS encryption configuration where applicable.
- Windows Local Machine certificate stores.

Track:

- Subject/SAN.
- Issuer.
- Thumbprint.
- Store/location.
- Friendly name.
- `NotAfter` expiry date.
- Key availability where permitted.
- IIS binding association.
- Server and application owner.

Initial alerts:

| Certificate condition | Action |
|---|---|
| Expiry within 60 days | Informational/planning notification. |
| Expiry within 30 days | Warning; create work item or low-priority ticket. |
| Expiry within 14 or 7 days | High-priority alert and owner escalation. |
| Already expired or TLS endpoint check failing | Critical incident. |

## 8.2 Password and Login Management Boundaries

The pilot should **never collect, transmit, display, or track actual passwords**.

Instead, monitor **identity risk and compliance posture**:

- Windows service logon account.
- IIS application-pool identity.
- Scheduled-task run-as account.
- SQL Agent proxy/job account where applicable.
- Account enabled/disabled/locked status.
- Password-expiry date or policy exposure where directory policy permits.
- Last password set.
- Whether the account is a gMSA.
- Whether the service is failing due to logon/authentication errors.
- Failed-login events and account lockouts.

## 8.3 Recommended Identity Controls

1. Use **Windows LAPS** for local administrator accounts.
2. Prefer **gMSA** for eligible Windows services, IIS app pools, and scheduled tasks.
3. Avoid using shared human accounts for application services.
4. Monitor Windows Event Log and SQL error-log authentication failures.
5. Place monitoring credentials, service principals, and API secrets in Azure Key Vault where feasible.
6. Create an ownership matrix for every non-human identity.

---

# 9. Workstream 6 — Alerting, ServiceNow, Teams, and Outlook

## 9.1 Alert Design Principles

Alerts must be actionable. Every alert should include:

- Application/service name.
- Server name and environment.
- Affected dependency.
- Severity.
- Threshold or observed value.
- Start time and duration.
- Link to Azure Workbook or Log Analytics query.
- Recommended first-response runbook.
- Support assignment group.
- Deduplication/correlation key.

## 9.2 Alert Classes

| Alert class | Example |
|---|---|
| Availability | ERP API health endpoint returns HTTP 500 or fails for three consecutive checks. |
| Connectivity | ERP app cannot connect to SQL TCP port or has sustained probe loss. |
| Capacity | SQL log drive below critical capacity threshold. |
| Performance | SQL I/O latency and `PAGEIOLATCH`/write-log waits exceed baseline. |
| Runtime | ERP Windows service stopped or IIS app pool unavailable. |
| Security lifecycle | Certificate expires within 14 days; service account nearing expiry. |
| Database operations | SQL backup or SQL Agent job fails. |

## 9.3 ITSM Workflow

Implement one end-to-end path:

```text
Azure Monitor alert
   → Action Group
      → Logic App or approved ServiceNow integration
         → Create or update ServiceNow incident
         → Post alert summary to Teams channel
         → Email escalation mailbox
```

The ServiceNow integration design must prevent an incident storm. Use alert grouping/deduplication based on:

- Application.
- Environment.
- Resource/server.
- Alert rule.
- Dependency path.
- Time window.

Test both:
- Incident creation.
- Incident update/auto-resolution when the alert clears.

---

# 10. Workstream 7 — Dashboards, Workbooks, and Operations Runbooks

## 10.1 Pilot Workbook Design

Create a primary **ERP End-to-End Operations Workbook** with tabs or sections:

1. **Executive/service health**
   - Application availability.
   - Open critical alerts.
   - Service dependency status.
2. **Windows infrastructure**
   - CPU, memory, uptime, services, event errors.
3. **Disk and capacity**
   - Free-space heat map.
   - Disk latency.
   - Volume growth.
   - SQL data/log/tempdb capacity.
4. **API and network**
   - TCP dependency test status.
   - REST success rate.
   - API response time.
   - DNS/TLS failure indication.
5. **SQL performance**
   - Sessions by application host.
   - Blocking.
   - waits.
   - I/O latency.
   - database/log-file growth.
   - backup/job status.
6. **Certificate and identity lifecycle**
   - Certificates nearing expiry.
   - Service accounts approaching expiry.
   - Authentication failures.
7. **Incident diagnostics**
   - Saved KQL queries and links to on-demand connection-state snapshots.

## 10.2 Operations Runbooks

Produce concise Tier 1/Tier 2 procedures for:

- Disk-space critical alert.
- ERP REST API unavailable.
- TCP/port dependency failure.
- SQL blocking/high wait/slow I/O.
- SQL backup or SQL Agent job failure.
- Certificate nearing expiry.
- Service-account authentication failure.
- Azure Arc/AMA telemetry missing.

---

# 11. High-Value Additional Candidates Added to the Pilot

The following were not explicitly requested initially but are strong candidates because they frequently cause ERP outages and materially improve end-to-end diagnosis.

| Candidate | Why it should be included |
|---|---|
| **Windows service and IIS app-pool monitoring** | A server can be healthy while the ERP service or application pool is stopped. |
| **SQL backup monitoring** | Successful database availability monitoring is incomplete if recovery capability is unknown. |
| **SQL Agent/batch-job monitoring** | ERP processes often depend on scheduled jobs, integrations, and overnight batches. |
| **DNS resolution checks** | Incorrect DNS records or stale load-balancer resolution can look like random connectivity failure. |
| **TLS handshake validation** | A port can be open while HTTPS fails due to expired/mismatched certificates or protocol issues. |
| **Application log/error-pattern monitoring** | Vendor application errors may appear before CPU/network/database symptoms. |
| **Time synchronization monitoring** | Clock drift causes authentication, certificate, Kerberos, token, and distributed logging problems. |
| **Telemetry health monitoring** | Monitor Arc agent, AMA, ingestion latency, and collector success. A silent monitoring failure is an operational risk. |
| **Configuration baseline/change tracking** | Capture selected changes: firewall rules, service startup types, certificate binding, SQL configuration, and application service identity. This should be limited and approved to avoid excessive scope. |
| **Backup-volume and log-retention checks** | Backup accumulation and uncontrolled application logs are common disk-exhaustion causes. |

---

# 12. Items Explicitly Deferred from the Pilot

To keep the pilot practical, defer these unless they are an existing requirement:

- Full network packet capture or deep packet inspection.
- Full distributed tracing inside vendor ERP code, unless vendor-supported instrumentation exists.
- Full SIEM/SOC implementation.
- Endpoint protection/Defender rollout.
- Full configuration-management database replacement.
- Enterprise certificate authority redesign.
- Complete migration of service identities to gMSA.
- Full production rollout and high-availability design for all custom collectors.

These may become phase-two initiatives after the pilot validates the operating model.

---

# 13. Delivery Phases

## Phase 0 — Discovery and Design

- Identify servers, applications, owners, dependency paths, ports, APIs, SQL instances, service accounts, certificates, and disk volumes.
- Confirm security, firewall, proxy, Azure subscription, RBAC, and data-retention requirements.
- Agree on baseline thresholds and severity model.
- Obtain vendor guidance for safe health endpoints and relevant ERP logs/performance counters.

**Deliverables:** dependency matrix, monitoring requirements catalogue, security design, pilot architecture, test plan.

## Phase 1 — Azure and Agent Foundation

- Create Azure resources.
- Onboard Arc.
- Deploy AMA.
- Configure base DCRs.
- Validate telemetry arrival and agent health.

**Deliverables:** onboarding runbook, DCR baseline, inventory dashboard.

## Phase 2 — Windows, Disk, Services, and Logs

- Enable OS counters, disk counters, service checks, IIS/application logs, Windows events.
- Build basic dashboards and alerts.

**Deliverables:** infrastructure workbook, disk alerts, service-health alerts.

## Phase 3 — Network and REST Monitoring

- Configure TCP dependency probes.
- Build internal synthetic REST health checker.
- Implement on-demand connection snapshot capability.
- Validate failures through controlled testing.

**Deliverables:** dependency dashboard, API health dashboard, network diagnostic runbook.

## Phase 4 — SQL Monitoring

- Implement native Arc SQL monitoring features where validated.
- Build minimum custom DMV collection.
- Add SQL backup/job/status monitoring.
- Establish disk-to-SQL correlation.

**Deliverables:** SQL workbook, SQL alert rules, secured DMV collector design.

## Phase 5 — Certificates, Identity, and ITSM Integration

- Implement certificate inventory and expiry alerts.
- Implement service-account risk/compliance inventory.
- Integrate alerts with ServiceNow, Teams, and Outlook.
- Test deduplication, assignment, escalation, and auto-resolution.

**Deliverables:** lifecycle dashboard, ServiceNow workflow, alert-routing guide.

## Phase 6 — Operational Acceptance

- Run tabletop scenarios and controlled fault injection.
- Validate alert-to-ticket timing.
- Review cost and ingestion.
- Document gaps, risks, and rollout recommendation.

**Deliverables:** pilot report, architecture decision record, production rollout backlog.

---

# 14. Technical Corrections and Guardrails

A few implementation guardrails are important:

1. **DCRs collect and transform data; they do not themselves execute PowerShell or SQL scripts.**  
   Custom scripts require a controlled execution method—such as SQL Agent, Task Scheduler, Azure Automation Hybrid Worker, or an approved collector service—and then use the Logs Ingestion API/DCR to ingest results.

2. **Do not assume public availability tests represent an internal network path.**  
   Internal ERP APIs require a probe located inside the relevant on-premises network segment.

3. **Do not collect passwords or unrestricted SQL text.**  
   Collect expiry/compliance status and sanitized operational telemetry only.

4. **Collect SQL wait-stat deltas, not only cumulative wait totals.**  
   Cumulative DMV counters without baselining produce misleading results.

5. **Start with a controlled data model and tags.**  
   Unrestricted event logs, process data, query text, and high-frequency DMV snapshots can cause high cost and create data-governance issues.

6. **Monitor the monitoring solution.**  
   Alert when Azure Arc/AMA is disconnected, custom collectors fail, data ingestion stops, or Log Analytics latency exceeds the agreed operating threshold.

---

## External capability validation

Here’s a detailed overview of the Azure Monitor capabilities that currently support monitoring on-premises Windows servers through Azure Arc, including how you can leverage Azure Monitor Agent, VM Insights, SQL Server monitoring, and alert integration with ServiceNow and Microsoft Teams:

### 1. **Azure Monitor Agent on Azure Arc-Enabled Servers**
- The Azure Monitor Agent (AMA) is the recommended monitoring agent for Azure Arc-enabled servers (replacing the legacy Microsoft Monitoring Agent).
- The agent can collect both performance metrics and Windows event logs (customizable via Data Collection Rules, DCRs) and stream them to a Log Analytics workspace in Azure Monitor.
- Installation and management can be done via the Azure portal, Azure CLI, ARM templates, or automated at scale using Azure Policy[[1]](https://docs.azure.cn/en-us/azure-arc/servers/azure-monitor-agent-deployment)[[2]](https://jumpstart.azure.com/azure_arc_jumpstart/azure_arc_servers/day2/arc_monitoring).

### 2. **VM Insights (Virtual Machine Insights)**
- VM Insights gives deep visibility into the performance and health of Windows servers, including those running on-prem and managed by Azure Arc.
- Provides curated dashboards with CPU, memory, disk, and network metrics, as well as process and dependency mapping when the Dependency Agent is also deployed.
- Works seamlessly for Arc-connected servers once enabled in Azure Monitor and the agent(s) are in place[[3]](https://learn.microsoft.com/en-us/azure/azure-arc/servers/cloud-native/monitor-alerts)[[4]](https://learn.microsoft.com/en-us/windows-server/manage/windows-admin-center/azure/azure-monitor)[[5]](https://dev.to/jimiog/monitoring-azure-arc-enabled-windows-with-vm-insights-494g).

### 3. **SQL Server Monitoring**
- With Azure Arc-enabled SQL Server, you can use Azure Monitor to collect insights on performance, query health, and errors for both on-premises and multi-cloud SQL Server instances.
- This requires enrollment of your SQL Server instance with Azure Arc and enables integration with Azure Monitor Workbooks and alerts[[6]](https://azure.microsoft.com/en-us/products/azure-arc/).

### 4. **Alert Integration with ServiceNow and Microsoft Teams**
- Azure Monitor’s alerting system fully supports the creation of alert rules on data ingested from Arc-enabled servers (metrics or log query–based).
- Alerts can be routed to:
  - **ServiceNow** via Azure Logic Apps or direct ITSM integration, allowing automated ticket creation when specific conditions are met.
  - **Microsoft Teams** using Azure Logic Apps, which post messages to Teams channels in real time for alert notification and collaboration.
- Built-in connectors for both ServiceNow and Teams are available in Logic Apps, and can be triggered by alert actions in Azure Monitor’s alerting workflow[[7]](https://azure.microsoft.com/en-us/products/monitor/)[[8]](https://docs.azure.cn/en-us/azure-monitor/).

### 5. **Other Capabilities**
- **Custom Workbooks:** Visualize and share data from on-prem Arc-connected servers, combining metrics, logs, and context across teams.
- **Update Management:** Arc-enabled servers can participate in central patch management using Azure Update Manager.
- **Security and Governance:** Integrates with Microsoft Defender for Cloud and policy compliance solutions for on-prem and multi-cloud workloads.

---

**Summary Table**

| Capability                        | Available for Arc-Enabled Windows Servers? | Requirements                         |
|------------------------------------|-------------------------------------------|--------------------------------------|
| Azure Monitor Agent (AMA)          | Yes                                       | AMA, DCR, Arc onboarded server       |
| VM Insights (Performance/Health)   | Yes                                       | AMA + Dependency Agent recommended   |
| SQL Server Monitoring              | Yes (If SQL instance is Arc-enabled)      | SQL Server onboarded via Arc         |
| ServiceNow Integration (Alerts)    | Yes (via Logic Apps/ITSM Connector)       | Logic Apps or native ITSM connector  |
| Microsoft Teams Integration        | Yes (via Logic Apps)                      | Logic Apps + Teams connector         |

**References:**
- Azure documentation and monitoring setup guides[[3]](https://learn.microsoft.com/en-us/azure/azure-arc/servers/cloud-native/monitor-alerts)[[4]](https://learn.microsoft.com/en-us/windows-server/manage/windows-admin-center/azure/azure-monitor)[[1]](https://docs.azure.cn/en-us/azure-arc/servers/azure-monitor-agent-deployment)[[2]](https://jumpstart.azure.com/azure_arc_jumpstart/azure_arc_servers/day2/arc_monitoring)[[8]](https://docs.azure.cn/en-us/azure-monitor/)[[5]](https://dev.to/jimiog/monitoring-azure-arc-enabled-windows-with-vm-insights-494g)

---

1. [Deploy Azure Monitor agent on Arc-enabled servers - Azure Arc](https://docs.azure.cn/en-us/azure-arc/servers/azure-monitor-agent-deployment)
2. [Azure Arc Jumpstart](https://jumpstart.azure.com/azure_arc_jumpstart/azure_arc_servers/day2/arc_monitoring)
3. [Cloud-native monitoring and alerts with Azure Arc-enabled servers](https://learn.microsoft.com/en-us/azure/azure-arc/servers/cloud-native/monitor-alerts)
4. [Monitor servers and configure alerts with Azure Monitor from Windows ...](https://learn.microsoft.com/en-us/windows-server/manage/windows-admin-center/azure/azure-monitor)
5. [Monitoring Azure Arc-Enabled Windows with VM Insights](https://dev.to/jimiog/monitoring-azure-arc-enabled-windows-with-vm-insights-494g)
6. [Azure Arc | Microsoft Azure](https://azure.microsoft.com/en-us/products/azure-arc/)
7. [Azure Monitor | Microsoft Azure](https://azure.microsoft.com/en-us/products/monitor/)
8. [Azure Monitor documentation - Azure Monitor | Azure Docs](https://docs.azure.cn/en-us/azure-monitor/)
