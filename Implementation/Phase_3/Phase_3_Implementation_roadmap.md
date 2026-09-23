# Phase 3 — Certificate Monitoring: KQL, Alerts, Teams/Email, and Workbook

This phase turns `ServerCertificateInventory_CL` and `CollectorHealth_CL` into an operational certificate-monitoring service.

It provides:

- Certificate-health KQL queries.
- Scheduled-query alert rules in Bicep.
- Email through the existing Action Group.
- Teams through Azure Monitor → Action Group → Logic App → Teams.
- A certificate-focused Azure Monitor Workbook.
- Acceptance and fault-test procedures.

---

## 1. Certificate Monitoring Operating Model

```text
Daily certificate collector
    │
    ▼
ServerCertificateInventory_CL
    │
    ├── Certificate expired
    ├── Certificate expires ≤ 7 days
    ├── Certificate expires ≤ 14 days
    ├── Certificate expires ≤ 30 days
    ├── WebHosting certificate lacks private key
    ├── Collector failed
    └── Collector did not report
            │
            ▼
Azure Monitor scheduled-query rules
            │
            ▼
Action Group
    ├── Outlook/email
    └── Logic App action
            │
            ▼
Microsoft Teams channel
```

## Alert Severity Model

| Condition | Severity | Operational response |
|---|---:|---|
| Certificate already expired | Sev 1 | Immediate investigation and certificate replacement/binding validation. |
| Certificate expires within 7 days | Sev 2 | Urgent renewal/replacement action. |
| Certificate expires within 14 days | Sev 2 | Certificate owner action required. |
| Certificate expires within 30 days | Sev 3 | Planned renewal work item. |
| Required private key missing | Sev 2 | Certificate binding/service risk; investigate. |
| Certificate collector reports failure | Sev 2 | Restore monitoring collection. |
| No certificate inventory in 30 hours | Sev 2 | Treat as monitoring blind spot. |

Do not create separate 30-, 14-, and 7-day tickets every day for the same certificate unless suppression/correlation is configured. The recommended starting model is:

- One **30-day warning** alert.
- One **14-day high-priority** alert.
- One **7-day urgent** alert.
- One **expired** critical alert.

Enable auto-mitigation for all alerts so Azure Monitor sends resolution notifications when the latest certificate inventory indicates that the certificate has been renewed or removed.

---

# 2. Certificate KQL Query Set

Create this folder:

```text
kql/
└── certificate/
    ├── CurrentCertificateInventory.kql
    ├── ExpiredCertificates.kql
    ├── CertificatesExpiring30Days.kql
    ├── CertificatesExpiring14Days.kql
    ├── CertificatesExpiring7Days.kql
    ├── MissingPrivateKeys.kql
    ├── CertificateCollectorFailures.kql
    └── MissingCertificateInventory.kql
```

---

## 2.1 Current Certificate Inventory

This query returns the latest known state of each certificate on each server/store combination.

```kusto name=kql/certificate/CurrentCertificateInventory.kql
ServerCertificateInventory_CL
| where TimeGenerated > ago(7d)
| summarize arg_max(TimeGenerated, *) by Computer, StoreLocation, StoreName, Thumbprint
| project
    TimeGenerated,
    Computer,
    Environment,
    Application,
    StoreLocation,
    StoreName,
    Subject,
    DnsNames,
    Issuer,
    Thumbprint,
    SerialNumber,
    NotBefore,
    NotAfter,
    DaysUntilExpiry,
    HasPrivateKey,
    IisBindings,
    CollectionStatus,
    CollectorVersion
| order by DaysUntilExpiry asc, Computer asc
```

---

## 2.2 Expired Certificates

```kusto name=kql/certificate/ExpiredCertificates.kql
ServerCertificateInventory_CL
| where TimeGenerated > ago(7d)
| summarize arg_max(TimeGenerated, *) by Computer, StoreLocation, StoreName, Thumbprint
| where CollectionStatus == "Success"
| where DaysUntilExpiry < 0 or NotAfter < now()
| project
    Computer,
    Environment,
    Application,
    StoreName,
    Subject,
    DnsNames,
    NotAfter,
    DaysUntilExpiry,
    IisBindings,
    Thumbprint
| order by DaysUntilExpiry asc
```

---

## 2.3 Certificates Expiring Within 30 Days

```kusto name=kql/certificate/CertificatesExpiring30Days.kql
ServerCertificateInventory_CL
| where TimeGenerated > ago(7d)
| summarize arg_max(TimeGenerated, *) by Computer, StoreLocation, StoreName, Thumbprint
| where CollectionStatus == "Success"
| where DaysUntilExpiry between (0 .. 30)
| project
    Computer,
    Environment,
    Application,
    StoreName,
    Subject,
    DnsNames,
    Issuer,
    NotAfter,
    DaysUntilExpiry,
    IisBindings,
    Thumbprint
| order by DaysUntilExpiry asc
```

---

## 2.4 Certificates Expiring Within 14 Days

```kusto name=kql/certificate/CertificatesExpiring14Days.kql
ServerCertificateInventory_CL
| where TimeGenerated > ago(7d)
| summarize arg_max(TimeGenerated, *) by Computer, StoreLocation, StoreName, Thumbprint
| where CollectionStatus == "Success"
| where DaysUntilExpiry between (0 .. 14)
| project
    Computer,
    Environment,
    Application,
    StoreName,
    Subject,
    DnsNames,
    Issuer,
    NotAfter,
    DaysUntilExpiry,
    IisBindings,
    Thumbprint
| order by DaysUntilExpiry asc
```

---

## 2.5 Certificates Expiring Within Seven Days

```kusto name=kql/certificate/CertificatesExpiring7Days.kql
ServerCertificateInventory_CL
| where TimeGenerated > ago(7d)
| summarize arg_max(TimeGenerated, *) by Computer, StoreLocation, StoreName, Thumbprint
| where CollectionStatus == "Success"
| where DaysUntilExpiry between (0 .. 7)
| project
    Computer,
    Environment,
    Application,
    StoreName,
    Subject,
    DnsNames,
    Issuer,
    NotAfter,
    DaysUntilExpiry,
    IisBindings,
    Thumbprint
| order by DaysUntilExpiry asc
```

---

## 2.6 WebHosting Certificates Without a Private Key

A certificate in the WebHosting store intended for IIS HTTPS normally requires an associated private key. This query highlights a potential service/binding risk.

```kusto name=kql/certificate/MissingPrivateKeys.kql
ServerCertificateInventory_CL
| where TimeGenerated > ago(7d)
| summarize arg_max(TimeGenerated, *) by Computer, StoreLocation, StoreName, Thumbprint
| where CollectionStatus == "Success"
| where StoreLocation == "LocalMachine"
| where StoreName == "WebHosting"
| where HasPrivateKey == false
| project
    Computer,
    Environment,
    Application,
    Subject,
    DnsNames,
    NotAfter,
    DaysUntilExpiry,
    IisBindings,
    Thumbprint
| order by DaysUntilExpiry asc
```

> Review this rule with the Windows/IIS team. Some certificate records may legitimately have no private key, so the production alert should be limited to certificates that are expected to be serving HTTPS or have an IIS binding.

---

## 2.7 Certificate Collector Failure

```kusto name=kql/certificate/CertificateCollectorFailures.kql
CollectorHealth_CL
| where TimeGenerated > ago(48h)
| where CollectorName == "CertificateInventory"
| summarize arg_max(TimeGenerated, *) by Computer, CollectorName
| where Status != "Success"
| project
    TimeGenerated,
    Computer,
    Status,
    TargetScope,
    RecordsCollected,
    RecordsSubmitted,
    DurationSeconds,
    ErrorCode,
    ErrorMessage,
    CollectorVersion
| order by TimeGenerated desc
```

---

## 2.8 Missing Certificate Inventory

This detects a server that has not delivered certificate inventory in the expected 30-hour collection window.

For the pilot, explicitly list monitored certificate hosts. In production, generate this inventory from Azure Arc tags, CMDB, or an approved resource inventory.

```kusto name=kql/certificate/MissingCertificateInventory.kql
let ExpectedCertificateCollectorHosts = datatable(Computer:string)
[
    "ERP-MW-01"
];
ExpectedCertificateCollectorHosts
| join kind=leftouter (
    ServerCertificateInventory_CL
    | where TimeGenerated > ago(30h)
    | summarize LastInventory = max(TimeGenerated) by Computer
) on Computer
| where isempty(LastInventory) or LastInventory < ago(30h)
| project
    Computer,
    LastInventory,
    Status = "Certificate inventory missing or stale"
```

---

# 3. Alert Query Design

Azure Monitor scheduled-query alerts require an aggregation condition. For all certificate alerts, use:

```text
timeAggregation: Count
operator: GreaterThan
threshold: 0
```

The KQL query must return one or more rows only when a problem exists.

The certificate data is collected daily, so alert rules should typically evaluate every **6 hours** with a **30-hour window**. This reduces needless query executions while allowing a small buffer for delayed collection.

For expired and near-expiry certificates, an evaluation frequency of **6 hours** is sufficient because certificate expiration is not second-by-second telemetry.

---

# 4. Certificate Alert Bicep Module

Create:

```text
azure/modules/certificate-alerts.bicep
```

```bicep name=azure/modules/certificate-alerts.bicep
@description('Azure region for scheduled query rules.')
param location string

@description('Log Analytics Workspace resource ID.')
param workspaceResourceId string

@description('Action Group resource ID.')
param actionGroupResourceId string

@description('Deployment environment.')
param environment string

@description('Prefix used for alert resource names.')
param alertNamePrefix string = 'sqra-erpmon-cert'

var evaluationFrequency = 'PT6H'
var windowSize = 'PT30H'

var expiredCertificatesQuery = '''
ServerCertificateInventory_CL
| where TimeGenerated > ago(30h)
| summarize arg_max(TimeGenerated, *) by Computer, StoreLocation, StoreName, Thumbprint
| where CollectionStatus == "Success"
| where DaysUntilExpiry < 0 or NotAfter < now()
| project Computer, StoreName, Subject, DnsNames, NotAfter, DaysUntilExpiry, IisBindings, Thumbprint
'''

var expiringWithinSevenDaysQuery = '''
ServerCertificateInventory_CL
| where TimeGenerated > ago(30h)
| summarize arg_max(TimeGenerated, *) by Computer, StoreLocation, StoreName, Thumbprint
| where CollectionStatus == "Success"
| where DaysUntilExpiry between (0 .. 7)
| project Computer, StoreName, Subject, DnsNames, NotAfter, DaysUntilExpiry, IisBindings, Thumbprint
'''

var expiringWithinFourteenDaysQuery = '''
ServerCertificateInventory_CL
| where TimeGenerated > ago(30h)
| summarize arg_max(TimeGenerated, *) by Computer, StoreLocation, StoreName, Thumbprint
| where CollectionStatus == "Success"
| where DaysUntilExpiry between (8 .. 14)
| project Computer, StoreName, Subject, DnsNames, NotAfter, DaysUntilExpiry, IisBindings, Thumbprint
'''

var expiringWithinThirtyDaysQuery = '''
ServerCertificateInventory_CL
| where TimeGenerated > ago(30h)
| summarize arg_max(TimeGenerated, *) by Computer, StoreLocation, StoreName, Thumbprint
| where CollectionStatus == "Success"
| where DaysUntilExpiry between (15 .. 30)
| project Computer, StoreName, Subject, DnsNames, NotAfter, DaysUntilExpiry, IisBindings, Thumbprint
'''

var missingPrivateKeyQuery = '''
ServerCertificateInventory_CL
| where TimeGenerated > ago(30h)
| summarize arg_max(TimeGenerated, *) by Computer, StoreLocation, StoreName, Thumbprint
| where CollectionStatus == "Success"
| where StoreLocation == "LocalMachine" and StoreName == "WebHosting"
| where HasPrivateKey == false
| where isnotempty(IisBindings)
| project Computer, StoreName, Subject, DnsNames, NotAfter, DaysUntilExpiry, IisBindings, Thumbprint
'''

var certificateCollectorFailureQuery = '''
CollectorHealth_CL
| where TimeGenerated > ago(30h)
| where CollectorName == "CertificateInventory"
| summarize arg_max(TimeGenerated, *) by Computer, CollectorName
| where Status != "Success"
| project Computer, Status, ErrorCode, ErrorMessage, RecordsCollected, RecordsSubmitted, DurationSeconds
'''

resource expiredCertificatesAlert 'Microsoft.Insights/scheduledQueryRules@2023-12-01' = {
  name: '${alertNamePrefix}-expired-${environment}-001'
  location: location
  tags: {
    Application: 'ERP-Monitoring'
    Environment: environment
    ManagedBy: 'Bicep'
    AlertDomain: 'Certificate'
    Severity: 'Sev1'
  }
  properties: {
    displayName: 'ERP Certificate Expired'
    description: 'A WebHosting certificate used by the ERP monitoring scope has expired.'
    enabled: true
    scopes: [
      workspaceResourceId
    ]
    severity: 1
    evaluationFrequency: evaluationFrequency
    windowSize: windowSize
    autoMitigate: true
    muteActionsDuration: 'PT6H'
    criteria: {
      allOf: [
        {
          query: expiredCertificatesQuery
          timeAggregation: 'Count'
          operator: 'GreaterThan'
          threshold: 0
          failingPeriods: {
            numberOfEvaluationPeriods: 1
            minFailingPeriodsToAlert: 1
          }
        }
      ]
    }
    actions: {
      actionGroups: [
        actionGroupResourceId
      ]
      customProperties: {
        Service = 'ERP Certificate Monitoring'
        Runbook = 'Renew or replace expired certificate and validate IIS HTTPS binding.'
      }
    }
  }
}

resource expiringSevenDaysAlert 'Microsoft.Insights/scheduledQueryRules@2023-12-01' = {
  name: '${alertNamePrefix}-7days-${environment}-001'
  location: location
  tags: {
    Application: 'ERP-Monitoring'
    Environment: environment
    ManagedBy: 'Bicep'
    AlertDomain: 'Certificate'
    Severity: 'Sev2'
  }
  properties: {
    displayName: 'ERP Certificate Expires Within 7 Days'
    description: 'A WebHosting certificate used by the ERP monitoring scope expires within seven days.'
    enabled: true
    scopes: [
      workspaceResourceId
    ]
    severity: 2
    evaluationFrequency: evaluationFrequency
    windowSize: windowSize
    autoMitigate: true
    muteActionsDuration: 'PT12H'
    criteria: {
      allOf: [
        {
          query: expiringWithinSevenDaysQuery
          timeAggregation: 'Count'
          operator: 'GreaterThan'
          threshold: 0
          failingPeriods: {
            numberOfEvaluationPeriods: 1
            minFailingPeriodsToAlert: 1
          }
        }
      ]
    }
    actions: {
      actionGroups: [
        actionGroupResourceId
      ]
      customProperties: {
        Service = 'ERP Certificate Monitoring'
        Runbook = 'Prioritize certificate renewal and confirm replacement binding before expiry.'
      }
    }
  }
}

resource expiringFourteenDaysAlert 'Microsoft.Insights/scheduledQueryRules@2023-12-01' = {
  name: '${alertNamePrefix}-14days-${environment}-001'
  location: location
  tags: {
    Application: 'ERP-Monitoring'
    Environment: environment
    ManagedBy: 'Bicep'
    AlertDomain: 'Certificate'
    Severity: 'Sev2'
  }
  properties: {
    displayName: 'ERP Certificate Expires Within 14 Days'
    description: 'A WebHosting certificate used by the ERP monitoring scope expires within 14 days.'
    enabled: true
    scopes: [
      workspaceResourceId
    ]
    severity: 2
    evaluationFrequency: evaluationFrequency
    windowSize: windowSize
    autoMitigate: true
    muteActionsDuration: 'PT24H'
    criteria: {
      allOf: [
        {
          query: expiringWithinFourteenDaysQuery
          timeAggregation: 'Count'
          operator: 'GreaterThan'
          threshold: 0
          failingPeriods: {
            numberOfEvaluationPeriods: 1
            minFailingPeriodsToAlert: 1
          }
        }
      ]
    }
    actions: {
      actionGroups: [
        actionGroupResourceId
      ]
      customProperties: {
        Service = 'ERP Certificate Monitoring'
        Runbook = 'Schedule certificate renewal and test target IIS binding.'
      }
    }
  }
}

resource expiringThirtyDaysAlert 'Microsoft.Insights/scheduledQueryRules@2023-12-01' = {
  name: '${alertNamePrefix}-30days-${environment}-001'
  location: location
  tags: {
    Application: 'ERP-Monitoring'
    Environment: environment
    ManagedBy: 'Bicep'
    AlertDomain: 'Certificate'
    Severity: 'Sev3'
  }
  properties: {
    displayName: 'ERP Certificate Expires Within 30 Days'
    description: 'A WebHosting certificate used by the ERP monitoring scope expires within 30 days.'
    enabled: true
    scopes: [
      workspaceResourceId
    ]
    severity: 3
    evaluationFrequency: evaluationFrequency
    windowSize: windowSize
    autoMitigate: true
    muteActionsDuration: 'P1D'
    criteria: {
      allOf: [
        {
          query: expiringWithinThirtyDaysQuery
          timeAggregation: 'Count'
          operator: 'GreaterThan'
          threshold: 0
          failingPeriods: {
            numberOfEvaluationPeriods: 1
            minFailingPeriodsToAlert: 1
          }
        }
      ]
    }
    actions: {
      actionGroups: [
        actionGroupResourceId
      ]
      customProperties: {
        Service = 'ERP Certificate Monitoring'
        Runbook = 'Create certificate renewal work item and identify certificate owner.'
      }
    }
  }
}

resource missingPrivateKeyAlert 'Microsoft.Insights/scheduledQueryRules@2023-12-01' = {
  name: '${alertNamePrefix}-privatekey-${environment}-001'
  location: location
  tags: {
    Application: 'ERP-Monitoring'
    Environment: environment
    ManagedBy: 'Bicep'
    AlertDomain: 'Certificate'
    Severity: 'Sev2'
  }
  properties: {
    displayName: 'ERP IIS-Bound Certificate Missing Private Key'
    description: 'An IIS-bound certificate in the WebHosting store does not report an associated private key.'
    enabled: true
    scopes: [
      workspaceResourceId
    ]
    severity: 2
    evaluationFrequency: evaluationFrequency
    windowSize: windowSize
    autoMitigate: true
    muteActionsDuration: 'PT12H'
    criteria: {
      allOf: [
        {
          query: missingPrivateKeyQuery
          timeAggregation: 'Count'
          operator: 'GreaterThan'
          threshold: 0
          failingPeriods: {
            numberOfEvaluationPeriods: 1
            minFailingPeriodsToAlert: 1
          }
        }
      ]
    }
    actions: {
      actionGroups: [
        actionGroupResourceId
      ]
      customProperties: {
        Service = 'ERP Certificate Monitoring'
        Runbook = 'Validate certificate import, private-key association, and IIS HTTPS binding.'
      }
    }
  }
}

resource collectorFailureAlert 'Microsoft.Insights/scheduledQueryRules@2023-12-01' = {
  name: '${alertNamePrefix}-collectorfailure-${environment}-001'
  location: location
  tags: {
    Application: 'ERP-Monitoring'
    Environment: environment
    ManagedBy: 'Bicep'
    AlertDomain: 'MonitoringHealth'
    Severity: 'Sev2'
  }
  properties: {
    displayName: 'ERP Certificate Collector Failed'
    description: 'The certificate inventory collector reported a failure or partial failure.'
    enabled: true
    scopes: [
      workspaceResourceId
    ]
    severity: 2
    evaluationFrequency: evaluationFrequency
    windowSize: windowSize
    autoMitigate: true
    muteActionsDuration: 'PT6H'
    criteria: {
      allOf: [
        {
          query: certificateCollectorFailureQuery
          timeAggregation: 'Count'
          operator: 'GreaterThan'
          threshold: 0
          failingPeriods: {
            numberOfEvaluationPeriods: 1
            minFailingPeriodsToAlert: 1
          }
        }
      ]
    }
    actions: {
      actionGroups: [
        actionGroupResourceId
      ]
      customProperties: {
        Service = 'ERP Certificate Monitoring'
        Runbook = 'Review task history, collector local logs, Azure Arc identity, and ingestion configuration.'
      }
    }
  }
}

output expiredCertificatesAlertResourceId string = expiredCertificatesAlert.id
output expiringSevenDaysAlertResourceId string = expiringSevenDaysAlert.id
output expiringFourteenDaysAlertResourceId string = expiringFourteenDaysAlert.id
output expiringThirtyDaysAlertResourceId string = expiringThirtyDaysAlert.id
output missingPrivateKeyAlertResourceId string = missingPrivateKeyAlert.id
output collectorFailureAlertResourceId string = collectorFailureAlert.id
```

---

# 5. Add Certificate Alerts to `main.bicep`

Add these parameters:

```bicep name=azure/main.bicep
@description('Enable certificate monitoring alert rules.')
param enableCertificateAlerts bool = true

@description('Certificate alert resource-name prefix.')
param certificateAlertNamePrefix string = 'sqra-erpmon-cert'
```

Then add this module after the Action Group module:

```bicep name=azure/main.bicep
module certificateAlerts './modules/certificate-alerts.bicep' = if (enableCertificateAlerts) {
  name: 'deploy-certificate-alerts'
  params: {
    location: location
    workspaceResourceId: workspace.outputs.workspaceResourceId
    actionGroupResourceId: actionGroup.outputs.actionGroupResourceId
    environment: environment
    alertNamePrefix: certificateAlertNamePrefix
  }
  dependsOn: [
    tables
    actionGroup
  ]
}
```

Add to the pilot parameter file:

```bicep name=azure/parameters/pilot.bicepparam
param enableCertificateAlerts = true
param certificateAlertNamePrefix = 'sqra-erpmon-cert'
```

---

# 6. Email Notification Design

The Phase 2 Action Group already sends email to the operations distribution list.

Recommended email recipients:

```text
erp-platform-support@example.com
windows-infrastructure-support@example.com
certificate-management@example.com
```

For the pilot, begin with one operational distribution list. Avoid sending directly to individual people because alert ownership changes over time.

## Email Operational Requirements

- Enable the Common Alert Schema.
- Use a distribution list with clear ownership.
- Include alert resolution notifications.
- Include a workbook link and operational runbook link in alert custom properties where possible.
- Review alerts at least daily during the pilot.
- Avoid separate distribution lists for the 30/14/7-day alert tiers until the alert-volume baseline is known.

---

# 7. Teams Notification Architecture

## Recommended Pattern

```text
Azure Monitor scheduled-query alert
    │
    ▼
Action Group
    │
    ▼
Logic App action
    │
    ├── HTTP request trigger
    ├── Receive Common Alert Schema
    ├── Parse alert details
    ├── Format Adaptive Card
    └── Post card to approved Teams channel
```

Use **Common Alert Schema** for the Action Group → Logic App call. This keeps Teams automation independent of individual alert payload variations.

Microsoft documents Logic Apps as the supported flexible approach for customizing Azure Monitor alert notifications, and Common Alert Schema provides the consistent alert payload format.[[1]](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-logic-apps)[[2]](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-common-schema)

## Recommended Teams Channel

```text
Team: ERP Operations
Channel: Monitoring Alerts
```

Avoid sending directly to group chats or personal chats because these are harder to govern, audit, and support.

## Adaptive Card Content

The Teams card should include:

| Field | Example |
|---|---|
| Alert status | Activated / Resolved |
| Severity | Sev 1 / Sev 2 / Sev 3 |
| Alert rule | ERP Certificate Expired |
| Server | `ERP-MW-01` |
| Certificate subject | `CN=api.erp.example.com` |
| Expiry date | `2026-10-15T23:59:59Z` |
| Days remaining | `7` |
| IIS binding | `ERP API\|*:443:api.erp.example.com\|My` |
| Runbook | Link to certificate-renewal procedure |
| Workbook | Link to certificate workbook |
| Alert portal link | Azure Monitor alert link |

## Logic App Security

- Use a managed identity for Logic App connectors where supported.
- Restrict who can modify the workflow.
- Use a dedicated Teams service connection/owner account approved by collaboration administration.
- Do not include certificate private-key information, tokens, or secrets in the Teams card.
- Ensure alerts are sent only to the approved operations channel.

---

# 8. Teams Logic App Implementation Outline

The Teams workflow is deliberately presented as a controlled integration design rather than a hard-coded legacy Teams webhook.

## Logic App Steps

1. Create a Logic App:
   ```text
   la-erpmon-alerts-pilot-001
   ```

2. Create an HTTP request trigger:
   ```text
   When an HTTP request is received
   ```

3. Use the Azure Monitor Common Alert Schema as the expected payload.

4. Add a Parse JSON action, if required by the selected Logic App design.

5. Add a condition:
   ```text
   AlertStatus == "Activated"
   ```
   Optional: post resolved alerts in the same thread or post a resolution message.

6. Add Teams connector action:
   ```text
   Post adaptive card in a channel
   ```

7. Create an Adaptive Card containing:
   - Severity
   - Alert rule
   - Description
   - Firing/resolved status
   - Link to Azure Portal alert
   - Alert context table/results
   - Workbook/runbook links

8. Add the Logic App as an Action Group action.

9. Enable Common Alert Schema for that Action Group receiver.

10. Test with a deliberately created validation certificate record.

---

# 9. Workbook Design

Create this folder:

```text
workbooks/
└── certificate-monitoring.workbook.json
```

The Workbook should be deployed as `Microsoft.Insights/workbooks` through Bicep. The serialized JSON can be created in the Azure Portal Workbook editor first and then exported through **Advanced Editor** for source control.

## Workbook Sections

| Section | Visualization | Data source |
|---|---|---|
| Certificate health summary | KPI tiles | Current inventory query |
| Expired certificates | Grid | Expired certificate query |
| Expiring in 7 days | Grid | 7-day query |
| Expiring in 14 days | Grid | 14-day query |
| Expiring in 30 days | Grid | 30-day query |
| Expiry trend | Time chart | Current/latest records by expiry date |
| Server/store distribution | Bar chart | Certificate count by server/store |
| IIS binding risk | Grid | Missing private key / binding query |
| Collector health | Grid and KPI | `CollectorHealth_CL` |
| Stale inventory | Grid | Missing inventory query |

## Workbook Parameters

Use these Workbook parameters:

| Parameter | Type | Purpose |
|---|---|---|
| `TimeRange` | Time range | User-selectable query time range |
| `Environment` | Dropdown | Pilot/test/production filter |
| `Computer` | Dropdown | Server filter |
| `Application` | Dropdown | ERP/middleware filter |
| `ExpiryDays` | Dropdown | 7, 14, 30, 60 days |

---

# 10. Certificate Workbook JSON Template

This is a compact starter Workbook definition. It provides a title, certificate summary, expired grid, expiring grid, and collector-health grid.

```json name=workbooks/certificate-monitoring.workbook.json
{
  "version": "Notebook/1.0",
  "items": [
    {
      "type": 1,
      "content": {
        "json": "# ERP Certificate Monitoring\nThis workbook shows the current certificate inventory from `Cert:\\LocalMachine\\WebHosting`, certificate expiry risks, IIS binding context, and collector health."
      },
      "name": "CertificateMonitoringTitle"
    },
    {
      "type": 9,
      "content": {
        "version": "KqlParameterItem/1.0",
        "parameters": [
          {
            "id": "c6ca35b2-00ad-4f96-9fe7-73c836149080",
            "version": "KqlParameterItem/1.0",
            "name": "TimeRange",
            "type": 4,
            "isRequired": true,
            "value": {
              "durationMs": 604800000
            },
            "typeSettings": {
              "selectableValues": [
                {
                  "durationMs": 86400000
                },
                {
                  "durationMs": 604800000
                },
                {
                  "durationMs": 2592000000
                }
              ]
            }
          }
        ],
        "style": "pills",
        "queryType": 0,
        "resourceType": "microsoft.operationalinsights/workspaces"
      },
      "name": "CertificateMonitoringParameters"
    },
    {
      "type": 3,
      "content": {
        "version": "KqlItem/1.0",
        "query": "ServerCertificateInventory_CL\n| where TimeGenerated {TimeRange}\n| summarize arg_max(TimeGenerated, *) by Computer, StoreLocation, StoreName, Thumbprint\n| where CollectionStatus == \"Success\"\n| summarize TotalCertificates = count(), Expired = countif(DaysUntilExpiry < 0), Expiring7Days = countif(DaysUntilExpiry between (0 .. 7)), Expiring30Days = countif(DaysUntilExpiry between (0 .. 30))",
        "size": 0,
        "title": "Certificate Health Summary",
        "timeContext": {
          "durationMs": 604800000
        },
        "visualization": "tiles",
        "tileSettings": {
          "showBorder": true,
          "titleContent": {
            "columnMatch": "Name",
            "formatter": 1
          },
          "leftContent": {
            "columnMatch": "Value",
            "formatter": 12
          }
        }
      },
      "name": "CertificateHealthSummary"
    },
    {
      "type": 3,
      "content": {
        "version": "KqlItem/1.0",
        "query": "ServerCertificateInventory_CL\n| where TimeGenerated {TimeRange}\n| summarize arg_max(TimeGenerated, *) by Computer, StoreLocation, StoreName, Thumbprint\n| where CollectionStatus == \"Success\"\n| where DaysUntilExpiry < 0\n| project Computer, StoreName, Subject, DnsNames, NotAfter, DaysUntilExpiry, IisBindings, Thumbprint\n| order by DaysUntilExpiry asc",
        "size": 0,
        "title": "Expired Certificates",
        "timeContext": {
          "durationMs": 604800000
        },
        "visualization": "table"
      },
      "name": "ExpiredCertificates"
    },
    {
      "type": 3,
      "content": {
        "version": "KqlItem/1.0",
        "query": "ServerCertificateInventory_CL\n| where TimeGenerated {TimeRange}\n| summarize arg_max(TimeGenerated, *) by Computer, StoreLocation, StoreName, Thumbprint\n| where CollectionStatus == \"Success\"\n| where DaysUntilExpiry between (0 .. 30)\n| project Computer, StoreName, Subject, DnsNames, NotAfter, DaysUntilExpiry, IisBindings, Thumbprint\n| order by DaysUntilExpiry asc",
        "size": 0,
        "title": "Certificates Expiring Within 30 Days",
        "timeContext": {
          "durationMs": 604800000
        },
        "visualization": "table"
      },
      "name": "CertificatesExpiring30Days"
    },
    {
      "type": 3,
      "content": {
        "version": "KqlItem/1.0",
        "query": "CollectorHealth_CL\n| where TimeGenerated {TimeRange}\n| where CollectorName == \"CertificateInventory\"\n| summarize arg_max(TimeGenerated, *) by Computer, CollectorName\n| project TimeGenerated, Computer, Status, RecordsCollected, RecordsSubmitted, DurationSeconds, ErrorCode, ErrorMessage, CollectorVersion\n| order by TimeGenerated desc",
        "size": 0,
        "title": "Certificate Collector Health",
        "timeContext": {
          "durationMs": 604800000
        },
        "visualization": "table"
      },
      "name": "CertificateCollectorHealth"
    }
  ],
  "isLocked": false,
  "fallbackResourceIds": [
    "/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.OperationalInsights/workspaces/<workspace-name>"
  ]
}
```

> The exact Workbook JSON format is best finalized by creating the workbook in the portal, validating it, then exporting from the Workbook Advanced Editor. Treat the JSON above as a source-controlled starter template.

---

# 11. Workbook Bicep Module

```bicep name=azure/modules/certificate-workbook.bicep
@description('Azure region for the Workbook resource.')
param location string

@description('Log Analytics Workspace resource ID.')
param workspaceResourceId string

@description('Workbook display name.')
param workbookDisplayName string = 'ERP Certificate Monitoring'

@description('Workbook serialized JSON definition.')
param workbookSerializedData string

resource certificateWorkbook 'Microsoft.Insights/workbooks@2022-04-01' = {
  name: guid(workspaceResourceId, workbookDisplayName)
  location: location
  kind: 'shared'
  properties: {
    displayName: workbookDisplayName
    serializedData: workbookSerializedData
    sourceId: workspaceResourceId
    category: 'workbook'
  }
}

output workbookResourceId string = certificateWorkbook.id
```

Add to `main.bicep`:

```bicep name=azure/main.bicep
@description('Enable deployment of the certificate-monitoring Workbook.')
param enableCertificateWorkbook bool = true

module certificateWorkbook './modules/certificate-workbook.bicep' = if (enableCertificateWorkbook) {
  name: 'deploy-certificate-workbook'
  params: {
    location: location
    workspaceResourceId: workspace.outputs.workspaceResourceId
    workbookDisplayName: 'ERP Certificate Monitoring'
    workbookSerializedData: loadTextContent('../workbooks/certificate-monitoring.workbook.json')
  }
  dependsOn: [
    tables
  ]
}
```

---

# 12. Certificate Monitoring Runbook

Create:

```text
docs/runbooks/CertificateExpiryResponse.md
```

```markdown name=docs/runbooks/CertificateExpiryResponse.md
# Certificate Expiry Response Runbook

## Trigger Conditions

- Certificate expired.
- Certificate expires within 7, 14, or 30 days.
- IIS-bound certificate missing a private key.
- Certificate collector has failed or has not reported.

## Initial Triage

1. Open the **ERP Certificate Monitoring** Workbook.
2. Identify:
   - Server name.
   - Certificate subject and SAN.
   - Thumbprint.
   - Expiry date.
   - IIS binding, if present.
   - Certificate owner/application owner.
3. Confirm whether the certificate is actively bound to an ERP or middleware endpoint.
4. Check whether a replacement certificate already exists in `Cert:\LocalMachine\WebHosting`.
5. Confirm certificate chain, private-key association, and validity dates.

## Resolution

1. Obtain or issue replacement certificate through the approved PKI/renewal process.
2. Import into the correct Local Machine certificate store.
3. Confirm private key is present.
4. Apply or update the IIS HTTPS binding.
5. Restart/reload application or IIS only if required by the application/vendor procedure.
6. Test the endpoint from an approved monitoring/client network.
7. Run the certificate collector manually.
8. Confirm the old alert resolves and the workbook shows the replacement certificate.

## Evidence Required

- Replacement certificate thumbprint.
- New certificate expiry date.
- Binding validation result.
- API/HTTPS validation result.
- Change record reference.
```

---

# 13. Phase 3 Test Plan

## Test 1 — Normal Certificate Inventory

1. Run the certificate collector.
2. Verify records in `ServerCertificateInventory_CL`.
3. Confirm the Workbook lists certificate subject, thumbprint, expiry, private-key state, and IIS binding.

## Test 2 — Expiring Certificate

Use an isolated test server and a short-lived test certificate.

1. Add a test certificate with expiry within 7 days.
2. Run the collector.
3. Validate the 7-day KQL query.
4. Confirm the Sev 2 alert triggers.
5. Confirm email arrives.
6. Confirm Teams card arrives after Logic App integration.
7. Replace/remove the certificate.
8. Re-run collector and confirm auto-mitigation/resolution.

## Test 3 — Missing Private Key

1. Import a public certificate without private key in an isolated test environment.
2. Bind only if safely possible and approved.
3. Run the collector.
4. Validate the missing-private-key query and alert.
5. Remove the test certificate.

## Test 4 — Collector Failure

1. Temporarily configure an invalid DCR immutable ID on a test server.
2. Run the collector.
3. Verify local log and spooled payload.
4. Verify `CollectorHealth_CL` reports failure when connectivity/configuration permits.
5. Restore valid configuration.
6. Replay spool and verify recovery.

## Test 5 — Missing Data

1. Disable the scheduled task on a non-production test server.
2. Wait beyond the 30-hour monitoring threshold or use a temporary short test threshold.
3. Verify stale/missing inventory query and alert.
4. Re-enable task and validate recovery.

---

# 14. Phase 3 Acceptance Criteria

| Area | Acceptance criterion |
|---|---|
| Inventory | Latest certificate inventory is visible in `ServerCertificateInventory_CL`. |
| IIS correlation | HTTPS bindings are shown for IIS-hosted certificates where enabled. |
| Expiry visibility | Workbook displays expired, 7-day, 14-day, and 30-day certificate risk views. |
| Private-key detection | IIS-bound certificates without private keys are identified. |
| Email alerting | Alert activation and resolution are sent to operations email. |
| Teams alerting | Logic App posts Common Alert Schema-based alert cards to approved Teams channel. |
| Alert suppression | Repeated notifications are controlled by mute duration and auto-mitigation. |
| Collector health | Collector failure and missing-data conditions are detectable. |
| Runbook | Certificate renewal/binding response procedure is approved and linked. |
| IaC | Alerts and Workbook are deployed through Bicep and version controlled. |

Microsoft recommends using Logic Apps for customized Azure Monitor alert notifications and enabling Common Alert Schema for a stable alert payload contract.[[1]](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-logic-apps)[[2]](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-common-schema) Workbooks and Action Groups can be deployed and maintained through Bicep as infrastructure-as-code.[[3]](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/scenarios-monitoring)

### Want me to…
<!-- copilot:suggestions -->
1. Continue to Phase 4 with disk and ERP log-folder KQL, alerts, and Workbook dashboard implementation.
2. Provide the Bicep/Logic App definition for Teams Adaptive Card notifications using the Common Alert Schema.
3. Hand this certificate-monitoring implementation off to the coding agent to open a pull request in a repository you provide.
