# Confirmed Understanding

Good — this confirms a **hybrid observability architecture**: on-prem Windows Server fleet (ERP app tier, middleware tier, SQL tier) staying on-prem permanently, with a **cloud-hosted monitoring/alerting backend**, no SCOM today, and a requirement for dashboards + alert routing into ServiceNow/Teams/Outlook. Below is the practical mapping to Azure's current platform, since "Azure Operations Insights" (the old OMS brand) is now **Azure Monitor**.

---

## Recommended Architecture: Azure Arc + Azure Monitor Hybrid Stack

### Step 1 — Connectivity Layer: Azure Arc-enabled Servers
Since your Windows servers stay on-prem, they need to be **projected into Azure as resources** without migrating compute. This is the foundational enabler:

- Install the **Azure Arc agent (Azure Connected Machine agent)** on each on-prem Windows server
- This registers each server as an **Arc-enabled server** resource in Azure — giving it an Azure Resource ID, without moving the VM
- Once Arc-enabled, the server becomes eligible for **Azure Monitor Agent (AMA)** deployment, Azure Policy, and Azure RBAC — same as a native Azure VM
- Outbound-only HTTPS (443) connectivity to Azure endpoints required — no inbound firewall changes needed on-prem

This is the **proven, standard pattern** for exactly your scenario (on-prem servers, cloud-based monitoring backend). It's Microsoft's primary recommended path since SCOM/OMS retirement.

---

### Step 2 — Data Collection Layer (mapped to your 3 tiers)

| Your Requirement | Azure Capability | Mechanism |
|---|---|---|
| **1. Windows Server runtime status** (CPU, memory, disk, OS services, event logs) | **VM Insights** (via Azure Monitor Agent + Data Collection Rules) | AMA installed on Arc-enabled server collects perf counters, Windows Event Logs, and Windows Service status into a **Log Analytics Workspace (LAW)** |
| **2. Application connects on network ports** | **Network connection monitoring** — combination of: <br>• **Connection Monitor** (part of Azure Network Watcher, supports on-prem agents) <br>• Custom AMA Data Collection Rules for TCP port checks <br>• Or lightweight synthetic checks (Windows perf counter + PowerShell/Test-NetConnection scripted checks feeding into Log Analytics as custom logs) | This is the **weakest native fit** in Azure — see caveat below |
| **3. Database resource performance** | **SQL Insights (preview/GA depending on region)** or **SQL Server Management Pack data via AMA + custom DCR for SQL DMVs**, or simpler: **Telegraf/Custom Log ingestion** of SQL perf counters into LAW | Collects wait stats, blocking, CPU, buffer cache, I/O latency from SQL Server instances (works for on-prem SQL, not just Azure SQL) |

**Important caveat on network/port monitoring**: Azure doesn't have a single polished "is Service A able to reach Service B on port X" product for on-prem-to-on-prem traffic the way it does for Azure-native VNets. Azure Network Watcher's Connection Monitor **does support hybrid/on-prem endpoints** via installed agents, and this is the most Azure-native option — but many teams supplement it with **synthetic transaction monitoring** (scheduled scripts using `Test-NetConnection` / `tcping` pushed as custom logs into Log Analytics) for ERP↔middleware↔SQL port checks specifically. I'd flag this as the area needing a proof-of-concept before you commit.

---

### Step 3 — Unified Data Plane: Log Analytics Workspace (LAW)
All three tiers' telemetry lands in **one (or a small number of) centralized Log Analytics Workspace(s)** in Azure. This is your "single pane of glass" data layer — queried using **KQL (Kusto Query Language)**.

This satisfies your "end-to-end" requirement: you can write cross-tier KQL queries correlating, e.g., "SQL CPU spike at time T" with "middleware port 1433 connection failures at time T" with "ERP app server event log errors at time T."

---

### Step 4 — Visualization: Azure Workbooks / Dashboards
- **Azure Monitor Workbooks** — interactive, parameterized dashboards combining metrics + logs across all three tiers
- Can be organized per-tier (Infra view, Network/Port view, DB view) and per-application (ERP end-to-end view)
- Optionally embed into **Azure Managed Grafana** if your IT support team prefers Grafana-style dashboards
- Shareable via Azure Portal, or exported to **Power BI** if that's your org's standard reporting tool

---

### Step 5 — Alerting & Ticketing Integration (your ServiceNow/Teams/Outlook requirement)

This is well-supported and proven:

- **Azure Monitor Alert Rules** (metric alerts, log query alerts on KQL, activity log alerts) trigger on thresholds across all three tiers
- **Action Groups** fan out each alert to multiple channels simultaneously:
  - **Email/Outlook** — native, no extra config
  - **Microsoft Teams** — native Teams webhook connector action, or via Logic App for richer adaptive cards
  - **ServiceNow** — Microsoft provides a **certified ITSM Connector (IT Service Management Connector)** that integrates Azure Monitor alerts directly with ServiceNow, creating/updating incidents automatically — this is the standard, supported path (also works with alternatives, but ServiceNow is explicitly first-class)

This gives you closed-loop: **detect → alert → auto-ticket in ServiceNow → notify Teams/Outlook**.

---

## Summary Architecture Flow

```
On-Prem Windows Servers (ERP / Middleware / SQL)
        │
        ├─ Azure Arc agent (registers server in Azure, no migration)
        │
        ├─ Azure Monitor Agent (AMA) + Data Collection Rules
        │      ├─ VM Insights → OS/runtime health
        │      ├─ Network Watcher Connection Monitor / custom port-check DCR → connectivity
        │      └─ SQL perf counters/DMVs (custom DCR or SQL Insights) → DB performance
        │
        ▼
Log Analytics Workspace (centralized, cloud-hosted, KQL queryable)
        │
        ├─► Azure Workbooks / Managed Grafana → Dashboards for IT Support
        │
        └─► Azure Monitor Alert Rules → Action Groups
                   ├─ ServiceNow (ITSM Connector) → Incident creation
                   ├─ Microsoft Teams → Channel notification
                   └─ Outlook/Email → Notification
```

---

## What I'd Recommend as Next Steps
1. **Pilot on a small subset** (e.g., 2-3 servers: one ERP, one middleware, one SQL) to validate Arc onboarding, AMA data flow, and port-monitoring approach before fleet-wide rollout
2. **Validate the network/port monitoring approach specifically** — this is the one area without an out-of-the-box turnkey Azure product; decide between Network Watcher Connection Monitor vs. custom synthetic scripts early, since it affects your Data Collection Rule design
3. **Confirm ServiceNow instance readiness** for the ITSM Connector (requires a ServiceNow app installation + Azure AD app registration)
4. **Decide LAW topology** — one workspace vs. per-environment workspaces (affects cost, RBAC, and query scope)

Would you like me to go deeper into any one of these four next steps — for example, a detailed pilot design for the network/port monitoring gap, or the specific Data Collection Rule configuration for SQL Server DMV collection?
