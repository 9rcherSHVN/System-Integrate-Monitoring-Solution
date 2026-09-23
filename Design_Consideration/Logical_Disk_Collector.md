## Phase 2: Meaning of “AMA Performance Counter Collection Is Not Approved”

### Direct Meaning

This does **not** mean Azure Monitor Agent is technically unsupported.

It means the organization may decide that installing or using **Azure Monitor Agent (AMA)** for Windows performance-counter collection is not permitted or not acceptable in a particular environment.

This is an **architecture, security, operational, vendor-support, or governance decision**.

The alternative custom logical-disk collector exists so the pilot can still collect selected disk metrics when AMA is unavailable.

---

### Standard Preferred Design

When approved, AMA is the preferred approach for native Windows disk metrics:

```text
Azure Arc-enabled Windows Server
    │
    ├── Azure Monitor Agent
    └── AMA Data Collection Rule
           └── LogicalDisk performance counters
                 │
                 ▼
             Log Analytics
```

AMA can collect:

- `% Free Space`
- `Free Megabytes`
- `Avg. Disk sec/Read`
- `Avg. Disk sec/Write`
- `Current Disk Queue Length`
- `Disk Transfers/sec`

Advantages:

- Microsoft-supported native monitoring agent.
- No custom PowerShell scan needed for standard counters.
- Efficient continuous counter collection.
- Central DCR configuration and association.
- Lower custom-code maintenance.
- Better fit for broad Windows infrastructure telemetry.

---

### Reasons AMA May Not Be Approved

## 2.1 Security or Endpoint-Agent Policy

Some organizations limit the number of agents on ERP or vendor-managed servers.

Possible concerns:

- Third-party ERP vendor supports only a fixed software/agent baseline.
- Security team requires endpoint-agent review before installation.
- Existing endpoint protection, backup, management, or monitoring agents already consume permitted agent capacity.
- Server hardening policy prohibits unapproved extensions/services.
- Production change-management process for agent installation is lengthy.

Example:

```text
ERP vendor statement:
“Only vendor-approved agents may be installed on the application server.”
```

In that case, a lightweight scheduled PowerShell script may be easier to approve than a continuously running monitoring agent—although it still requires security review.

---

## 2.2 Azure Arc Is Approved but AMA Is Not Yet Approved

Azure Arc and AMA are separate components:

| Component | Function |
|---|---|
| Azure Arc Connected Machine agent | Registers and manages on-premises server as an Azure resource. |
| Azure Monitor Agent | Collects monitoring telemetry such as event logs and performance counters. |

An organization may approve Azure Arc for inventory/governance but defer AMA due to:

- Incomplete agent assessment.
- Pending proxy/firewall testing.
- Pending licensing/cost review.
- Pending data-classification review.
- Planned enterprise monitoring-agent standardization.

The custom collector can use Azure Arc managed identity without needing AMA.

---

## 2.3 Narrow Pilot Scope

The pilot may need only a few metrics from a few ERP volumes:

```text
C: operating system volume
D: ERP application/log volume
```

If the immediate goal is to validate:

- Logs Ingestion API.
- DCR custom streams.
- Arc managed identity.
- Dashboard and alert workflow.
- ERP log-directory correlation.

then a custom collector is a controlled fallback that avoids deploying AMA before the broader monitoring design is approved.

---

## 2.4 Existing Monitoring Tool Is Already Responsible for OS Counters

The organization might already collect CPU, memory, disk, and Windows events using another tool, such as:

- Existing enterprise monitoring platform.
- Vendor monitoring tool.
- SCOM.
- A commercial APM or infrastructure monitoring product.
- Endpoint/observability platform.

In that model, IT may not want AMA collecting duplicate data. The custom collector can provide only the new ERP-specific logical-disk fields needed to correlate with ERP log-directory growth.

---

## 2.5 Network/Proxy or DCR Association Restrictions

AMA requires:

- AMA extension deployment.
- DCR association to Arc-enabled servers.
- Network reachability to Azure Monitor endpoints.
- Approved agent update lifecycle.
- Operational ownership for the agent.

If these controls are not ready within the pilot time frame, the custom collector can still use:

```text
Windows Task Scheduler
    → Azure Arc managed identity
    → Logs Ingestion API
```

However, this is a fallback, not automatically a better architecture.

---

## 2.6 Data Collection Governance

A security team may be concerned that a general AMA DCR could collect more than the intended data if poorly configured, such as:

- Broad Windows event logs.
- Excessive performance counters.
- IIS logs.
- Application logs.
- Security logs.

This concern should usually be addressed by precise DCR design—not by rejecting AMA entirely. But during a restricted pilot, a custom collector can be easier to constrain because it submits only explicit fields.

Example custom record:

```json name=WindowsDiskMetrics-example.json
{
  "Computer": "ERP-APP-01",
  "Drive": "D:",
  "TotalGB": 500,
  "FreeGB": 85,
  "FreePercent": 17,
  "ReadLatencyMs": 4.2,
  "WriteLatencyMs": 6.5,
  "CollectionStatus": "Success"
}
```

There is no broad event/log collection in that payload.

---

# 3. Why the Custom Logical-Disk Collector Exists

The custom collector is a **controlled alternative** to AMA. It is not intended to replace AMA across the enterprise.

```text
Custom PowerShell Disk Collector
    │
    ├── Win32_LogicalDisk
    │      └── capacity, free space, file system, label
    │
    └── Windows LogicalDisk performance counters
           └── latency, queue, transfer rate
                │
                ▼
         Structured JSON only
                │
                ▼
   Logs Ingestion API / custom DCR
                │
                ▼
     WindowsDiskMetrics_CL
```

It provides these specific benefits:

| Benefit | Explanation |
|---|---|
| No AMA requirement | Works where AMA deployment has not been approved. |
| Same identity model | Uses Azure Arc system-assigned managed identity. |
| Same Azure architecture | Uses DCE, DCR, custom tables, KQL, Workbooks, and alerts. |
| Narrow collection scope | Reports only selected volumes and fields. |
| Application tagging | Can label a drive as `ERP`, `WindowsOS`, `Middleware`, or `SQL`. |
| Direct ERP correlation | Can correlate `D:\ERP\Logs` growth with free capacity on the same logical volume. |
| Controlled schedule | Runs every 15 minutes or other approved interval. |
| Lower agent footprint | Uses scheduled PowerShell rather than a continuously running additional monitoring agent. |

---

# 4. Technical Limitations of the Custom Disk Collector

The custom collector has trade-offs and should be evaluated carefully.

| Limitation | Why it matters |
|---|---|
| Not continuous telemetry | A 15-minute scheduled sample can miss transient latency spikes. |
| Less mature than AMA | Custom code must be tested, signed, maintained, versioned, and supported. |
| Performance-counter localization | Counter names may differ on non-English Windows installations. |
| Scheduled-task dependency | Task Scheduler, account rights, and local script deployment must remain healthy. |
| Additional Logs Ingestion API dependency | DCR, DCE, RBAC, endpoint, and replay behavior must be maintained. |
| More custom alert logic | AMA `Perf` table is standard; custom table queries are organization-owned. |
| Data point accuracy | A short `Get-Counter` sample is less representative than sustained collection. |
| Potential duplicate data | Avoid running AMA and custom collector for same monitoring purpose long-term. |

---

# 5. Recommended Architecture Decision

Use this decision rule.

```text
Question: Is Azure Monitor Agent approved and operational for the target server?

Yes
 └── Use AMA for logical-disk capacity and performance counters.
     Use custom PowerShell only for ERP folder-size/growth metrics.

No
 └── Use custom logical-disk PowerShell collector.
     Use custom DCR → WindowsDiskMetrics_CL.
     Keep collection limited to approved drives and 15-minute cadence.
```

## Preferred Final State

For a mature production implementation:

```text
AMA
 └── Standard Windows infrastructure telemetry
      ├── CPU
      ├── Memory
      ├── Disk capacity
      ├── Disk performance
      └── Windows events

Custom PowerShell collectors
 └── Application-specific telemetry
      ├── ERP log folder size/growth
      ├── Certificate inventory
      ├── SQL DMVs
      ├── ERP REST health checks
      └── On-demand TCP connection diagnostics
```

This minimizes custom code while preserving the ability to collect business-application-specific operational data.
