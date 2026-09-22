## My Understanding of Your Intended Deliverable

You want an **implementation-grade technical tutorial and roadmap** for system integrators to build, deploy, operate, and later productionize the custom PowerShell telemetry-collector platform used in this hybrid Azure Monitor pilot.

The tutorial should focus on a complete, working reference implementation using:

> **Windows Certificate Expiration Monitoring and Tracking**

as the example workload, including certificate inventory, ingestion into Azure Monitor, custom Log Analytics tables, KQL queries, alert rules, and dashboard visualization.

## Expected Technical Scope

You want the implementation roadmap to cover the full lifecycle:

### 1. PowerShell collector project structure

Define a maintainable project layout containing:

- Shared Azure Monitor ingestion module.
- Certificate discovery and inventory collector.
- Configuration files.
- Environment-specific settings.
- Logging and error-handling components.
- Authentication components.
- Packaging and version metadata.
- Unit/integration test structure.
- Deployment and rollback scripts.
- KQL queries.
- DCR/table definitions.
- Documentation and operational runbooks.

### 2. Windows deployment model

Explain how integrators deploy the collector package to Windows servers, including:

- Target directory structure.
- Required PowerShell and module versions.
- Service account or managed identity model.
- File-system permissions.
- Script signing.
- Secret/certificate handling.
- Configuration deployment.
- Scheduled Task or Azure Automation Hybrid Worker execution.
- Upgrade and rollback procedures.
- Local logging and troubleshooting.

### 3. Data-collection job scheduling

Define how jobs are scheduled and operated:

- Certificate inventory frequency.
- Task Scheduler or Hybrid Worker configuration.
- Run-as identity.
- Concurrency control.
- Timeout handling.
- Retry behavior.
- Failure detection.
- Collector-health telemetry.
- Maintenance windows.
- Execution history and audit requirements.

### 4. Azure Monitor Logs Ingestion API integration layer

Show how the collector communicates with Azure Monitor, including:

- Microsoft Entra authentication.
- Managed identity or service-principal options.
- DCE endpoint handling.
- DCR immutable ID.
- Custom stream name.
- JSON payload construction.
- Typed schema handling.
- Batching and payload limits.
- Retry and backoff.
- HTTP error handling.
- API throttling.
- Data validation.
- Correlation and execution IDs.
- Secure configuration.
- Network/proxy requirements.

You are referring to the “Logs Analytical APIs” as the **Azure Monitor Logs Ingestion API integration layer**.

### 5. Custom Log Analytics table implementation

Explain how integrators create and manage:

- Custom Log Analytics tables.
- Table schemas and data types.
- DCR stream declarations.
- DCR data flows.
- DCR transformations.
- Workspace destinations.
- DCE association, where required.
- DCR/table deployment through portal, CLI, ARM/Bicep, or REST.
- Schema versioning.
- Infrastructure-as-code promotion from pilot to production.
- Table retention and cost considerations.

### 6. Certificate-expiration monitoring reference implementation

The tutorial should implement a complete certificate monitoring use case:

```text
Windows certificate stores and IIS bindings
        ↓
PowerShell certificate collector
        ↓
Structured JSON records
        ↓
Azure Monitor Logs Ingestion API
        ↓
DCE / DCR custom stream
        ↓
Log Analytics custom table
        ↓
KQL queries
        ↓
Azure Monitor scheduled query alerts
        ↓
Action Groups
        ├── ServiceNow
        ├── Teams
        └── Outlook/email
        ↓
Azure Workbook/dashboard
```

The certificate example should cover:

- Certificate-store discovery.
- IIS certificate-binding correlation.
- Subject, SAN, issuer, thumbprint, and validity dates.
- Days-until-expiry calculation.
- Private-key presence without exporting private keys.
- Duplicate certificate handling.
- Expired and near-expiry detection.
- Collection errors and inaccessible stores.
- Alert severity tiers.
- KQL for current inventory.
- KQL for certificates expiring in 60/30/14/7 days.
- KQL for expired certificates.
- KQL for missing collector data.
- Dashboard design.
- Alert-to-ServiceNow/Teams/Outlook integration.

### 7. Production-readiness details

You also want the additional knowledge system integrators need to fulfill the pilot and prepare for production, including:

- Security and least privilege.
- RBAC and identity separation.
- Data classification and sensitive-data controls.
- Code signing and source control.
- Environment promotion.
- Configuration management.
- Monitoring the monitoring platform.
- Cost and ingestion-volume control.
- Testing and fault injection.
- Backup and recovery.
- Operational ownership.
- Support runbooks.
- Version compatibility.
- Azure region and feature validation.
- Change management.
- Deployment automation.
- Acceptance criteria.
- Production rollout gates.

## Expected Depth and Audience

The audience is not beginner PowerShell users. The material should be written for:

- System integrators.
- System solution architects.
- System analysts.
- Azure Monitor administrators.
- Windows infrastructure engineers.
- Security and operations teams.

Therefore, the final tutorial should explain not only **what commands to run**, but also:

- Why each component exists.
- How the components interact.
- Which responsibilities belong to PowerShell, DCE, DCR, Log Analytics, Azure Monitor alerts, and the dashboard.
- How to secure and operate the solution.
- What should be validated before production rollout.

I will proceed with this interpretation once you confirm it.

---
---


## Updated Understanding of Your Intended Deliverable

You want an **implementation-grade technical tutorial and roadmap** for system integrators to build, deploy, operate, and productionize a custom PowerShell telemetry-collector platform for the hybrid Azure Monitor pilot.

The tutorial will use **two complete reference use cases**:

1. **Windows certificate expiration monitoring and tracking**
2. **Disk-space monitoring and tracking**

The design will use:

> **On-premises Windows servers → PowerShell collectors → Azure Monitor Logs Ingestion API → DCE/DCR → Log Analytics custom tables → KQL → Azure Monitor alerts → Workbooks/dashboards → ServiceNow, Teams, and Outlook**

The servers remain on-premises and are not migrated to Azure VMs.

---

# Expected Technical Scope

## 1. PowerShell Collector Project Setup

Define a maintainable project structure containing:

- Shared Azure Monitor ingestion module.
- Certificate discovery and inventory collector.
- Disk-space and folder-growth collector.
- Configuration files.
- Environment-specific settings.
- Authentication components.
- Logging and error-handling components.
- Payload and schema models.
- Version metadata.
- Unit and integration test structure.
- Deployment and rollback scripts.
- KQL queries.
- DCR and custom-table definitions.
- Operational documentation and runbooks.

The project should support multiple collectors while avoiding duplication of common functionality such as:

- Azure authentication.
- JSON serialization.
- Batch construction.
- HTTP submission.
- Retry and backoff.
- Local logging.
- Collector-health reporting.
- Configuration loading.
- Error normalization.

---

## 2. Windows Deployment Model

Explain how integrators deploy the collector package to on-premises Windows servers, including:

- Target directory structure.
- Required PowerShell and module versions.
- Windows Server compatibility.
- Service account or managed identity execution model.
- File-system permissions.
- Script-signing requirements.
- Secret and certificate handling.
- Configuration deployment.
- Scheduled Task or Azure Automation Hybrid Worker execution.
- Proxy and outbound network requirements.
- Upgrade and rollback procedures.
- Local log files and troubleshooting.
- Collector-health reporting.

The deployment design should distinguish between:

- **Local collectors**, such as certificate and disk collectors running on each monitored server.
- **Central collectors**, such as directory-wide service-account or Microsoft Entra/LAPS metadata collectors.
- **Optional management collectors**, running from a designated monitoring utility server.

---

## 3. Data-Collection Job Scheduling

Define scheduling and operational behavior for both use cases.

### Certificate collector

- Run once or twice daily.
- Scan approved certificate stores.
- Correlate certificates with IIS bindings where applicable.
- Record expiry and inventory metadata.
- Detect expired and near-expiry certificates.
- Report collection failures and inaccessible stores.

### Disk collector

- Run at a more frequent interval, such as every 5–15 minutes for critical volumes.
- Monitor logical-drive free space.
- Capture absolute free space and percentage free.
- Measure disk performance where required.
- Calculate ERP, IIS, middleware, and backup-folder size.
- Track folder growth over time.
- Identify files older than the approved retention period.
- Detect scan failures and access-denied locations.

The scheduling design should address:

- Task Scheduler or Hybrid Worker configuration.
- Run-as identity.
- Job frequency.
- Concurrency control.
- Timeouts.
- Retry behavior.
- Overlapping execution prevention.
- Maintenance windows.
- Execution history.
- Collector-health telemetry.
- Server resource impact.

---

## 4. Azure Monitor Logs Ingestion API Integration Layer

Show how both collectors submit structured telemetry to Azure Monitor, including:

- Microsoft Entra authentication.
- Managed identity and service-principal options.
- DCE endpoint handling.
- DCR immutable ID.
- Custom stream names.
- JSON payload construction.
- Strongly typed schema handling.
- Batching.
- Request-size and record-count limits.
- Retry and exponential backoff.
- HTTP error handling.
- API throttling.
- Network and proxy handling.
- Correlation and execution IDs.
- Secure configuration.
- Collector versioning.
- Ingestion success and failure reporting.

The shared ingestion layer should support multiple custom streams, for example:

```text
Custom-ServerCertificateInventoryRaw
Custom-DiskMetricsRaw
Custom-FolderMetricsRaw
Custom-CollectorHealthRaw
```

The tutorial should clarify that:

- The PowerShell collector gathers data.
- The Logs Ingestion API transports data.
- The DCR validates, transforms, and routes data.
- The Log Analytics Workspace stores and indexes data.
- KQL queries and alerts consume the stored data.

---

## 5. Custom Log Analytics Table Implementation

Explain how integrators create, deploy, and maintain custom tables for both use cases.

### Certificate tables

Potential table:

```text
ServerCertificateInventory_CL
```

Potential fields:

- `TimeGenerated`
- `Computer`
- `Environment`
- `Application`
- `StoreLocation`
- `StoreName`
- `Subject`
- `DnsNames`
- `Issuer`
- `Thumbprint`
- `SerialNumber`
- `NotBefore`
- `NotAfter`
- `DaysUntilExpiry`
- `HasPrivateKey`
- `EnhancedKeyUsage`
- `IisBindings`
- `CollectionStatus`
- `CollectorVersion`

### Disk and folder tables

Potential tables:

```text
WindowsDiskMetrics_CL
ErpFolderMetrics_CL
```

Potential disk fields:

- `TimeGenerated`
- `Computer`
- `Environment`
- `Drive`
- `VolumeLabel`
- `FileSystem`
- `TotalBytes`
- `FreeBytes`
- `UsedBytes`
- `FreePercent`
- `UsedPercent`
- `DiskLatency`
- `QueueLength`
- `CollectionStatus`
- `CollectorVersion`

Potential folder fields:

- `TimeGenerated`
- `Computer`
- `Application`
- `FolderPath`
- `TotalBytes`
- `TotalGB`
- `FileCount`
- `OldestFileUtc`
- `NewestFileUtc`
- `FilesOlderThanRetention`
- `AccessErrorCount`
- `ScanDurationSeconds`
- `CollectionStatus`
- `CollectorVersion`

The tutorial should cover:

- Custom table creation.
- Table schema and Azure Monitor data types.
- DCR stream declarations.
- DCR data flows.
- DCR transformations.
- Workspace destinations.
- DCE association where required.
- DCR and table deployment through Azure portal, CLI, ARM/Bicep, or REST.
- Schema versioning.
- Infrastructure-as-code promotion.
- Retention and archive policies.
- Ingestion-cost management.

---

## 6. Certificate-Expiration Monitoring Reference Implementation

The tutorial should implement this complete certificate workflow:

```text
Windows certificate stores
    ├── LocalMachine\My
    ├── LocalMachine\WebHosting
    └── Approved application stores
            │
            ▼
PowerShell certificate collector
            │
            ├── Certificate metadata
            ├── SAN/DNS names
            ├── Issuer and thumbprint
            ├── Validity dates
            ├── Private-key presence
            └── IIS binding correlation
            │
            ▼
Structured JSON records
            │
            ▼
Azure Monitor Logs Ingestion API
            │
            ▼
DCE / DCR custom stream
            │
            ▼
ServerCertificateInventory_CL
            │
            ▼
KQL queries
            │
            ├── Current certificate inventory
            ├── Expired certificates
            ├── Expiring within 60 days
            ├── Expiring within 30 days
            ├── Expiring within 14 days
            ├── Expiring within 7 days
            └── Missing collector data
            │
            ▼
Azure Monitor scheduled query alerts
            │
            ▼
Action Groups
            ├── ServiceNow
            ├── Microsoft Teams
            └── Outlook/email
            │
            ▼
Azure Workbook certificate dashboard
```

The implementation must not collect:

- Private keys.
- PFX files.
- Certificate passwords.
- Tokens.
- Secrets.

---

## 7. Disk-Space Monitoring Reference Implementation

The tutorial should implement a complete disk-monitoring workflow:

```text
Windows server
    ├── Logical-drive capacity
    ├── Free and used space
    ├── Disk latency
    ├── Queue length
    ├── ERP log directories
    ├── IIS log directories
    ├── Middleware queues or archives
    ├── SQL data/log/tempdb volumes
    └── Backup directories
            │
            ▼
PowerShell disk/folder collector
            │
            ├── Drive metrics
            ├── Folder size
            ├── File count
            ├── Growth calculation
            ├── Retention-age analysis
            └── Access/scan status
            │
            ▼
Structured JSON records
            │
            ▼
Azure Monitor Logs Ingestion API
            │
            ▼
DCE / DCR custom streams
            │
            ├── WindowsDiskMetrics_CL
            └── ErpFolderMetrics_CL
            │
            ▼
KQL queries
            │
            ├── Volumes below free-space threshold
            ├── Volumes below absolute free-space threshold
            ├── Fastest-growing folders
            ├── SQL log-volume risk
            ├── Backup-directory accumulation
            ├── Files beyond retention
            └── Failed or incomplete scans
            │
            ▼
Azure Monitor alerts
            │
            ├── Warning: capacity planning
            ├── Critical: outage risk
            ├── Rapid growth
            └── Collector/scan failure
            │
            ▼
Azure Workbook disk and capacity dashboard
```

The disk implementation should distinguish between:

- Percentage free space.
- Absolute free bytes.
- Growth rate.
- File count.
- Disk latency.
- Folder scan failure.
- Application-specific capacity risk.

It should not rely exclusively on a fixed “10% free” rule.

---

## 8. Dashboard and KQL Scope

The tutorial should provide KQL and dashboard designs for:

### Certificate dashboard

- Certificate inventory by server.
- Expiry timeline.
- Certificates expired.
- Certificates expiring within 60/30/14/7 days.
- Certificates without expected IIS bindings.
- Certificates with missing private keys where a private key is required.
- Collector data freshness.
- Certificate ownership and application association.

### Disk dashboard

- Free-space heat map by server and volume.
- Absolute free-space risk.
- Disk latency and queue length.
- Top folders by size.
- Fastest-growing folders.
- SQL data/log/tempdb capacity.
- Backup-directory usage.
- Files older than retention.
- Access-denied and incomplete scans.
- Collector data freshness.

---

## 9. Alerting and ITSM Integration

The tutorial should show how to implement alert rules for:

### Certificate alerts

- Expired certificate.
- Certificate expires within 7 days.
- Certificate expires within 14 days.
- Certificate expires within 30 days.
- Certificate inventory has not updated.
- HTTPS endpoint serves an unexpected or invalid certificate.

### Disk alerts

- Free space below warning threshold.
- Free space below critical threshold.
- Absolute free space below minimum.
- Rapid folder growth.
- SQL log or `tempdb` volume approaching exhaustion.
- Backup directory exceeds capacity threshold.
- Folder scan partially failed.
- Disk latency exceeds baseline.
- Collector data is missing.

Alert design should include:

- Severity.
- Evaluation period.
- Query frequency.
- Suppression.
- Deduplication.
- Assignment group.
- ServiceNow mapping.
- Teams channel.
- Outlook escalation.
- Auto-resolution.
- Maintenance-window behavior.
- Runbook link.

---

## 10. Production-Readiness Details

The tutorial should also cover the skills and controls required for pilot and production implementation:

### Security

- Least privilege.
- Azure RBAC.
- Collector identity separation.
- Managed identities or certificate-based service principals.
- Key Vault integration.
- Script signing.
- No password or private-key collection.
- Data classification.
- Sensitive-field filtering.

### Operations

- Collector-health monitoring.
- AMA and Arc health monitoring.
- Missing-data alerts.
- Local logs.
- Retry and backoff.
- Agent and module upgrades.
- Maintenance windows.
- Troubleshooting runbooks.
- Support ownership.

### Engineering

- Source control.
- Code review.
- Unit testing.
- Integration testing.
- Fault injection.
- Configuration separation by environment.
- Semantic versioning.
- Schema compatibility.
- Infrastructure as code.
- Deployment and rollback automation.

### Platform governance

- Workspace region.
- Data retention.
- Ingestion cost.
- API limits.
- DCR quotas.
- DCE requirements.
- Proxy and firewall rules.
- Azure feature and regional availability.
- ServiceNow and Teams integration support.

### Production transition

- Pilot acceptance criteria.
- Performance and cost results.
- Security approval.
- Operational handover.
- Production architecture decision.
- Rollout waves.
- Disaster recovery.
- Change management.
- Vendor support model.

---

## Intended Final Deliverable

The final output should be a detailed implementation guide that allows system integrators to:

1. Build the PowerShell project.
2. Deploy it to Windows servers.
3. Configure scheduled collection jobs.
4. Authenticate and submit telemetry to Azure Monitor.
5. Configure DCE/DCR resources.
6. Create custom Log Analytics tables.
7. Implement certificate-expiration monitoring.
8. Implement disk-space and folder-growth monitoring.
9. Create KQL queries and dashboards.
10. Configure alerts and ITSM notifications.
11. Test failure and recovery scenarios.
12. Operate, upgrade, troubleshoot, and productionize the solution.

This updated interpretation adds disk-space monitoring as a **first-class custom telemetry use case**, alongside certificate monitoring, rather than treating it only as a standard AMA performance-counter scenario.
