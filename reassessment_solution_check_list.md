# Reassessment: Critical Items Still Required for the Pilot

The pilot scope is technically well-developed, but it is not yet execution-ready. The main remaining gaps are **requirements baselining, dependency validation, security/data governance, operational ownership, fault testing, cost modeling, and platform capability verification**.

## 1. Resolve Azure Capability Assumptions Before Build

Several previously discussed features should be treated as **pilot validation items**, not assumed capabilities:

- Validate whether Azure Network Watcher Connection Monitor supports the exact on-premises-to-on-premises endpoint topology required.
- Validate the supported method for internal/private REST availability testing.
- Confirm the current availability and licensing of Arc-enabled SQL monitoring features in the target Azure region.
- Confirm whether each custom Logs Ingestion API stream requires a Data Collection Endpoint in the selected architecture.
- Confirm the current API version, DCR schema syntax, custom-table provisioning method, and ingestion limits.
- Validate ServiceNow integration method: native connector, Logic App, or the organization’s existing integration platform.
- Confirm whether Teams notifications will use an approved Logic App/connector pattern rather than an unsupported legacy webhook pattern.

These should be documented as **architecture assumptions with acceptance tests**.

---

# 2. Missing Workstream: Formal Dependency and Service Inventory

Before deploying collectors, create a dependency catalogue for every pilot application.

## Required inventory

| Category | Required information |
|---|---|
| Servers | Hostname, IP, OS version, environment, Azure Arc status |
| Application roles | ERP, API, IIS, middleware, SQL, batch, file-processing roles |
| Services | Windows service name, startup mode, service account, owner |
| APIs | URL, method, authentication, expected response, health endpoint |
| Network paths | Source, destination, protocol, port, DNS name, firewall owner |
| SQL | Instance, database, listener, port, authentication, owner |
| Certificates | Store, thumbprint, binding, owner, renewal process |
| Directories | Log, queue, backup, archive, temporary-file paths |
| Identities | Domain accounts, gMSAs, local accounts, Entra identities |
| Business criticality | Critical, high, medium, low |
| Support ownership | Infrastructure, ERP, middleware, database, security, vendor |

Do not configure monitoring from informal diagrams or individual administrator knowledge. The dependency catalogue should be the source for:

- DCR scope.
- Connection-monitor tests.
- REST tests.
- Alert rules.
- Workbook filters.
- ServiceNow assignment groups.
- Support runbooks.

---

# 3. Missing Workstream: Monitoring Requirements and SLO Baselines

The pilot needs agreed thresholds based on normal behavior.

## Establish baselines for

- CPU utilization.
- Available memory.
- Disk latency.
- SQL wait classes.
- SQL blocking duration.
- API response time.
- API error rate.
- TCP connection latency.
- Folder growth rate.
- Certificate renewal lead time.
- Service-account expiry lead time.
- Expected backup completion time.
- Normal batch-job duration.

Avoid using generic thresholds without workload validation. For example:

- A 10% disk threshold may be too late for a high-volume SQL server.
- A 500 ms API response may be normal for one ERP endpoint and unacceptable for another.
- A SQL wait type can be normal in one workload and critical in another.

Define operational targets such as:

```text
ERP API availability: >= 99.9%
ERP API response time: p95 <= 2 seconds
ERP-to-SQL connectivity: 100% successful probes over 5 minutes
SQL blocking: alert when blocking exceeds 60 seconds
Certificate warning: 30 days
Certificate critical: 14 days
```

These values are examples only and must be approved by the application and operations owners.

---

# 4. Missing Workstream: Telemetry Health and Monitoring the Monitoring Platform

The design must detect when telemetry itself is missing.

Monitor:

- Azure Arc agent heartbeat.
- Azure Monitor Agent health.
- DCR association status.
- Agent extension status.
- Last successful ingestion time.
- Custom collector execution status.
- Collector records collected versus records submitted.
- Logs Ingestion API failures.
- DCR schema errors.
- Data latency from collection to workspace availability.
- Server clock synchronization.
- Disk space used by local agent buffers/logs.

Create a dedicated `CollectorHealth_CL` table and alert on:

```text
No collector execution within expected interval
Collector execution failed
Records collected = 0 unexpectedly
Records submitted < records collected
DCR ingestion failures
AMA heartbeat missing
Arc connection unhealthy
```

A monitoring system that silently stops collecting data is itself an operational risk.

---

# 5. Missing Workstream: Identity and Access Architecture

The pilot needs a formal identity matrix.

## Define identities for

1. Arc onboarding.
2. AMA deployment.
3. DCR administration.
4. Logs Ingestion API submission.
5. SQL DMV collection.
6. Active Directory account-status lookup.
7. Microsoft Graph/LAPS metadata lookup.
8. Azure Automation execution.
9. ServiceNow integration.
10. Workbook and Log Analytics query access.

For each identity document:

- Human or non-human identity.
- Authentication method.
- Required permissions.
- Resource scope.
- Secret/certificate storage.
- Rotation process.
- Owner.
- Break-glass procedure.
- Audit requirements.

Use separate identities for:

- **Ingestion** — write telemetry.
- **Query** — read workspace data.
- **Administration** — manage DCRs and tables.

Do not give a collector broad Contributor access to the workspace merely because it needs to submit records.

---

# 6. Missing Workstream: Data Classification and Privacy Review

The proposed telemetry can contain sensitive information:

- Windows account names.
- SQL login names.
- Client IP addresses.
- Hostnames and internal network topology.
- IIS URLs and query strings.
- ERP transaction identifiers.
- SQL query text.
- Certificate subject names.
- Directory paths.
- Vendor application error messages.

Before ingestion, determine:

- Whether the data is confidential.
- Whether it contains personal information.
- Whether SQL query text is permitted.
- Whether service-account distinguished names may be stored.
- Whether the Log Analytics region satisfies data-residency requirements.
- Who can query the workspace.
- How long each table should be retained.
- Whether data must be masked or transformed before storage.

For SQL monitoring, start with:

- Query hash.
- Database name.
- Duration.
- CPU.
- Execution count.
- Wait category.

Do not collect full query text by default.

---

# 7. Missing Workstream: Network and Firewall Design

Produce an explicit connectivity matrix covering:

| Source | Destination | Port | Direction | Purpose | Proxy required |
|---|---|---:|---|---|---|
| Arc-enabled server | Azure Arc endpoints | TCP 443 | Outbound | Arc management |
| AMA server | Azure Monitor endpoints | TCP 443 | Outbound | Standard telemetry |
| Custom collector | DCE/ingestion endpoint | TCP 443 | Outbound | Custom telemetry |
| Probe host | ERP API | TCP 443 | Internal | REST test |
| ERP server | Middleware | Application port | Internal | ERP integration |
| Middleware | SQL Server | SQL port | Internal | Database access |

Validate:

- DNS.
- Firewall allowlists.
- Proxy authentication.
- TLS inspection.
- Certificate trust.
- Egress restrictions.
- Private Link requirements.
- Regional endpoints.
- Failure behavior when Azure is unavailable.

The pilot should test both direct outbound access and the organization’s proxy path if both are used.

---

# 8. Missing Workstream: Alert Engineering and Event Correlation

The pilot currently identifies alert types, but it needs a complete alert specification.

Every alert rule should define:

- Rule name.
- Data source/table.
- Query.
- Evaluation frequency.
- Lookback period.
- Threshold.
- Severity.
- Suppression period.
- Grouping key.
- ServiceNow assignment group.
- Teams channel.
- Escalation owner.
- Auto-resolution behavior.
- Runbook link.
- Maintenance-window behavior.

## Prevent alert storms

Use correlation and suppression for common dependency failures.

Example:

```text
SQL server unavailable
    ├── ERP-to-SQL port failures
    ├── Middleware-to-SQL port failures
    ├── API errors
    └── ERP application errors
```

The underlying SQL outage should not create four unrelated critical incidents. Define a primary incident and related symptoms.

Test:

- Alert creation.
- Repeated event suppression.
- Incident deduplication.
- Alert recovery.
- Maintenance windows.
- Dependency outage correlation.
- ServiceNow assignment.
- Teams formatting.
- Email escalation.

---

# 9. Missing Workstream: Application-Level Monitoring

The pilot correctly adds TCP and REST checks, but should also define the minimum application observability available from the ERP vendor.

Ask the vendor for:

- Supported health endpoint.
- Read-only synthetic transaction.
- Application-specific event log channel.
- Performance counters.
- Error-code catalogue.
- Correlation ID support.
- Request ID propagation.
- Vendor-supported log format.
- API authentication method for monitoring.
- Service restart and recovery behavior.
- Application-specific queue and batch metrics.

TCP monitoring proves reachability. REST monitoring proves HTTP behavior. Neither necessarily proves the ERP business function is healthy.

Use this monitoring hierarchy:

```text
Layer 1: Server reachable
Layer 2: Port listening
Layer 3: TCP connection succeeds
Layer 4: TLS handshake succeeds
Layer 5: HTTP response succeeds
Layer 6: API response content is valid
Layer 7: Safe business transaction succeeds
```

The pilot should explicitly identify which layers are implemented.

---

# 10. Missing Workstream: SQL Operations Beyond DMV Collection

Add or explicitly defer:

- SQL database backup success/failure.
- SQL Agent job failures.
- Database state: online, recovery pending, suspect.
- Transaction-log reuse wait.
- Autogrowth events.
- Tempdb contention.
- Deadlocks.
- SQL error-log severity events.
- Login failures.
- Database integrity-check status.
- Always On or clustering status, if applicable.
- SQL Server service and listener status.
- Encryption/TLS configuration, if required.

Also define how SQL telemetry is collected:

- SQL Agent.
- Hybrid Worker.
- Secure collector service.
- Native Azure Arc SQL monitoring.
- Vendor tool.

The SQL monitoring account should be read-only and should not retrieve unrestricted query text or sensitive values.

---

# 11. Missing Workstream: Certificate Operational Ownership

Certificate inventory and expiry alerts are not enough. Add:

- Certificate owner.
- Application/service binding.
- Renewal method.
- Renewal request lead time.
- Replacement validation.
- Rollback procedure.
- Private-key access requirements.
- Certificate chain validation.
- TLS protocol and cipher requirements.
- Post-renewal service reload/restart procedure.
- Confirmation that the renewed certificate is actually bound to the endpoint.

Include an endpoint-level TLS test, because a valid certificate in the store may not be the certificate actually served by IIS or middleware.

---

# 12. Missing Workstream: Local Collection Failure and Performance

The scripts should be tested for:

- Large certificate stores.
- Very large log directories.
- Millions of files.
- Access-denied folders.
- Reparse points and junctions.
- Long-running scans.
- Server CPU impact.
- Memory consumption.
- Network outage.
- Azure throttling.
- Expired collector certificate.
- DCR schema mismatch.
- Duplicate data submissions.

For folder scanning, define:

- Maximum scan duration.
- Maximum directory depth.
- Reparse-point handling.
- Excluded directories.
- File count limit.
- Whether to scan files newer than a time window only.
- Whether to calculate size from filesystem metadata or use a storage metric.

---

# 13. Missing Workstream: Deployment and Lifecycle Management

The pilot needs a repeatable deployment process.

Use infrastructure as code for:

- Resource groups.
- Workspace.
- Tables.
- DCRs.
- DCEs.
- Role assignments.
- Alert rules.
- Action Groups.
- Workbooks.
- Automation accounts.
- Key Vault objects.

Use source control for:

- PowerShell scripts.
- DCR definitions.
- Table schemas.
- KQL queries.
- Workbook JSON.
- Test scripts.
- Runbooks.
- Configuration files.

Define environments:

```text
Development → Pilot/Test → Production
```

Do not manually modify production DCRs and tables without change control.

---

# 14. Missing Workstream: Version and Compatibility Matrix

Document compatibility for:

- Windows Server versions.
- Azure Connected Machine agent.
- Azure Monitor Agent.
- PowerShell version.
- Az PowerShell modules.
- LAPS module.
- Microsoft Graph PowerShell module.
- SQL Server versions.
- SQL Server Agent.
- IIS versions.
- ERP vendor application version.
- Azure region.
- DCR/API version.
- ServiceNow connector version.

The current PowerShell ingestion script should be tested with the exact module versions that will be supported operationally.

---

# 15. Missing Workstream: Cost and Capacity Model

Estimate:

```text
Daily ingestion =
  record count
  × average record size
  × collection frequency
  × number of servers
```

Measure separately:

- Windows event data.
- IIS logs.
- Application logs.
- Performance counters.
- SQL DMV records.
- Network snapshots.
- Certificate inventory.
- Account inventory.
- Folder metrics.
- Collector-health records.

Pilot cost review should include:

- Log Analytics ingestion.
- Interactive retention.
- Archive retention.
- Alert-rule evaluation.
- Workbooks.
- Logic Apps.
- Automation.
- Key Vault.
- Azure Arc-enabled server charges where applicable.
- Network traffic and proxy infrastructure.
- ServiceNow integration licensing or transaction impact.

---

# 16. Missing Workstream: Disaster Recovery and Service Continuity

Define what happens if:

- Azure Monitor is unavailable.
- The Log Analytics Workspace is unavailable.
- The on-premises network to Azure is unavailable.
- The collector host is unavailable.
- Azure Automation is unavailable.
- ServiceNow integration fails.
- Teams notification fails.
- The DCR is accidentally changed.
- A custom table is deleted or modified.

The on-premises ERP application must continue operating if monitoring is unavailable. Monitoring must not become a runtime dependency for ERP, middleware, or SQL services.

Define:

- Local buffering expectations.
- Alert delivery fallback.
- Critical alerts through an alternate path.
- Configuration backup.
- DCR/table redeployment.
- Workspace recovery and retention requirements.

---

# 17. Missing Workstream: Pilot Test and Acceptance Plan

Use controlled fault injection, preferably in a non-production environment.

## Required scenarios

1. Stop ERP Windows service.
2. Stop IIS app pool.
3. Block ERP-to-middleware port.
4. Block middleware-to-SQL port.
5. Make REST endpoint return HTTP 500.
6. Expire or replace a test certificate.
7. Disable a test service account.
8. Create a controlled SQL blocking session.
9. Fill a test log directory.
10. Create SQL transaction-log growth.
11. Fail a SQL Agent backup test job.
12. Stop AMA.
13. Block Azure outbound connectivity.
14. Submit an invalid DCR payload.
15. Break the ServiceNow integration.
16. Generate duplicate alerts.

For each scenario record:

- Detection time.
- Ingestion delay.
- Alert time.
- ServiceNow ticket time.
- Teams/email delivery time.
- Diagnostic usefulness.
- False positives.
- Operator action.
- Recovery behavior.

---

# 18. Revised Pilot Work Breakdown

The pilot should now include these workstreams:

1. Requirements and dependency discovery.
2. Azure subscription, region, and governance foundation.
3. Network and firewall connectivity.
4. Arc and AMA onboarding.
5. Native Windows telemetry.
6. Disk capacity and I/O monitoring.
7. Windows services, IIS, and application logs.
8. TCP dependency monitoring.
9. REST/API synthetic monitoring.
10. On-demand connection diagnostics.
11. SQL Server health, DMV, job, and backup monitoring.
12. Certificate inventory and endpoint TLS validation.
13. Service-account and LAPS metadata monitoring.
14. Custom tables, DCRs, DCEs, and ingestion security.
15. Workbooks and KQL operating queries.
16. Alert engineering and correlation.
17. ServiceNow, Teams, and Outlook integration.
18. Collector and agent health monitoring.
19. Data classification and access governance.
20. Cost and capacity measurement.
21. Fault injection and operational acceptance.
22. Production rollout decision.

---

# Final Assessment

The technical direction is sound, but the pilot should not proceed directly to scripting and dashboard construction. The immediate next deliverable should be a **Pilot Technical Design Package** containing:

1. Dependency and asset inventory.
2. Data-source and schema catalogue.
3. Network/endpoint matrix.
4. Identity and RBAC matrix.
5. Security/data-classification assessment.
6. DCR/table deployment templates.
7. Collector execution design.
8. Alert catalogue.
9. ServiceNow integration design.
10. Workbook wireframes.
11. Fault-injection test plan.
12. Cost model.
13. Operations and support runbooks.
14. Acceptance criteria.

The most important gaps are **monitoring-health detection, precise application dependency discovery, identity/data governance, alert correlation, SQL backup/job monitoring, application-level health checks, and a controlled acceptance test plan**.

---

When deploying Azure Monitor for on-premises Windows servers through Azure Arc, several critical design considerations and limitations must be validated to ensure a robust, secure, and cost-effective monitoring solution. Here is a breakdown across your mentioned aspects:

---

## 1. Agent Connectivity

- **Network Requirements:**  
  Ensure that your on-premises servers can reach Azure endpoints required for Arc and Azure Monitor. This typically needs outbound connectivity over HTTPS (TCP 443) to Azure data collection and management endpoints.  
- **Proxy Support:**  
  Validate that network proxies/firewalls allow necessary traffic, and configure agent proxy settings if needed.
- **Intermittent Connectivity:**  
  Azure Monitor agent buffers some data, but extended outages may lead to data loss. Test how long your network can be unavailable before data loss occurs.
- **Hybrid Requirements:**  
  Confirm that hybrid identities and hybrid networking (e.g., VPN, ExpressRoute) are available and reliable for secure communication.

---

## 2. Azure Arc Agent and Extensions

- **Supported OS/Versions:**  
  Check the compatibility matrix for Arc agent and Azure Monitor agent on your Windows Server versions .
- **Agent Updates:**  
  Plan for how agents and extensions will be updated and maintained.
- **Resource Limits:**  
  There are limits on the number of Arc-connected machines per resource group and region.

---

## 3. Data Collection Rules (DCRs)

- **Granularity:**  
  DCRs are used to define which data (logs, metrics) is collected and sent to which Log Analytics Workspace. Design DCRs to avoid overlapping/conflicting rules.
- **Scoping:**  
  DCRs can target machines via Azure Resource Graph queries, tags, or explicit resource lists. Ensure your resource grouping logic is maintainable.
- **Limitations:**  
  There are quotas on the number of DCRs per subscription and on target resources per DCR.
- **Custom vs. Built-in:**  
  Validate that all required data types (event logs, performance counters, etc.) are supported in DCRs.

---

## 4. Custom Log Ingestion

- **Format Support:**  
  New custom log ingestion is via DCRs and the AMA. Some legacy features are not yet fully available.
- **Log Size/Rate:**  
  Check ingestion limits for custom logs, including message size and rate per agent.
- **Transformation:**  
  Azure Monitor supports KQL-based transformations, but complex parsing or enrichments may require extra steps.

---

## 5. Alerting

- **Rule Types:**  
  You can configure metric, log, and activity log alerts. Ensure the necessary data is available to trigger alerts.
- **Latency:**  
  Log alerts may have processing delays (typically a few minutes). Validate this matches your operational needs.
- **Action Groups:**  
  Design notification and automation workflows (e.g., email, SMS, Logic Apps) accordingly.
- **Scale:**  
  Beware of alert storms from verbose DCRs—fine-tune alert logic.

---

## 6. Cost Considerations

- **Data Volume:**  
  Costs scale linearly with ingestion volume and retention in Log Analytics. Estimate ingestion volume per agent: logs, metrics, custom logs.
- **DCR Optimization:**  
  Only collect what you need! Overly broad DCRs drive up costs quickly.
- **Retention:**  
  Longer retention in Log Analytics is more expensive; set policies based on business/regulatory needs.
- **Outbound Traffic:**  
  Consider bandwidth and egress costs if collecting large volumes from on-prem to Azure.

---

## 7. Security

- **Identity and Access:**  
  Use least-privilege managed identities or service principals for agent registration.
- **Data Security:**  
  Data in transit is encrypted. Understand your compliance requirements for telemetry sent to Azure.
- **Resource Permissions:**  
  Control who can deploy agents, configure DCRs, or query logs.
- **Log Data Sensitivity:**  
  Consider masking/anonymizing sensitive data before ingestion.

---

## 8. Operational Readiness

- **Monitoring & Support:**  
  Establish monitoring for the agents themselves (heartbeat, failed/degraded status).
- **Incident Response:**  
  Ensure support staff know how to troubleshoot Arc agent and AMA issues.
- **Change Management:**  
  Document procedures for updating agents, changing DCRs, rolling back, and disaster recovery.
- **Integration:**  
  Integrate with ITSM systems for incident and change tracking.

---

## 9. Limitations and Gaps

- **Feature Parity:**  
  Not all Log Analytics capabilities available in Azure VMs apply to Arc-connected servers.
- **Legacy Agents:**  
  The legacy Microsoft Monitoring Agent (MMA/Log Analytics agent) will reach end of support; use Azure Monitor Agent (AMA) only.
- **Regional Availability:**  
  Some features might be limited to specific Azure regions or have region-specific quotas.
- **Scaling:**  
  There are quotas for the number of servers, DCRs, and data streams. Check Azure documentation for up-to-date values.

---

**References**

- [Azure Monitor agent overview](https://learn.microsoft.com/en-us/azure/azure-monitor/agents/azure-monitor-agent-overview)
- [Data Collection Rules in Azure Monitor](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/data-collection-rule-overview)
- [Monitoring Windows computers with Azure Monitor](https://learn.microsoft.com/en-us/azure/azure-monitor/agents/agent-manage)

---

**Summary Table**

| Area | Check/Validate |
|---|---|
| Connectivity | Network/proxy, outbound 443, buffering |
| Agent | Supported OS/version, updates, capacity |
| DCRs | Overlap, target logic, quotas |
| Custom Logs | Format, ingestion limits, transformations |
| Alerting | Data needed, delay, scale, action groups |
| Cost | Ingestion, retention, egress/bandwidth |
| Security | Identity, permissions, sensitive data |
| Ops Readiness | Monitoring, troubleshooting, DR, ITSM |
| Limitations | Feature gaps, legacy deprecation, scaling |

---

**Tip:** Always start with a **pilot deployment** to validate connectivity, DCR efficacy, alerting workflow, and cost impact before scaling up.
