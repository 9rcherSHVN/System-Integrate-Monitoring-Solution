# Executive Project Summary  
## Hybrid End-to-End Monitoring Pilot for On-Premises Windows ERP Platform

### 1. Executive Overview

This project will evaluate and implement a practical hybrid monitoring solution for on-premises Windows servers hosting:

- Third-party ERP application services and REST APIs.
- Web and middleware services.
- Microsoft SQL Server databases.

The proposed platform uses **Azure Arc-enabled servers, Azure Monitor Agent, Data Collection Rules, Log Analytics Workspace, Azure Monitor alerts, and Azure Workbooks**. On-premises servers will remain on-premises; no migration to Azure virtual machines is included in this pilot.

The pilot is intended to provide IT support with centralized, cloud-based visibility into infrastructure health, application dependencies, network connectivity, database performance, storage capacity, certificates, service identities, and incident workflows.

---

## 2. Business and Operational Objectives

The pilot will help IT support:

1. Detect infrastructure and application problems before they become service outages.
2. Monitor end-to-end dependencies between ERP, middleware, web, and SQL tiers.
3. Troubleshoot network and application connectivity during active incidents.
4. Correlate Windows, API, network, SQL, disk, and security-lifecycle telemetry.
5. Provide centralized dashboards and historical trend analysis.
6. Automatically notify support teams through Teams and Outlook.
7. Create or update incidents in ServiceNow.
8. Establish a repeatable monitoring architecture for future production rollout.

---

## 3. Pilot Scope

### 3.1 Representative Pilot Environment

The pilot should include at least:

| Server | Role |
|---|---|
| ERP application server | Hosts ERP services and REST APIs |
| Web/middleware server | Hosts IIS, web services, integration, proxy, or message-processing components |
| SQL Server | Hosts a representative ERP database |
| Optional monitoring utility server | Hosts custom collectors or an Azure Automation Hybrid Runbook Worker if required |

The pilot should use a realistic application dependency chain and representative workload.

---

## 4. Functional Monitoring Scope

### 4.1 Windows Server Runtime Monitoring

Using Azure Monitor Agent and Data Collection Rules:

- CPU utilization.
- Memory utilization and pressure.
- Network interface activity.
- Disk capacity and disk performance.
- Windows operating-system events.
- Application events.
- ERP and middleware Windows services.
- IIS and application-pool status where applicable.
- Server heartbeat and agent health.
- Selected scheduled-task and batch-process status.

### 4.2 Disk and Capacity Monitoring

The pilot will monitor:

- Logical-drive free space.
- Absolute free space.
- Disk latency.
- Disk queue and utilization.
- ERP and middleware log-directory size.
- IIS log growth.
- SQL data, log, and `tempdb` volumes.
- SQL database-file growth.
- Backup-directory capacity.
- Files exceeding defined retention periods.

Disk alerts will use both percentage and absolute free-space thresholds because percentage-only alerts are insufficient for large SQL volumes.

### 4.3 Network and Dependency Monitoring

The pilot will validate:

- ERP-to-middleware TCP connectivity.
- ERP-to-SQL connectivity.
- Middleware-to-SQL connectivity.
- Expected listening ports.
- Connection success/failure.
- Latency and packet-loss trends where supported.
- On-demand active connection diagnostics.
- Process-to-port mapping using controlled PowerShell collection.

The solution will distinguish between:

1. Server reachability.
2. TCP port availability.
3. TLS handshake.
4. HTTP/API response.
5. API functional health.

### 4.4 REST API Monitoring

The pilot will monitor selected ERP and middleware API endpoints for:

- DNS resolution.
- TCP connection.
- TLS certificate validity.
- HTTP response status.
- Response time.
- Optional response-body validation.
- Authentication behavior using an approved non-privileged monitoring identity.

The preferred endpoint is a vendor-supported, read-only health or status endpoint. Business transactions that modify ERP data will not be used for synthetic monitoring.

### 4.5 SQL Server Monitoring

The pilot will combine native Azure monitoring capabilities with controlled custom collection where necessary.

Monitoring will include:

- SQL Server service and SQL Agent status.
- CPU, memory, and disk performance.
- SQL sessions and connection origin.
- Client host and application name.
- Blocking sessions and blocked duration.
- Wait-stat deltas.
- Database and transaction-log usage.
- File-level I/O latency.
- Autogrowth activity.
- `tempdb` indicators.
- SQL errors and login failures.
- SQL Agent job failures.
- Database backup status.
- Database state and availability.

SQL DMV collection will use a read-only monitoring identity and will avoid unrestricted SQL statement text unless specifically approved.

### 4.6 Certificate and Identity Lifecycle Monitoring

The pilot will inventory:

- Windows certificate stores.
- IIS HTTPS bindings.
- Certificate subject, SAN, issuer, thumbprint, and expiry.
- Certificates approaching expiration.
- Windows services, IIS application pools, and scheduled-task identities.
- Domain service-account status.
- Account enablement and lockout status.
- Password last-set and calculated expiry information where permitted.
- gMSA usage and validation.
- Windows LAPS metadata, including backup and expiry status, without retrieving passwords.

Actual passwords, LAPS password values, secrets, tokens, and private keys will not be collected or stored.

---

## 5. Proposed Technical Architecture

```text
On-premises Windows servers
    │
    ├── Azure Arc Connected Machine agent
    ├── Azure Monitor Agent
    ├── Native Windows event/performance collection
    └── Controlled custom collectors
             │
             │ Outbound HTTPS/TLS
             ▼
Azure Monitor ingestion services
    │
    ├── Data Collection Rules
    ├── Data Collection Endpoint where required
    └── Logs Ingestion API for custom telemetry
             │
             ▼
Log Analytics Workspace
    │
    ├── KQL queries
    ├── Azure Monitor Workbooks
    ├── Log and metric alert rules
    └── Historical trend analysis
             │
             ▼
Action Groups / Logic Apps / Integration
    ├── ServiceNow incidents
    ├── Microsoft Teams notifications
    └── Outlook/email escalation
```

### Native collection

Azure Monitor Agent will collect supported:

- Windows performance counters.
- Windows Event Logs.
- IIS logs.
- Compatible application text logs.
- Agent heartbeat and supported infrastructure telemetry.

### Custom collection

PowerShell or another approved collector will be used only where Azure Monitor Agent cannot directly obtain the required information, including:

- SQL Server DMVs.
- Active TCP connection state.
- REST API functional checks.
- Certificate-store inventory.
- Service-account metadata.
- Windows LAPS metadata.
- ERP log-directory size.
- Explicit current service state where event logs are insufficient.

Custom collectors will submit structured JSON records through custom DCR streams into Log Analytics custom tables.

---

## 6. Custom Data Model

The pilot will define and implement the following custom Log Analytics tables:

| Custom table | Purpose |
|---|---|
| `ServerCertificateInventory_CL` | Certificate inventory and IIS binding information |
| `ServiceAccountStatus_CL` | Service, task, application-pool, and account status |
| `LapsDeviceStatus_CL` | Windows LAPS backup and expiry metadata |
| `ErpFolderMetrics_CL` | ERP, middleware, IIS, and backup directory metrics |
| `SQLSessionActivity_CL` | SQL sessions and connection origin |
| `SQLBlocking_CL` | Blocking and blocked-session information |
| `SQLWaitStats_CL` | SQL wait-stat deltas |
| `SQLFileIo_CL` | Database-file I/O measurements |
| `SQLDatabaseFileGrowth_CL` | Database and log-file size/growth |
| `CollectorHealth_CL` | Custom collector execution and ingestion health |
| `RestEndpointHealth_CL` | REST endpoint response and availability results |
| `NetworkConnectionState_CL` | On-demand TCP connection snapshots |

Each table will have:

- UTC `TimeGenerated`.
- Server/application identity.
- Strongly typed columns.
- Collection status.
- Collector version.
- No passwords, secrets, private keys, or unrestricted sensitive payloads.

---

## 7. Dashboard and Reporting Scope

An Azure Monitor Workbook will provide a consolidated operational view containing:

1. **Overall ERP service health**
   - Open critical alerts.
   - API availability.
   - Dependency status.
2. **Windows infrastructure**
   - CPU, memory, disk, network, services, and events.
3. **Storage and capacity**
   - Volume free space.
   - Disk latency.
   - Folder growth.
   - SQL data/log/tempdb capacity.
4. **Network and API**
   - TCP dependency results.
   - REST status and response times.
   - TLS and DNS failures.
5. **SQL performance**
   - Active sessions.
   - Blocking.
   - Waits.
   - I/O latency.
   - Database-file growth.
   - Backup/job status.
6. **Certificates and identities**
   - Expiring certificates.
   - Account expiry risk.
   - LAPS backup status.
   - Authentication failures.
7. **Monitoring platform health**
   - Arc connectivity.
   - AMA heartbeat.
   - Custom collector status.
   - Ingestion failures.

---

## 8. Alerting and ITSM Integration

Alert rules will be designed around actionable operational conditions.

### Alert examples

- ERP or middleware service stopped.
- REST API unavailable or returning errors.
- ERP-to-SQL connection failure.
- Critical disk-space threshold reached.
- Excessive SQL blocking.
- SQL backup or Agent job failure.
- Certificate expiring within defined thresholds.
- Service account disabled, locked, or approaching password expiry.
- LAPS backup metadata missing or expired.
- Custom collector or AMA telemetry missing.
- Azure Arc connection failure.

### Notification flow

```text
Azure Monitor alert
    → Action Group
       → ServiceNow incident or update
       → Teams notification
       → Outlook/email escalation
```

The pilot will validate:

- Incident creation.
- Incident deduplication.
- Alert grouping and suppression.
- Assignment group routing.
- Auto-resolution when alerts clear.
- Maintenance-window behavior.
- Escalation and recovery notifications.

---

## 9. Security and Governance Principles

The implementation will apply:

- Outbound-only server connectivity where feasible.
- TLS-encrypted data transmission.
- Azure RBAC and least-privilege access.
- Separate ingestion, query, and administration identities.
- Managed identity or certificate-based authentication where supported.
- Key Vault or approved enterprise secret storage.
- Read-only SQL monitoring account.
- Metadata-only LAPS access.
- No password or private-key collection.
- Data classification before ingestion.
- Sanitization of SQL, ERP, IIS, and exception data.
- Script signing and source-control management.
- Controlled access to the Log Analytics Workspace.
- Audit logging for DCR, table, script, and alert changes.

---

## 10. Pilot Phases

### Phase 1 — Discovery and Design

- Confirm server and application inventory.
- Document dependencies, ports, APIs, databases, identities, certificates, and directories.
- Define monitoring thresholds and support ownership.
- Confirm security, network, Azure region, retention, and privacy requirements.

### Phase 2 — Azure Foundation

- Create resource groups, workspace, DCRs, tables, and required endpoints.
- Configure RBAC and collector identities.
- Validate firewall, proxy, DNS, and outbound connectivity.

### Phase 3 — Arc, AMA, and Native Monitoring

- Onboard pilot servers to Azure Arc.
- Deploy Azure Monitor Agent.
- Configure baseline DCRs.
- Validate Windows, disk, event, IIS, and service telemetry.

### Phase 4 — Custom Monitoring

- Deploy REST health checks.
- Implement network connection diagnostics.
- Implement SQL DMV and operational-status collectors.
- Implement certificate, identity, LAPS, and folder-size collectors.

### Phase 5 — Dashboards and ITSM

- Build Azure Workbooks.
- Create alert rules.
- Integrate ServiceNow, Teams, and Outlook.
- Configure alert grouping and escalation.

### Phase 6 — Testing and Acceptance

- Execute controlled failure scenarios.
- Measure detection and ticket-creation times.
- Validate data quality, ingestion latency, cost, and operator usability.
- Document gaps and production rollout recommendations.

---

## 11. Pilot Success Criteria

The pilot will be considered successful when:

- All representative servers are visible through Azure Arc.
- AMA successfully collects baseline telemetry.
- Critical application dependencies are represented in monitoring tests.
- API health checks identify controlled failures.
- SQL performance and operational telemetry is queryable.
- Disk-capacity and growth conditions generate correct alerts.
- Certificate and identity risks are visible without collecting secrets.
- Custom records arrive in the expected Log Analytics tables.
- Dashboards support first-line and second-line troubleshooting.
- ServiceNow, Teams, and Outlook workflows operate correctly.
- Alert deduplication and auto-resolution are demonstrated.
- Collector and agent failures are themselves detected.
- Data ingestion volume and cost are measured.
- Security, privacy, and RBAC reviews are completed.
- A production rollout plan is approved or documented with remaining gaps.

---

## 12. Key Risks and Mitigations

| Risk | Mitigation |
|---|---|
| Third-party ERP exposes insufficient telemetry | Use vendor-supported logs, health endpoints, and carefully scoped synthetic tests. |
| Internal REST endpoints cannot be tested from Azure | Run probes from an on-premises monitoring host or Hybrid Worker. |
| Custom collectors become difficult to maintain | Keep collectors small, version-controlled, signed, and independently deployable. |
| Excessive Log Analytics ingestion cost | Use targeted DCRs, sampling, filtering, retention controls, and cost measurement. |
| Sensitive data enters logs | Perform data classification and redact at source or through DCR transformation. |
| Alert storm during shared dependency failure | Apply alert grouping, suppression, dependency correlation, and incident deduplication. |
| Monitoring silently stops | Implement Arc, AMA, collector-health, and missing-data alerts. |
| SQL telemetry creates security exposure | Use least-privilege SQL access and avoid unrestricted query-text collection. |
| Azure feature availability differs by region or licensing | Validate all assumptions during Phase 1 before production design. |

---

## 13. Executive Recommendation

Proceed with the pilot using Azure Arc and Azure Monitor as the primary hybrid monitoring platform, but treat the project as an **end-to-end observability and IT operations integration pilot**, not simply an agent deployment.

The highest-value pilot outcomes are:

1. Demonstrating cross-tier troubleshooting from ERP API through middleware to SQL.
2. Proving that on-premises private endpoints can be monitored from the correct network location.
3. Establishing secure and maintainable custom telemetry collection.
4. Validating ServiceNow and collaboration workflows.
5. Measuring operational value, data quality, and cost before scaling.

The pilot should conclude with a formal **production-readiness decision** identifying which capabilities can use native Azure Monitor features, which require custom collectors, and which require supplemental tools or vendor participation.
