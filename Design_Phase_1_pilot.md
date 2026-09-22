# Pilot Design: Network/Port Monitoring + SQL DMV Collection + Credential/Certificate Governance

Given your architect/integrator audience, I'll go deep on mechanism, not just concept — including where Azure's native tooling stops and where you need custom collection logic.

---

## PART A — Network/Port & REST API Runtime Monitoring Pilot

### A.1 Problem Restatement (Architect-Level)
You have a **3-tier dependency chain per ERP instance**:

```
ERP App Server (hosts REST APIs, e.g., IIS/Kestrel on ports 443/8080/custom)
      │  outbound calls to
      ▼
Middleware/Web Server (reverse proxy, integration layer, ports 443/80/custom)
      │  outbound calls to
      ▼
SQL Server (TCP 1433 default, or named instance dynamic ports via SQL Browser 1434)
```

The requirement is **runtime, ad-hoc, on-demand troubleshooting** ("anytime") — not just periodic health checks. This means the pilot needs **two complementary collection patterns**, not one:

1. **Continuous synthetic monitoring** (baseline trend + alerting)
2. **On-demand diagnostic capability** (IT support triggers deep inspection during an incident)

### A.2 Component Selection

| Capability Needed | Tool | Why |
|---|---|---|
| Continuous TCP reachability/latency between tiers | **Azure Network Watcher — Connection Monitor** (hybrid mode, agent-based) | Native Azure product, supports on-prem endpoints via installed monitoring agent, gives loss %, latency, hop-by-hop path analysis, historical trend in Log Analytics |
| REST API endpoint health (HTTP status, response time, payload validation) | **Application Insights — Availability Tests (URL Ping / Multi-step Web Test)** or **standalone synthetic test agent** | Purpose-built for HTTP/REST endpoint monitoring, not just TCP; supports custom headers/auth tokens for ERP API auth |
| Live/on-demand connection state (what's connected right now, to what, on what port) | **Custom DCR + PowerShell script (netstat/Get-NetTCPConnection) → Custom Table in LAW**, triggered on-demand via **Azure Automation Runbook / Hybrid Worker** | No native Azure product gives point-in-time "show me active connections now" — must build this |
| Correlation across tiers during an incident | **KQL cross-table query in LAW** joining Connection Monitor data + Availability Test data + Custom connection-state table + Windows Event Log | This is where "end-to-end troubleshooting on the fly" actually happens |

### A.3 Detailed Design — Continuous Layer (Connection Monitor)

**Topology:**
- Deploy **Network Watcher extension / monitoring agent** on: 1 ERP app server, 1 middleware server, 1 SQL server (pilot scope)
- Define **Test Groups** in Connection Monitor:
  - Test Group 1: ERP App Server → Middleware Server, port = actual REST API port (e.g., 443)
  - Test Group 2: Middleware Server → SQL Server, port = 1433 (or dynamic port if named instance — resolve via SQL Browser first)
  - Test Group 3: ERP App Server → SQL Server (if ERP connects directly to DB, bypassing middleware — common in ERP architectures)
- Each test group produces: **checks failed %, round-trip time (RTT), hop count/path**
- Output lands automatically in the Log Analytics Workspace under `NWConnectionMonitorTestResult` table

**Alerting threshold example (architect-level KQL for alert rule):**
```kql
NWConnectionMonitorTestResult
| where TestGroupName_s == "ERP-to-Middleware"
| where TimeGenerated > ago(5m)
| summarize FailPct = avg(todouble(ChecksFailedPercent_s)) by SourceAddress_s, DestinationAddress_s
| where FailPct > 20
```

### A.4 Detailed Design — REST API Layer (Application Insights Availability Tests)

Since ERP hosts REST APIs, TCP reachability alone isn't sufficient — you need **application-layer health** (is the API actually returning valid responses, not just "is the port open").

- Create an **Application Insights resource** (can be workspace-based, pointing to the same LAW — keeps everything unified)
- Configure **Multi-step Web Test** (via Visual Studio web test recorder, or **Standard/Custom TrackAvailability() SDK calls** if you want the ERP itself to self-report):
  - Step 1: Call ERP REST API health endpoint (or a representative business endpoint) with auth token
  - Step 2: Validate response code (200) + optional payload content match (e.g., JSON contains `"status":"ok"`)
  - Step 3: Measure response time
- Run from **on-prem test locations** using **Private Availability Testing** (deploy Availability Test agent on an on-prem monitoring VM/hybrid worker) — this is important because public multi-region Azure test points won't reflect your internal network path; you need the test to originate from inside your network, same as real traffic

This gives you **application-layer synthetic transactions**, which is the correct fit for "REST APIs... monitored during runtime."

### A.5 Detailed Design — On-Demand Diagnostic Layer (the "anytime troubleshooting" piece)

This is the part with no out-of-box Azure product — you build a lightweight custom collector:

**Mechanism: Custom Data Collection Rule (DCR) + Data Collection Endpoint (DCE) + Logs Ingestion API**

1. Create a **Data Collection Endpoint (DCE)** in Azure — this is the ingestion HTTPS endpoint on-prem agents post to
2. Create a **custom table** in LAW, e.g., `ERP_NetworkConnectionState_CL`, with schema:
   - `TimeGenerated`, `ComputerName`, `LocalPort`, `RemoteAddress`, `RemotePort`, `State` (Established/TimeWait/CloseWait), `ProcessName`, `PID`
3. Create a **DCR** mapping a PowerShell-collected payload to this custom table via **Logs Ingestion API**
4. Deploy a script (scheduled via AMA custom DCR data source, OR run on-demand via **Azure Automation Hybrid Runbook Worker** installed on each server) using:
```powershell
Get-NetTCPConnection | Where-Object {$_.State -eq 'Established'} |
Select-Object LocalAddress, LocalPort, RemoteAddress, RemotePort, State, OwningProcess |
# enrich with process name, then POST to DCE via Logs Ingestion API (REST call with bearer token)
```
5. IT support can **trigger this runbook on-demand** from Azure Automation (or expose via a simple internal web trigger) during an active incident — giving a **point-in-time snapshot of exactly what's connected to what, on which port, from which process**, landed directly into LAW within seconds
6. Query pattern for troubleshooting:
```kql
ERP_NetworkConnectionState_CL
| where ComputerName == "ERP-APP-01"
| where RemotePort == 1433
| project TimeGenerated, RemoteAddress, State, ProcessName
```

This satisfies your "monitor/track network connections during runtime and troubleshoot on the fly anytime" requirement — continuous baseline (Connection Monitor + Availability Tests) plus on-demand deep-dive (custom DCR snapshot).

---

## PART B — SQL Server DMV Collection via Data Collection Rules

### B.1 Key Architectural Fact
**Azure Monitor Agent (AMA) does not natively query SQL Server DMVs.** AMA's built-in data sources are Windows Perf Counters, Windows Event Logs, and Syslog. SQL DMV data requires one of these integration patterns — pick based on your team's operational preference:

| Option | Mechanism | Fit |
|---|---|---|
| **Option 1 (Recommended for pilot): SQL custom DCR via Logs Ingestion API** | Scheduled SQL Agent Job or PowerShell script runs T-SQL against DMVs → formats as JSON → posts to DCE/DCR → lands in custom LAW table | Full control over exact DMVs queried, works for any SQL version, no extra licensing |
| **Option 2: Telegraf + Azure Monitor output plugin** | Telegraf agent installed on SQL Server, `sqlserver` input plugin queries DMVs, outputs to Azure Monitor custom logs | Good if you already use Telegraf/InfluxDB elsewhere; more moving parts |
| **Option 3: Microsoft SQL Server Management Pack data via legacy MMA (deprecated)** | Not recommended — Log Analytics MMA agent is retired in favor of AMA |
| **Option 4: SQL Insights (Preview)** | Microsoft's purpose-built SQL monitoring in Azure Monitor | Check current regional/preview availability; if GA in your region, this reduces custom-build effort significantly — validate in pilot before committing to Option 1 |

**Pilot recommendation**: Build Option 1 (full control, proven pattern, works regardless of SQL Insights preview status), and **in parallel evaluate SQL Insights** as a potential replacement/supplement.

### B.2 DMV Query Set (Core Performance Signals)

| Metric Category | DMV(s) | Key Columns to Extract |
|---|---|---|
| CPU pressure | `sys.dm_os_schedulers`, `sys.dm_exec_query_stats` | `runnable_tasks_count`, `total_worker_time` |
| Blocking/deadlocks | `sys.dm_exec_requests`, `sys.dm_tran_locks` | `blocking_session_id`, `wait_type`, `wait_time` |
| Wait statistics | `sys.dm_os_wait_stats` | `wait_type`, `wait_time_ms`, `waiting_tasks_count` |
| Connection/session load | `sys.dm_exec_sessions`, `sys.dm_exec_connections` | `session_id`, `host_name`, `program_name`, `client_net_address`, `login_name` — **this directly ties back to Part A, showing exactly which app-tier host/process is connected** |
| Buffer cache/memory pressure | `sys.dm_os_memory_clerks`, `sys.dm_os_buffer_descriptors` | `memory_used_mb`, `page counts by database` |
| I/O latency (data/log files) | `sys.dm_io_virtual_file_stats` | `io_stall_read_ms`, `io_stall_write_ms`, per-database file |
| Top expensive queries | `sys.dm_exec_query_stats` joined to `sys.dm_exec_sql_text` | `total_elapsed_time`, `execution_count`, query text |

### B.3 DCR Implementation Detail

1. **Collector script** (runs as SQL Agent Job every 1–5 min, or via Hybrid Runbook Worker):
```sql
SELECT 
    GETUTCDATE() AS TimeGenerated,
    @@SERVERNAME AS ServerInstance,
    s.session_id, s.login_name, s.host_name, s.program_name,
    c.client_net_address, r.blocking_session_id, r.wait_type, r.wait_time
FROM sys.dm_exec_sessions s
JOIN sys.dm_exec_connections c ON s.session_id = c.session_id
LEFT JOIN sys.dm_exec_requests r ON s.session_id = r.session_id
WHERE s.is_user_process = 1
```
2. Script wraps result set as **JSON array**, POSTs to the **Logs Ingestion API endpoint** (DCE URL) with the DCR's immutable ID and stream name in the request, authenticated via **Azure AD App Registration (service principal) with Monitoring Metrics Publisher role** on the DCR
3. Custom table schema in LAW, e.g., `SQL_SessionActivity_CL`, `SQL_WaitStats_CL`, `SQL_IOLatency_CL` — separate tables per DMV category keeps query performance and schema management clean
4. **Correlation query** (this is the payoff — ties Part A network data to Part B SQL data):
```kql
SQL_SessionActivity_CL
| where ServerInstance_s == "SQLPROD01"
| where client_net_address_s == "10.10.5.23"  // matches ERP app server IP from Connection Monitor
| project TimeGenerated, login_name_s, program_name_s, wait_type_s, blocking_session_id_d
```
This lets an architect/analyst answer: *"The ERP app server's REST API is timing out — is it because SQL session from that specific host is blocked, and on what wait type?"* — genuine end-to-end correlation.

### B.4 Collection Frequency & Cost Consideration
- DMV snapshots at 1–5 min intervals generate meaningful ingestion volume — for pilot, start at 5 min, tune based on LAW ingestion cost and diagnostic value
- Consider a **separate low-frequency table** for expensive query text capture (less frequent, larger payload) vs. **high-frequency table** for session/wait stats (small payload, frequent)

---

## PART C — Windows Login Password & Server Certificate Management/Tracking

### C.1 Windows Local Account Password Management

**Recommended: Windows LAPS (Local Administrator Password Solution)** — Microsoft's modern replacement for legacy LAPS, natively integrated with Azure/Entra ID.

- **Architecture fit for your hybrid model**: Windows LAPS supports backing up local admin passwords to **Entra ID (Azure AD)** directly — not just on-prem AD — which aligns with your cloud-preference
- If your servers are domain-joined on-prem AD only (not Entra-joined/hybrid-joined), Windows LAPS falls back to **on-prem AD attribute storage**, but can still be **reported into Azure** via a custom collector (Get-LapsADPassword audit events → custom DCR → LAW) for centralized tracking/rotation compliance dashboards
- **What to track in the pilot**: password rotation compliance (last rotation date vs. policy), which accounts are LAPS-managed vs. not yet onboarded (gap analysis) — surfaced via KQL/Workbook
- **Domain account password expiry** (service accounts running ERP/middleware/SQL services) is a **separate, higher-risk item** — recommend:
  - Inventory all service accounts via **Azure Automation Runbook querying AD** (`Get-ADServiceAccount` / `Get-ADUser -Properties PasswordLastSet, PasswordExpired`)
  - Push to custom LAW table `ServiceAccount_PasswordStatus_CL`
  - Alert on accounts approaching expiry (e.g., 14/7/1 day warnings) — this is a **common ERP outage root cause** (expired service account password silently breaking SQL/middleware auth), so this has direct end-to-end monitoring value
  - **Strategic recommendation**: where feasible, migrate service accounts to **Group Managed Service Accounts (gMSA)** to eliminate password-expiry risk entirely — worth flagging as a parallel hardening initiative, not just a monitoring one

### C.2 Server Certificate Management/Tracking

Certificates (TLS certs on IIS/middleware, SQL Server encryption certs) are a classic **silent outage cause** when they expire — high-value addition to your end-to-end monitoring scope.

**Recommended approach:**
1. **Discovery/inventory**: PowerShell script (via Hybrid Runbook Worker, scheduled daily) enumerating certificate stores on each server:
```powershell
Get-ChildItem -Path Cert:\LocalMachine\My -Recurse |
Select-Object Subject, Issuer, NotAfter, Thumbprint, FriendlyName
```
2. Push results to custom LAW table `Server_CertInventory_CL` via the same DCE/DCR pattern used in Parts A/B
3. **Alerting**: KQL-based alert rule for certs expiring within a threshold window:
```kql
Server_CertInventory_CL
| where NotAfter_t < now() + 30d
| project ComputerName, Subject_s, NotAfter_t, Thumbprint_s
```
4. **For a more mature/managed path**: if certificate issuance is centralized, consider **Azure Key Vault** as the certificate authority/store of record (even for on-prem-bound certs) — Key Vault has **native expiry notification** and can serve as the single inventory source of truth, with on-prem servers pulling certs via Key Vault agent/API rather than being tracked after-the-fact. This is a bigger architectural shift (changes cert issuance workflow, not just monitoring) — worth a separate design discussion if you want to pursue it, versus the lighter-weight inventory/alert approach above for the pilot.

---

## Consolidated Pilot Scope Recommendation

| Component | Pilot Action |
|---|---|
| Network/Port (continuous) | Deploy Connection Monitor on 3 pilot servers, 3 test groups |
| REST API (continuous) | Deploy 1 Availability Test (private, on-prem test point) against ERP's primary REST endpoint |
| Network/Port (on-demand) | Build 1 custom DCR + PowerShell collector, test manual trigger via Automation Runbook |
| SQL DMV | Build Option 1 custom DCR with 3 core tables (SessionActivity, WaitStats, IOLatency) |
| Credential mgmt | Deploy Windows LAPS pilot on 3 servers; build service-account expiry runbook |
| Certificate mgmt | Build cert inventory runbook + expiry alert on same 3 servers |
| Alerting/Ticketing | Wire all above into 1 shared Action Group → ServiceNow ITSM Connector + Teams |

This gives you a **fully working, proportionate proof-of-concept** covering all four of your stated concerns (network/port runtime visibility, REST API monitoring, credential management, certificate tracking) before fleet-wide rollout.

Would you like me to detail the **ServiceNow ITSM Connector configuration steps** next, or go deeper into the **Data Collection Rule JSON/ARM template structure** for one of the custom tables above so your integrators have an implementation-ready artifact?
