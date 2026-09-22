## Architecture Design Overview

**Azure Monitor Agent (AMA) collects supported Windows telemetry continuously at runtime without custom PowerShell.** A **Data Collection Rule (DCR)** tells AMA what to collect, how frequently to collect it, and which **Log Analytics Workspace (LAW)** receives it.

Custom PowerShell or another collector is required only when the required information is **not exposed through an AMA-supported data source**—for example, SQL Server DMVs, a live `Get-NetTCPConnection` snapshot, certificate-store inventory, service-account expiry, or a functional ERP REST transaction.

A Log Analytics Workspace does **not connect to or pull logs from your servers**. AMA running on each server reads the configured local sources and securely **pushes telemetry outbound to Azure Monitor’s ingestion service**, which stores it in the workspace.

---

# 1. Component Responsibilities

| Component | Technical responsibility |
|---|---|
| **Azure Arc Connected Machine agent** | Registers the on-premises server as an Azure resource and provides the management channel used to deploy extensions such as AMA. It is not the primary monitoring-data collector. |
| **Azure Monitor Agent (AMA)** | Runs locally on the Windows server, reads supported telemetry sources, buffers data, and sends it to Azure Monitor. |
| **Data Collection Rule (DCR)** | Configuration describing data sources, sampling intervals, filtering, transformations, streams, and destinations. |
| **DCR association** | Links a DCR to one or more Arc-enabled servers. Without the association, AMA does not know that the rule applies to that server. |
| **Data Collection Endpoint (DCE)** | An Azure endpoint used for configuration access and/or data ingestion in scenarios that require it, particularly custom ingestion, private connectivity, or specific network architectures. It is not always required for every standard AMA deployment. |
| **Log Analytics Workspace (LAW)** | Azure Monitor Logs storage and query boundary. It receives processed records through the Azure Monitor ingestion pipeline. |
| **Logs Ingestion API** | HTTPS API used by custom scripts or applications to submit structured records through a DCR into a workspace table. |
| **Azure Workbook** | Dashboard layer that queries the workspace using KQL. |
| **Azure Monitor alert rule** | Periodically evaluates metrics or KQL queries and invokes an Action Group when conditions are met. |

---

# 2. How AMA Collects Data During Runtime

## 2.1 End-to-end collection flow

The standard runtime flow is:

1. The on-premises Windows server is onboarded to **Azure Arc**.
2. The **Azure Monitor Agent extension** is installed on that Arc-enabled server.
3. A **DCR is associated** with the server.
4. AMA retrieves the applicable DCR configuration from Azure.
5. AMA subscribes to or samples the configured local data sources.
6. AMA converts the data into the appropriate Azure Monitor stream.
7. The data may be filtered or transformed according to the DCR.
8. AMA buffers the data locally if necessary.
9. AMA sends the data outbound over encrypted HTTPS to Azure Monitor.
10. Azure Monitor routes the data to the Log Analytics Workspace and destination table identified in the DCR.
11. Workbooks, KQL queries, and alerts consume the records from the workspace.

Conceptually:

```text
On-premises Windows server
  ├─ Windows Event Logs
  ├─ Performance Counters
  ├─ IIS logs
  ├─ Supported text/application logs
  └─ AMA local runtime
          │
          │ Reads only DCR-selected sources
          │ Buffers and batches records
          │ Outbound HTTPS/TLS
          ▼
Azure Monitor ingestion service
          │
          │ Applies DCR stream routing/transformation
          ▼
Log Analytics Workspace tables
          │
          ├─ KQL queries
          ├─ Azure Workbooks
          ├─ Alert rules
          └─ ServiceNow / Teams / email actions
```

The collection is generally **near-real-time**, not hard real-time. Depending on the source, sampling interval, buffering, network conditions, ingestion load, and query availability, data commonly appears after a short delay.

---

## 2.2 How different runtime sources are collected

### Windows performance counters

AMA periodically samples counters according to the interval configured in the DCR.

Examples:

- CPU utilization
- Available memory
- Paging
- Logical disk free space
- Disk latency
- Disk queue length
- Network throughput

If the DCR specifies a 60-second sampling interval, AMA requests those values from Windows every 60 seconds and sends the resulting samples to Azure Monitor.

No custom PowerShell is required.

### Windows Event Logs

AMA creates subscriptions against selected Windows Event Log channels. The DCR uses XPath queries to select relevant channels and event levels.

Examples:

- `System`
- `Application`
- Selected `Security` events
- IIS-related event channels
- SQL Server events written into the Application log
- Vendor ERP event channels, if the application registers one

AMA maintains collection state/bookmarks so it can identify newly generated events rather than repeatedly uploading the full event log.

No custom PowerShell is required if the ERP or web application writes useful events to a supported Windows Event Log channel.

### IIS logs

If the web server uses standard IIS W3C logging, AMA can collect supported IIS logs into the appropriate Azure Monitor table through a DCR.

Useful fields can include:

- HTTP method
- URI
- Response status
- Substatus
- Response time
- Client IP
- Server IP
- User agent
- Bytes sent/received

No custom PowerShell is normally required for standard IIS logging.

### Application text logs

If the ERP or middleware application writes structured or consistently formatted text files, AMA can collect supported custom text logs.

The design must define:

- Exact file path and wildcard.
- File rotation behavior.
- Encoding.
- Record boundaries.
- Timestamp format.
- Destination custom table.
- Transformation and parsing logic.
- Handling of multiline exceptions.
- Sensitive-data filtering.

For example, an application writing to `D:\ERP\Logs\erp-20260921.log` may be collectible without PowerShell if the log format is compatible with AMA custom text-log collection.

### Agent buffering

AMA provides local buffering to tolerate temporary interruptions. If Azure connectivity is unavailable, the agent can retain data within its supported buffering limits and retry later.

This is not an unlimited offline archive. Extended outages or very high event volume can cause data loss after buffer limits are reached. Therefore, the pilot must test:

- Azure endpoint outage.
- Proxy interruption.
- Server restart.
- High log volume.
- Recovery and data-delivery behavior.

---

# 3. When Custom PowerShell Is and Is Not Required

## 3.1 No custom script required

| Monitoring requirement | Native AMA collection method |
|---|---|
| CPU and memory | Windows performance counters |
| Disk free space | `LogicalDisk` performance counters |
| Disk latency and queue length | Windows performance counters |
| Windows operating-system errors | Windows Event Logs |
| Windows service start/stop events | Service Control Manager events in the System log |
| IIS request logs | IIS log data source |
| ERP application events written to Windows Event Log | Windows Event Log data source |
| Compatible ERP text logs | Custom text-log data source |
| SQL errors written to Windows Application log | Windows Event Log data source |
| Network interface throughput | Windows performance counters |

## 3.2 Custom collector required or strongly recommended

| Monitoring requirement | Why AMA alone is insufficient |
|---|---|
| Current Windows service state | Event 7036 shows transitions, but it does not always prove the current state. A periodic service-state collector may be required. |
| Active TCP connection inventory | AMA does not periodically execute `Get-NetTCPConnection`. |
| Process-to-port mapping | Requires querying Windows networking and process state. |
| REST API functional test | Requires an HTTP client to call the endpoint and validate status, latency, TLS, authentication, and response content. |
| SQL Server DMVs | AMA does not execute T-SQL DMV queries. |
| SQL blocking chains | Requires querying SQL runtime state. |
| Certificate-store inventory | Requires reading the Windows certificate store, IIS bindings, or application configuration. |
| Service-account password-expiry status | Requires querying Active Directory or Entra ID metadata. |
| ERP log-directory size | Requires measuring directory contents unless the application emits its own metric. |
| SQL database backup status | Requires SQL queries or SQL Agent/job telemetry. |
| Application-specific health | Requires vendor APIs, vendor counters, log parsing, or a synthetic transaction. |

## 3.3 Important boundary

A DCR **cannot run a PowerShell script**.

A DCR can:

- Define sources AMA knows how to read.
- Filter data.
- Apply supported ingestion-time transformations.
- Route streams.
- Select destination workspaces and tables.

A DCR cannot:

- Run `Get-NetTCPConnection`.
- Execute a REST call.
- Query SQL Server.
- Enumerate certificates.
- Query Active Directory.
- Measure folder sizes.

Those operations need an execution mechanism such as:

- Windows Task Scheduler.
- Azure Automation Hybrid Runbook Worker.
- SQL Server Agent.
- A managed Windows service.
- A vendor-supported monitoring agent.
- A custom .NET/PowerShell collector.

---

# 4. Two Patterns for Custom Data Collection

## Pattern A — Collector writes to a file; AMA tails the file

```text
PowerShell / application collector
        ↓
Writes records to a local text file
        ↓
AMA custom text-log data source
        ↓
DCR
        ↓
Log Analytics custom table
```

### Advantages

- Simple agent-based model.
- AMA handles transport, buffering, and retries.
- The custom collector does not directly authenticate to Azure Monitor ingestion.

### Disadvantages

- File rotation and cleanup must be managed.
- Structured fields may require ingestion-time parsing.
- Multiline and inconsistent formats are harder to process.
- Collection is less direct than structured API ingestion.

This pattern is reasonable for application logs already written to disk.

---

## Pattern B — Collector posts structured JSON to the Logs Ingestion API

```text
PowerShell / SQL Agent / collector service
        ↓
Queries Windows, REST endpoint, SQL DMV, or certificate store
        ↓
Builds structured JSON records
        ↓
Authenticates using Entra ID
        ↓
Logs Ingestion API
        ↓
DCR transformation and stream mapping
        ↓
Log Analytics custom table
```

### Advantages

- Strongly typed, structured records.
- Cleaner custom-table schema.
- No intermediate log file.
- Better for SQL DMVs, certificates, REST tests, and connection snapshots.

### Disadvantages

- The collector must authenticate securely.
- Retry, batching, and error handling must be implemented.
- DCR/table schemas must be managed.
- Authentication material or managed identity must be designed carefully.

For the pilot, this is the preferred pattern for:

- `SQLSessionActivity_CL`
- `SQLBlocking_CL`
- `SQLWaitStatsDelta_CL`
- `NetworkConnectionState_CL`
- `RestEndpointHealth_CL`
- `ServerCertificateInventory_CL`
- `ServiceAccountStatus_CL`

---

# 5. Technical Role of the Data Collection Rule

A DCR contains several logical sections.

## 5.1 Data sources

These identify what AMA should collect, such as:

- Windows Event Logs.
- Windows performance counters.
- IIS logs.
- Custom text logs.
- Other supported extension-based streams.

## 5.2 Streams

A stream identifies the type and schema of the incoming data.

Examples conceptually include:

- Windows events.
- Performance data.
- IIS data.
- A custom SQL session stream.
- A custom certificate inventory stream.

## 5.3 Data flows


A data flow maps:

- One or more input streams.
- An optional transformation.
- A destination.
- An output stream/table.

## 5.4 Destinations

For this pilot, the principal destination is the **Log Analytics Workspace**.

A DCR can therefore be summarized as:

> Collect these specific data sources from these associated machines, optionally transform them, and route them to this workspace and table.

---

# 6. What the DCE Does

A **Data Collection Endpoint** should not be confused with a DCR.

- **DCR:** collection and routing instructions.
- **DCE:** network endpoint used for configuration access and/or ingestion in applicable architectures.

Depending on the Azure Monitor feature and current endpoint model, standard AMA collection can use Azure-managed endpoints without requiring a dedicated DCE. A DCE is generally relevant when:

- Using the Logs Ingestion API under an endpoint model that requires it.
- Using Azure Monitor Private Link.
- Controlling network access paths.
- Separating collection/configuration endpoints.
- A specific DCR, region, or collection scenario requires one.

The implementation team should not create DCEs automatically for every DCR. It should first determine:

1. Whether standard public Azure Monitor ingestion is allowed.
2. Whether Private Link is required.
3. Whether the DCR exposes or requires its own ingestion endpoint.
4. Whether the custom Logs Ingestion API design requires a DCE.
5. Whether the DCE and destination workspace satisfy the applicable regional requirements.

---

# 7. How AMA Works with the Log Analytics Workspace

## 7.1 The workspace does not access the server

The Log Analytics Workspace does not:

- Sign in to Windows.
- Open SMB shares.
- Read server log files remotely.
- Query Windows Management Instrumentation remotely.
- Query SQL Server directly.
- Initiate inbound monitoring connections to the server.

Instead, AMA performs local collection and pushes records outbound.

This is important for firewall and security reviews:

- Normally no inbound Azure-to-server monitoring connection is required.
- The server must have outbound HTTPS connectivity to the required Azure Arc and Azure Monitor endpoints.
- Proxy support, TLS inspection, firewall allowlists, and DNS resolution must be validated.

## 7.2 AMA authentication

For an Arc-enabled server, AMA is deployed and managed as an Azure extension. The Arc-enabled machine has an Azure identity and resource context used to authenticate to Azure services.

AMA does not normally require IT staff to insert a Log Analytics workspace shared key into every server. The agent and Azure resource configuration establish the authorized collection path.

Custom applications using the Logs Ingestion API use a separate authentication flow, generally:

1. Obtain a Microsoft Entra ID access token.
2. Present the token to the ingestion endpoint.
3. Reference the appropriate DCR.
4. Submit records for a stream defined by that DCR.
5. Azure verifies the caller's authorization to submit data through the DCR.

This separates standard AMA authentication from custom-collector authentication.

## 7.3 Data destination

The DCR destination references the target Log Analytics Workspace. Azure Monitor uses that resource mapping to route accepted records into the workspace.

The destination table depends on the stream:

| Data type | Example destination |
|---|---|
| Windows events | `Event` |
| Performance counters | `Perf` or another configured monitoring table |
| VM Insights metrics | Commonly `InsightsMetrics` and related VM Insights tables |
| IIS logs | `W3CIISLog` |
| Heartbeat/agent presence | `Heartbeat` |
| Custom SQL sessions | `SQLSessionActivity_CL` |
| Custom certificate inventory | `ServerCertificateInventory_CL` |
| Custom TCP connections | `NetworkConnectionState_CL` |

Exact tables depend on the enabled Azure Monitor solution and DCR configuration.

---

# 8. How Data Becomes Available in Log Analytics

Once Azure Monitor accepts a record:

1. The record is validated against the stream schema.
2. Any configured DCR transformation is applied.
3. The output is routed to the correct table.
4. Azure Monitor assigns or validates `TimeGenerated`.
5. The record is indexed and made queryable.
6. Workspace retention and archive policies apply.
7. Users, workbooks, and alert rules access it through KQL.

A representative query path is:

```text
AMA → Azure Monitor ingestion → LAW table → KQL → Workbook/alert
```

For example:

- AMA samples free disk space.
- The DCR routes the sample into the configured workspace.
- The record appears in the applicable performance table.
- A workbook displays free space by computer and drive.
- An alert rule evaluates the table every five minutes.
- If free space is below the critical threshold, the Action Group invokes ServiceNow, Teams, and email.

---

# 9. Access Control: Who Can Read the Workspace Data?

Collection authorization and query authorization are separate.

## 9.1 Data ingestion access

Controls which agents or applications may submit records.

- AMA uses its managed Azure configuration and machine identity.
- Custom collectors use Entra identities and DCR-related permissions.
- The collector should receive write-only telemetry permissions where possible.

## 9.2 Data query access

Controls which people and services can read the collected logs.

Use Azure RBAC roles such as:

- Monitoring Reader.
- Log Analytics Reader.
- More restricted custom roles where needed.
- Contributor roles only for administrators who create DCRs, alert rules, and workbooks.

A collector that writes data should not automatically have permission to query all workspace data.

## 9.3 Sensitive-data implications

Because the workspace centralizes data, the pilot must review whether logs contain:

- Usernames.
- Client IP addresses.
- SQL login names.
- ERP transaction identifiers.
- Customer information.
- Authentication tokens.
- API query strings.
- SQL statement parameters.
- Personally identifiable or financial information.

Use DCR transformations and source-level configuration to exclude or redact sensitive fields before ingestion wherever possible.

---

# 10. Recommended Collection Design for the ERP Pilot

## 10.1 Native AMA collection

Implement without custom scripting:

| Server role | Native collection |
|---|---|
| All Windows servers | CPU, memory, disk capacity, disk latency, network throughput, heartbeat, selected Windows events. |
| ERP server | ERP Windows Event Log channels and compatible application text logs. |
| Web/middleware server | IIS logs, application events, compatible middleware logs. |
| SQL server | Windows performance counters, SQL/Windows service events, selected SQL events written to the Application log. |

## 10.2 Custom collection

Implement only for the identified functional gaps:

| Collector | Execution mechanism | Output |
|---|---|---|
| REST API health check | Scheduled PowerShell/.NET process or Hybrid Worker | HTTP status, latency, TLS result, response validation |
| TCP connection snapshot | On-demand or scheduled PowerShell | Local/remote endpoint, state, PID, process |
| SQL DMV collector | SQL Agent, secure service, or Hybrid Worker | Sessions, blocking, waits, file I/O, database files |
| Certificate inventory | Daily PowerShell collector | Subject, SAN, thumbprint, issuer, expiry, binding |
| Service-account status | Scheduled AD query | Enabled, locked, password expiry, gMSA status |
| Log-folder growth | Scheduled PowerShell collector | Folder size and growth |
| Explicit service-state check | Scheduled PowerShell or monitoring extension | Current service state rather than only state-change events |

---

# 11. Recommended DCR Separation

Do not place every collection source into one oversized DCR. Use logical separation:

1. **Windows-Baseline-DCR**
   - CPU, memory, disk, network.
   - Core Windows event logs.
2. **ERP-Application-DCR**
   - ERP-specific event channels and application logs.
3. **Web-IIS-DCR**
   - IIS and middleware logs.
4. **SQL-Windows-DCR**
   - SQL host performance counters and relevant Windows events.
5. **Custom-Ingestion-DCR**
   - SQL DMVs, REST checks, TCP snapshots, certificates, and account status.

This improves:

- Change management.
- Troubleshooting.
- Server-role targeting.
- Cost control.
- Security review.
- Rollback capability.

---

# 12. Operational Failure Scenarios to Test

The pilot should intentionally validate these conditions:

1. Stop AMA and confirm monitoring detects missing heartbeat.
2. Block outbound Azure connectivity and verify buffering/recovery behavior.
3. Stop an ERP Windows service.
4. Stop an IIS application pool.
5. Fill a test volume past warning and critical thresholds.
6. Block ERP-to-SQL TCP connectivity.
7. Return an HTTP 500 from a test API endpoint.
8. Introduce a SQL blocking session.
9. Cause a SQL Agent test job to fail.
10. Load a short-lived test certificate and validate expiry alerts.
11. Disable a test service account and confirm authentication-event detection.
12. Submit malformed custom JSON and verify ingestion failure diagnostics.
13. Change the custom table schema and validate deployment/change controls.

---

## Final architecture assessment

For the pilot, approximately **60–75% of the required telemetry can likely be collected using AMA and standard DCR data sources without custom scripts**. The exact percentage depends on how well the third-party ERP and middleware expose Windows events, IIS logs, performance counters, and structured text logs.

Custom collectors remain necessary for the deeper system-integration requirements:

- Live process-to-port connections.
- Internal REST functional checks.
- SQL DMV data.
- Certificate inventory.
- Service-account lifecycle status.
- Directory growth and explicit current service state.

The key design principle is:

> Use AMA for supported operating-system and application log sources; use small, controlled custom collectors only for telemetry that AMA cannot obtain, and route both forms of data into the same Log Analytics Workspace for correlation, visualization, and alerting.
