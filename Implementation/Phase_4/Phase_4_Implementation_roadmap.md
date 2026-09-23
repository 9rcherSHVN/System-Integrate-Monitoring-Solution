# Phase 4 — Disk Capacity and ERP Log-Folder Monitoring

## Important Design Correction

The Phase 1 and Phase 2 implementation currently provides a **custom ERP log-folder collector** and the `ErpFolderMetrics_CL` table. It does **not** yet deploy a `WindowsDiskMetrics_CL` custom table or custom logical-disk collector.

For the pilot, the recommended design is:

| Requirement | Collection method | Reason |
|---|---|---|
| Windows logical-drive free space, disk latency, queue length | **Azure Monitor Agent (AMA) performance counters** | Native, efficient, and more appropriate than custom PowerShell for standard Windows disk metrics. |
| ERP log-directory total size, growth, file age, retention risk | **Custom PowerShell collector** → `ErpFolderMetrics_CL` | Requires application-specific directory scanning and aggregation. |
| Disk alert correlation | Azure Monitor Workbook / KQL | Correlates a filling ERP disk with the ERP log directory responsible for growth. |

This means Phase 4 adds an AMA DCR for native disk counters, KQL, scheduled query alerts, and a combined Disk/ERP Folder Workbook.

---

# 1. Phase 4 Architecture

```text
ERP Application Server
    │
    ├── Azure Monitor Agent
    │      └── LogicalDisk performance counters
    │             ├── % Free Space
    │             ├── Free Megabytes
    │             ├── Avg. Disk sec/Read
    │             ├── Avg. Disk sec/Write
    │             └── Current Disk Queue Length
    │
    └── Scheduled PowerShell folder collector
           └── ERP log directory scan
                  ├── Total size
                  ├── File count
                  ├── File age / retention
                  ├── Access errors
                  └── Scan duration
                        │
                        ▼
              ErpFolderMetrics_CL
                        │
                        ▼
Azure Monitor / Log Analytics
    ├── Perf or AMA performance table
    ├── ErpFolderMetrics_CL
    ├── CollectorHealth_CL
    ├── KQL
    ├── Workbook
    └── Scheduled-query alerts
            ├── Outlook/email
            └── Teams via Logic App
```

---

# 2. Native AMA Disk Performance Counter DCR

Create a dedicated AMA DCR for ERP server disk monitoring. This is separate from the custom direct-ingestion DCRs used by PowerShell collectors.

```text
dcr-erpmon-windows-disk-pilot-001
```

## 2.1 Counters

| Windows counter | Purpose |
|---|---|
| `\LogicalDisk(*)\% Free Space` | Relative disk capacity risk. |
| `\LogicalDisk(*)\Free Megabytes` | Absolute available capacity. |
| `\LogicalDisk(*)\Avg. Disk sec/Read` | Disk read latency. |
| `\LogicalDisk(*)\Avg. Disk sec/Write` | Disk write latency. |
| `\LogicalDisk(*)\Current Disk Queue Length` | Disk queue/saturation indication. |
| `\LogicalDisk(*)\Disk Transfers/sec` | Disk activity rate. |

A five-minute sample interval is suitable for the pilot:

```text
samplingFrequencyInSeconds = 300
```

## 2.2 AMA Disk DCR Bicep Module

```bicep name=azure/modules/windows-disk-performance-dcr.bicep
@description('Azure region for the Data Collection Rule.')
param location string

@description('Name of the Windows disk-performance DCR.')
param dcrName string

@description('Resource ID of the target Log Analytics Workspace.')
param workspaceResourceId string

@description('Deployment environment.')
param environment string

resource diskPerformanceDcr 'Microsoft.Insights/dataCollectionRules@2023-03-11' = {
  name: dcrName
  location: location
  kind: 'Windows'
  tags: {
    Application: 'ERP-Monitoring'
    Environment: environment
    ManagedBy: 'Bicep'
    DataClassification: 'Internal'
    Collector: 'AzureMonitorAgent'
    MonitoringDomain: 'WindowsDisk'
  }
  properties: {
    dataSources: {
      performanceCounters: [
        {
          name: 'erpLogicalDiskCounters'
          streams: [
            'Microsoft-Perf'
          ]
          samplingFrequencyInSeconds: 300
          counterSpecifiers: [
            '\\LogicalDisk(*)\\% Free Space'
            '\\LogicalDisk(*)\\Free Megabytes'
            '\\LogicalDisk(*)\\Avg. Disk sec/Read'
            '\\LogicalDisk(*)\\Avg. Disk sec/Write'
            '\\LogicalDisk(*)\\Current Disk Queue Length'
            '\\LogicalDisk(*)\\Disk Transfers/sec'
          ]
        }
      ]
    }
    destinations: {
      logAnalytics: [
        {
          name: 'erpMonitoringWorkspace'
          workspaceResourceId: workspaceResourceId
        }
      ]
    }
    dataFlows: [
      {
        streams: [
          'Microsoft-Perf'
        ]
        destinations: [
          'erpMonitoringWorkspace'
        ]
      }
    ]
  }
}

output diskPerformanceDcrResourceId string = diskPerformanceDcr.id
```

## 2.3 DCR Association to Azure Arc Server

A DCR is not effective until it is associated with the Azure Arc-enabled ERP server.

The following module can be deployed at the Azure Arc machine resource scope.

```bicep name=azure/modules/arc-dcr-association.bicep
@description('Resource ID of the Azure Arc-enabled server.')
param arcMachineResourceId string

@description('Resource ID of the Data Collection Rule.')
param dataCollectionRuleResourceId string

@description('Association name.')
param associationName string

resource arcMachine 'Microsoft.HybridCompute/machines@2023-10-03-preview' existing = {
  scope: resourceGroup(
    split(arcMachineResourceId, '/')[4],
    split(arcMachineResourceId, '/')[8]
  )
  name: last(split(arcMachineResourceId, '/'))
}

resource association 'Microsoft.Insights/dataCollectionRuleAssociations@2023-03-11' = {
  scope: arcMachine
  name: associationName
  properties: {
    dataCollectionRuleId: dataCollectionRuleResourceId
  }
}

output associationResourceId string = association.id
```

> Validate the Azure Arc resource group, API version, and deployment scope in the target tenant before production rollout. If the Arc machine is in a different resource group from the monitoring resources, deploy the association module using a cross-resource-group scope.

---

# 3. Add Disk Monitoring to `main.bicep`

Add the following parameters.

```bicep name=azure/main.bicep
@description('Enable AMA logical-disk performance counter collection.')
param enableWindowsDiskMonitoring bool = true

@description('Windows disk performance DCR name.')
param windowsDiskPerformanceDcrName string = 'dcr-erpmon-windows-disk-pilot-001'

@description('Resource IDs of Azure Arc ERP servers receiving the disk DCR.')
param diskMonitoringArcMachineResourceIds array = []

@description('Enable disk and ERP log-folder alert rules.')
param enableDiskAndFolderAlerts bool = true

@description('Disk and folder alert resource-name prefix.')
param diskAlertNamePrefix string = 'sqra-erpmon-disk'
```

Add the DCR deployment:

```bicep name=azure/main.bicep
module windowsDiskPerformanceDcr './modules/windows-disk-performance-dcr.bicep' = if (enableWindowsDiskMonitoring) {
  name: 'deploy-windows-disk-performance-dcr'
  params: {
    location: location
    dcrName: windowsDiskPerformanceDcrName
    workspaceResourceId: workspace.outputs.workspaceResourceId
    environment: environment
  }
  dependsOn: [
    workspace
  ]
}
```

Associate it with each Arc-enabled ERP application server:

```bicep name=azure/main.bicep
module windowsDiskDcrAssociations './modules/arc-dcr-association.bicep' = [
  for arcMachineResourceId in diskMonitoringArcMachineResourceIds: if (enableWindowsDiskMonitoring) {
    name: 'associate-windows-disk-dcr-${uniqueString(arcMachineResourceId)}'
    params: {
      arcMachineResourceId: arcMachineResourceId
      dataCollectionRuleResourceId: windowsDiskPerformanceDcr.outputs.diskPerformanceDcrResourceId
      associationName: 'assoc-erpmon-windows-disk'
    }
    dependsOn: [
      windowsDiskPerformanceDcr
    ]
  }
]
```

Add pilot parameter values:

```bicep name=azure/parameters/pilot.bicepparam
param enableWindowsDiskMonitoring = true

param windowsDiskPerformanceDcrName = 'dcr-erpmon-windows-disk-pilot-001'

param diskMonitoringArcMachineResourceIds = [
  '/subscriptions/<subscription-id>/resourceGroups/<arc-resource-group>/providers/Microsoft.HybridCompute/machines/ERP-APP-01'
]

param enableDiskAndFolderAlerts = true

param diskAlertNamePrefix = 'sqra-erpmon-disk'
```

---

# 4. Disk and Folder KQL Query Set

Create the following folder structure:

```text
kql/
└── disk/
    ├── CurrentDiskCapacity.kql
    ├── DiskCapacityWarning.kql
    ├── DiskCapacityCritical.kql
    ├── DiskLatency.kql
    ├── CurrentErpFolderMetrics.kql
    ├── ErpFolderGrowth24Hours.kql
    ├── ErpRetentionRisk.kql
    ├── ErpFolderCollectionFailure.kql
    └── MissingErpFolderMetrics.kql
```

---

## 4.1 Current Disk Capacity

This query retrieves recent logical-disk free-space telemetry from AMA-collected performance counters.

```kusto name=kql/disk/CurrentDiskCapacity.kql
Perf
| where TimeGenerated > ago(30m)
| where ObjectName == "LogicalDisk"
| where CounterName in ("% Free Space", "Free Megabytes")
| where InstanceName != "_Total"
| summarize arg_max(TimeGenerated, CounterValue) by Computer, InstanceName, CounterName
| evaluate pivot(CounterName, max(CounterValue))
| extend FreePercent = round(todouble(["% Free Space"]), 2)
| extend FreeMB = tolong(["Free Megabytes"])
| extend FreeGB = round(todouble(FreeMB) / 1024, 2)
| project
    TimeGenerated,
    Computer,
    Drive = InstanceName,
    FreePercent,
    FreeMB,
    FreeGB
| order by FreePercent asc, FreeMB asc
```

---

## 4.2 Disk Capacity Warning

A warning should use both percentage and absolute thresholds. The following example triggers when either:

- Available capacity is below 20%; or
- Available capacity is below 50 GB.

Adjust thresholds by drive role after baseline analysis.

```kusto name=kql/disk/DiskCapacityWarning.kql
Perf
| where TimeGenerated > ago(30m)
| where ObjectName == "LogicalDisk"
| where CounterName in ("% Free Space", "Free Megabytes")
| where InstanceName != "_Total"
| summarize arg_max(TimeGenerated, CounterValue) by Computer, InstanceName, CounterName
| evaluate pivot(CounterName, max(CounterValue))
| extend FreePercent = round(todouble(["% Free Space"]), 2)
| extend FreeMB = tolong(["Free Megabytes"])
| extend FreeGB = round(todouble(FreeMB) / 1024, 2)
| where FreePercent < 20 or FreeGB < 50
| project
    Computer,
    Drive = InstanceName,
    FreePercent,
    FreeGB,
    AlertLevel = "Warning"
| order by FreePercent asc, FreeGB asc
```

---

## 4.3 Disk Capacity Critical

```kusto name=kql/disk/DiskCapacityCritical.kql
Perf
| where TimeGenerated > ago(30m)
| where ObjectName == "LogicalDisk"
| where CounterName in ("% Free Space", "Free Megabytes")
| where InstanceName != "_Total"
| summarize arg_max(TimeGenerated, CounterValue) by Computer, InstanceName, CounterName
| evaluate pivot(CounterName, max(CounterValue))
| extend FreePercent = round(todouble(["% Free Space"]), 2)
| extend FreeMB = tolong(["Free Megabytes"])
| extend FreeGB = round(todouble(FreeMB) / 1024, 2)
| where FreePercent < 10 or FreeGB < 20
| project
    Computer,
    Drive = InstanceName,
    FreePercent,
    FreeGB,
    AlertLevel = "Critical"
| order by FreePercent asc, FreeGB asc
```

> For SQL log, backup, `tempdb`, or high-volume ERP log volumes, production thresholds should be drive-specific. For example, a SQL transaction-log volume might need a critical absolute-free-space threshold of 100 GB rather than 20 GB.

---

## 4.4 Disk Latency

Initial latency thresholds should be tuned after observing normal workload. The following query treats:

- Read latency above 25 ms as warning.
- Write latency above 25 ms as warning.

```kusto name=kql/disk/DiskLatency.kql
Perf
| where TimeGenerated > ago(30m)
| where ObjectName == "LogicalDisk"
| where CounterName in ("Avg. Disk sec/Read", "Avg. Disk sec/Write", "Current Disk Queue Length")
| where InstanceName != "_Total"
| summarize AverageValue = avg(CounterValue) by Computer, InstanceName, CounterName
| evaluate pivot(CounterName, max(AverageValue))
| extend ReadLatencyMs = round(todouble(["Avg. Disk sec/Read"]) * 1000, 2)
| extend WriteLatencyMs = round(todouble(["Avg. Disk sec/Write"]) * 1000, 2)
| extend QueueLength = round(todouble(["Current Disk Queue Length"]), 2)
| project
    Computer,
    Drive = InstanceName,
    ReadLatencyMs,
    WriteLatencyMs,
    QueueLength
| order by WriteLatencyMs desc, ReadLatencyMs desc
```

---

## 4.5 Current ERP Folder Metrics

```kusto name=kql/disk/CurrentErpFolderMetrics.kql
ErpFolderMetrics_CL
| where TimeGenerated > ago(6h)
| summarize arg_max(TimeGenerated, *) by Computer, Application, FolderPath
| project
    TimeGenerated,
    Computer,
    Environment,
    Application,
    FolderPath,
    TotalGB,
    FileCount,
    OldestFileUtc,
    NewestFileUtc,
    FilesOlderThanRetention,
    RetentionDays,
    AccessErrorCount,
    ScanDurationSeconds,
    CollectionStatus,
    CollectorVersion
| order by TotalGB desc
```

---

## 4.6 ERP Folder Growth in 24 Hours

This query compares the newest record with the nearest record from approximately 24 hours before.

```kusto name=kql/disk/ErpFolderGrowth24Hours.kql
let CurrentMetrics =
    ErpFolderMetrics_CL
    | where TimeGenerated > ago(2h)
    | summarize arg_max(TimeGenerated, TotalBytes, TotalGB, FileCount, CollectionStatus)
        by Computer, Application, FolderPath;
let PreviousMetrics =
    ErpFolderMetrics_CL
    | where TimeGenerated between (ago(26h) .. ago(22h))
    | summarize arg_max(TimeGenerated, TotalBytes, TotalGB, FileCount)
        by Computer, Application, FolderPath;
CurrentMetrics
| join kind=leftouter PreviousMetrics
    on Computer, Application, FolderPath
| extend GrowthBytes24h = TotalBytes - TotalBytes1
| extend GrowthGB24h = round(todouble(GrowthBytes24h) / 1024 / 1024 / 1024, 2)
| extend FileGrowth24h = FileCount - FileCount1
| project
    Computer,
    Application,
    FolderPath,
    TotalGB,
    GrowthGB24h,
    FileCount,
    FileGrowth24h,
    CollectionStatus
| order by GrowthGB24h desc
```

---

## 4.7 ERP Folder Retention Risk

```kusto name=kql/disk/ErpRetentionRisk.kql
ErpFolderMetrics_CL
| where TimeGenerated > ago(6h)
| summarize arg_max(TimeGenerated, *) by Computer, Application, FolderPath
| where CollectionStatus in ("Success", "PartialSuccess")
| where FilesOlderThanRetention > 0
| project
    Computer,
    Application,
    FolderPath,
    TotalGB,
    FileCount,
    FilesOlderThanRetention,
    RetentionDays,
    OldestFileUtc,
    AccessErrorCount,
    CollectionStatus
| order by FilesOlderThanRetention desc, TotalGB desc
```

---

## 4.8 ERP Folder Collector Failure

```kusto name=kql/disk/ErpFolderCollectionFailure.kql
CollectorHealth_CL
| where TimeGenerated > ago(2h)
| where CollectorName == "ErpFolderMetrics"
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

## 4.9 Missing ERP Folder Metrics

For a collector scheduled every hour, treat no record for two hours as a monitoring failure.

```kusto name=kql/disk/MissingErpFolderMetrics.kql
let ExpectedErpFolderCollectorHosts = datatable(Computer:string)
[
    "ERP-APP-01"
];
ExpectedErpFolderCollectorHosts
| join kind=leftouter (
    ErpFolderMetrics_CL
    | where TimeGenerated > ago(2h)
    | summarize LastFolderMetric = max(TimeGenerated) by Computer
) on Computer
| where isempty(LastFolderMetric) or LastFolderMetric < ago(2h)
| project
    Computer,
    LastFolderMetric,
    Status = "ERP folder metrics missing or stale"
```

---

# 5. Disk and ERP Folder Alert Design

| Alert | Source | Severity | Evaluation frequency | Window |
|---|---|---:|---:|---:|
| Disk capacity critical | `Perf` | Sev 1 | 5 minutes | 15 minutes |
| Disk capacity warning | `Perf` | Sev 2 | 15 minutes | 30 minutes |
| Disk latency sustained | `Perf` | Sev 2 | 15 minutes | 30 minutes |
| ERP folder rapid growth | `ErpFolderMetrics_CL` | Sev 2 | 60 minutes | 26 hours |
| ERP retention accumulation | `ErpFolderMetrics_CL` | Sev 3 | 6 hours | 6 hours |
| ERP folder collector failure | `CollectorHealth_CL` | Sev 2 | 30 minutes | 2 hours |
| ERP folder metrics missing | `ErpFolderMetrics_CL` | Sev 2 | 30 minutes | 2 hours |

---

# 6. Disk and Folder Alert Bicep Module

Create:

```text
azure/modules/disk-folder-alerts.bicep
```

```bicep name=azure/modules/disk-folder-alerts.bicep
@description('Azure region for scheduled query rules.')
param location string

@description('Log Analytics Workspace resource ID.')
param workspaceResourceId string

@description('Action Group resource ID.')
param actionGroupResourceId string

@description('Deployment environment.')
param environment string

@description('Prefix used for disk/folder alert resource names.')
param alertNamePrefix string = 'sqra-erpmon-disk'

@description('ERP servers expected to submit folder metrics.')
param expectedErpFolderCollectorHosts array = []

@description('Warning free space percentage threshold.')
param diskWarningFreePercent int = 20

@description('Critical free space percentage threshold.')
param diskCriticalFreePercent int = 10

@description('Warning absolute free capacity threshold in GB.')
param diskWarningFreeGb int = 50

@description('Critical absolute free capacity threshold in GB.')
param diskCriticalFreeGb int = 20

@description('Read or write latency threshold in milliseconds.')
param diskLatencyThresholdMs int = 25

@description('ERP log folder growth threshold in GB over 24 hours.')
param erpFolderGrowthThresholdGb int = 5

var expectedHostsKql = '''
datatable(Computer:string)
[
${join([for host in expectedErpFolderCollectorHosts: '    "${host}"'], ',\n')}
]
'''

var diskCapacityCriticalQuery = '''
Perf
| where TimeGenerated > ago(15m)
| where ObjectName == "LogicalDisk"
| where CounterName in ("% Free Space", "Free Megabytes")
| where InstanceName != "_Total"
| summarize arg_max(TimeGenerated, CounterValue) by Computer, InstanceName, CounterName
| evaluate pivot(CounterName, max(CounterValue))
| extend FreePercent = round(todouble(["% Free Space"]), 2)
| extend FreeGB = round(todouble(["Free Megabytes"]) / 1024, 2)
| where FreePercent < ${diskCriticalFreePercent} or FreeGB < ${diskCriticalFreeGb}
| project Computer, Drive = InstanceName, FreePercent, FreeGB
'''

var diskCapacityWarningQuery = '''
Perf
| where TimeGenerated > ago(30m)
| where ObjectName == "LogicalDisk"
| where CounterName in ("% Free Space", "Free Megabytes")
| where InstanceName != "_Total"
| summarize arg_max(TimeGenerated, CounterValue) by Computer, InstanceName, CounterName
| evaluate pivot(CounterName, max(CounterValue))
| extend FreePercent = round(todouble(["% Free Space"]), 2)
| extend FreeGB = round(todouble(["Free Megabytes"]) / 1024, 2)
| where FreePercent >= ${diskCriticalFreePercent} and FreeGB >= ${diskCriticalFreeGb}
| where FreePercent < ${diskWarningFreePercent} or FreeGB < ${diskWarningFreeGb}
| project Computer, Drive = InstanceName, FreePercent, FreeGB
'''

var diskLatencyQuery = '''
Perf
| where TimeGenerated > ago(30m)
| where ObjectName == "LogicalDisk"
| where CounterName in ("Avg. Disk sec/Read", "Avg. Disk sec/Write")
| where InstanceName != "_Total"
| summarize AverageValue = avg(CounterValue) by Computer, InstanceName, CounterName
| evaluate pivot(CounterName, max(AverageValue))
| extend ReadLatencyMs = round(todouble(["Avg. Disk sec/Read"]) * 1000, 2)
| extend WriteLatencyMs = round(todouble(["Avg. Disk sec/Write"]) * 1000, 2)
| where ReadLatencyMs > ${diskLatencyThresholdMs} or WriteLatencyMs > ${diskLatencyThresholdMs}
| project Computer, Drive = InstanceName, ReadLatencyMs, WriteLatencyMs
'''

var erpFolderGrowthQuery = '''
let CurrentMetrics =
    ErpFolderMetrics_CL
    | where TimeGenerated > ago(2h)
    | summarize arg_max(TimeGenerated, TotalBytes, TotalGB, FileCount, CollectionStatus)
        by Computer, Application, FolderPath;
let PreviousMetrics =
    ErpFolderMetrics_CL
    | where TimeGenerated between (ago(26h) .. ago(22h))
    | summarize arg_max(TimeGenerated, TotalBytes, TotalGB, FileCount)
        by Computer, Application, FolderPath;
CurrentMetrics
| join kind=leftouter PreviousMetrics
    on Computer, Application, FolderPath
| extend GrowthBytes24h = TotalBytes - TotalBytes1
| extend GrowthGB24h = round(todouble(GrowthBytes24h) / 1024 / 1024 / 1024, 2)
| where GrowthGB24h >= ${erpFolderGrowthThresholdGb}
| project Computer, Application, FolderPath, TotalGB, GrowthGB24h, FileCount, CollectionStatus
'''

var erpRetentionRiskQuery = '''
ErpFolderMetrics_CL
| where TimeGenerated > ago(6h)
| summarize arg_max(TimeGenerated, *) by Computer, Application, FolderPath
| where CollectionStatus in ("Success", "PartialSuccess")
| where FilesOlderThanRetention > 0
| project Computer, Application, FolderPath, TotalGB, FileCount, FilesOlderThanRetention, RetentionDays, OldestFileUtc, AccessErrorCount
'''

var erpFolderCollectorFailureQuery = '''
CollectorHealth_CL
| where TimeGenerated > ago(2h)
| where CollectorName == "ErpFolderMetrics"
| summarize arg_max(TimeGenerated, *) by Computer, CollectorName
| where Status != "Success"
| project Computer, Status, TargetScope, RecordsCollected, RecordsSubmitted, DurationSeconds, ErrorCode, ErrorMessage
'''

var erpFolderMetricsMissingQuery = '''
let ExpectedErpFolderCollectorHosts = ${expectedHostsKql};
ExpectedErpFolderCollectorHosts
| join kind=leftouter (
    ErpFolderMetrics_CL
    | where TimeGenerated > ago(2h)
    | summarize LastFolderMetric = max(TimeGenerated) by Computer
) on Computer
| where isempty(LastFolderMetric) or LastFolderMetric < ago(2h)
| project Computer, LastFolderMetric, Status = "ERP folder metrics missing or stale"
'''

resource diskCapacityCriticalAlert 'Microsoft.Insights/scheduledQueryRules@2023-12-01' = {
  name: '${alertNamePrefix}-capacity-critical-${environment}-001'
  location: location
  tags: {
    Application: 'ERP-Monitoring'
    Environment: environment
    ManagedBy: 'Bicep'
    AlertDomain: 'DiskCapacity'
    Severity: 'Sev1'
  }
  properties: {
    displayName: 'ERP Server Disk Capacity Critical'
    description: 'A monitored ERP server volume is below the configured critical free-space threshold.'
    enabled: true
    scopes: [
      workspaceResourceId
    ]
    severity: 1
    evaluationFrequency: 'PT5M'
    windowSize: 'PT15M'
    autoMitigate: true
    muteActionsDuration: 'PT1H'
    criteria: {
      allOf: [
        {
          query: diskCapacityCriticalQuery
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
        Service = 'ERP Disk Monitoring'
        Runbook = 'Identify volume owner, stop uncontrolled growth, archive or clear approved data, and extend capacity if required.'
      }
    }
  }
}

resource diskCapacityWarningAlert 'Microsoft.Insights/scheduledQueryRules@2023-12-01' = {
  name: '${alertNamePrefix}-capacity-warning-${environment}-001'
  location: location
  tags: {
    Application: 'ERP-Monitoring'
    Environment: environment
    ManagedBy: 'Bicep'
    AlertDomain: 'DiskCapacity'
    Severity: 'Sev2'
  }
  properties: {
    displayName: 'ERP Server Disk Capacity Warning'
    description: 'A monitored ERP server volume is below the configured warning free-space threshold.'
    enabled: true
    scopes: [
      workspaceResourceId
    ]
    severity: 2
    evaluationFrequency: 'PT15M'
    windowSize: 'PT30M'
    autoMitigate: true
    muteActionsDuration: 'PT6H'
    criteria: {
      allOf: [
        {
          query: diskCapacityWarningQuery
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
        Service = 'ERP Disk Monitoring'
        Runbook = 'Review disk and ERP log growth trend; plan cleanup, retention, archival, or capacity expansion.'
      }
    }
  }
}

resource diskLatencyAlert 'Microsoft.Insights/scheduledQueryRules@2023-12-01' = {
  name: '${alertNamePrefix}-latency-${environment}-001'
  location: location
  tags: {
    Application: 'ERP-Monitoring'
    Environment: environment
    ManagedBy: 'Bicep'
    AlertDomain: 'DiskPerformance'
    Severity: 'Sev2'
  }
  properties: {
    displayName: 'ERP Server Disk Latency High'
    description: 'A monitored ERP server volume has sustained read or write latency above the initial threshold.'
    enabled: true
    scopes: [
      workspaceResourceId
    ]
    severity: 2
    evaluationFrequency: 'PT15M'
    windowSize: 'PT30M'
    autoMitigate: true
    muteActionsDuration: 'PT6H'
    criteria: {
      allOf: [
        {
          query: diskLatencyQuery
          timeAggregation: 'Count'
          operator: 'GreaterThan'
          threshold: 0
          failingPeriods: {
            numberOfEvaluationPeriods: 2
            minFailingPeriodsToAlert: 2
          }
        }
      ]
    }
    actions: {
      actionGroups: [
        actionGroupResourceId
      ]
      customProperties: {
        Service = 'ERP Disk Monitoring'
        Runbook = 'Correlate disk latency with ERP response time, CPU, memory, antivirus activity, backup jobs, and SQL I/O where applicable.'
      }
    }
  }
}

resource erpFolderGrowthAlert 'Microsoft.Insights/scheduledQueryRules@2023-12-01' = {
  name: '${alertNamePrefix}-foldergrowth-${environment}-001'
  location: location
  tags: {
    Application: 'ERP-Monitoring'
    Environment: environment
    ManagedBy: 'Bicep'
    AlertDomain: 'ErpLogGrowth'
    Severity: 'Sev2'
  }
  properties: {
    displayName: 'ERP Log Folder Growth High'
    description: 'An approved ERP log directory has exceeded the configured 24-hour growth threshold.'
    enabled: true
    scopes: [
      workspaceResourceId
    ]
    severity: 2
    evaluationFrequency: 'PT1H'
    windowSize: 'P1D'
    autoMitigate: true
    muteActionsDuration: 'PT12H'
    criteria: {
      allOf: [
        {
          query: erpFolderGrowthQuery
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
        Service = 'ERP Log Monitoring'
        Runbook = 'Identify abnormal ERP log production, preserve required evidence, apply approved retention/archival process, and validate free disk capacity.'
      }
    }
  }
}

resource erpRetentionRiskAlert 'Microsoft.Insights/scheduledQueryRules@2023-12-01' = {
  name: '${alertNamePrefix}-retention-${environment}-001'
  location: location
  tags: {
    Application: 'ERP-Monitoring'
    Environment: environment
    ManagedBy: 'Bicep'
    AlertDomain: 'ErpLogRetention'
    Severity: 'Sev3'
  }
  properties: {
    displayName: 'ERP Log Retention Accumulation'
    description: 'An ERP log directory contains files older than the configured retention policy.'
    enabled: true
    scopes: [
      workspaceResourceId
    ]
    severity: 3
    evaluationFrequency: 'PT6H'
    windowSize: 'PT6H'
    autoMitigate: true
    muteActionsDuration: 'P1D'
    criteria: {
      allOf: [
        {
          query: erpRetentionRiskQuery
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
        Service = 'ERP Log Monitoring'
        Runbook = 'Review retention policy, legal hold requirements, archive status, and approved log cleanup procedure.'
      }
    }
  }
}

resource erpFolderCollectorFailureAlert 'Microsoft.Insights/scheduledQueryRules@2023-12-01' = {
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
    displayName: 'ERP Folder Metrics Collector Failed'
    description: 'The custom ERP folder metrics collector reported failure or partial failure.'
    enabled: true
    scopes: [
      workspaceResourceId
    ]
    severity: 2
    evaluationFrequency: 'PT30M'
    windowSize: 'PT2H'
    autoMitigate: true
    muteActionsDuration: 'PT2H'
    criteria: {
      allOf: [
        {
          query: erpFolderCollectorFailureQuery
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
        Service = 'ERP Monitoring Health'
        Runbook = 'Review scheduled-task history, local collector log, spool directory, Arc managed identity, DCR authorization, and ERP log-folder permissions.'
      }
    }
  }
}

resource erpFolderMetricsMissingAlert 'Microsoft.Insights/scheduledQueryRules@2023-12-01' = {
  name: '${alertNamePrefix}-metricsmissing-${environment}-001'
  location: location
  tags: {
    Application: 'ERP-Monitoring'
    Environment: environment
    ManagedBy: 'Bicep'
    AlertDomain: 'MonitoringHealth'
    Severity: 'Sev2'
  }
  properties: {
    displayName: 'ERP Folder Metrics Missing or Stale'
    description: 'An expected ERP server has not submitted folder metrics within the expected collection window.'
    enabled: true
    scopes: [
      workspaceResourceId
    ]
    severity: 2
    evaluationFrequency: 'PT30M'
    windowSize: 'PT2H'
    autoMitigate: true
    muteActionsDuration: 'PT2H'
    criteria: {
      allOf: [
        {
          query: erpFolderMetricsMissingQuery
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
        Service = 'ERP Monitoring Health'
        Runbook = 'Confirm collector task status, run-as account access, Azure Arc identity token availability, and Log Analytics ingestion status.'
      }
    }
  }
}

output diskCapacityCriticalAlertResourceId string = diskCapacityCriticalAlert.id
output diskCapacityWarningAlertResourceId string = diskCapacityWarningAlert.id
output diskLatencyAlertResourceId string = diskLatencyAlert.id
output erpFolderGrowthAlertResourceId string = erpFolderGrowthAlert.id
output erpRetentionRiskAlertResourceId string = erpRetentionRiskAlert.id
output erpFolderCollectorFailureAlertResourceId string = erpFolderCollectorFailureAlert.id
output erpFolderMetricsMissingAlertResourceId string = erpFolderMetricsMissingAlert.id
```

---

# 7. Add Disk and Folder Alerts to `main.bicep`

```bicep name=azure/main.bicep
@description('ERP server names expected to submit custom folder metrics.')
param expectedErpFolderCollectorHosts array = []

@description('Warning disk free-space percentage.')
param diskWarningFreePercent int = 20

@description('Critical disk free-space percentage.')
param diskCriticalFreePercent int = 10

@description('Warning disk free-space threshold in GB.')
param diskWarningFreeGb int = 50

@description('Critical disk free-space threshold in GB.')
param diskCriticalFreeGb int = 20

@description('Initial disk read/write latency threshold in milliseconds.')
param diskLatencyThresholdMs int = 25

@description('ERP log-folder growth threshold in GB over 24 hours.')
param erpFolderGrowthThresholdGb int = 5
```

Add the module:

```bicep name=azure/main.bicep
module diskAndFolderAlerts './modules/disk-folder-alerts.bicep' = if (enableDiskAndFolderAlerts) {
  name: 'deploy-disk-folder-alerts'
  params: {
    location: location
    workspaceResourceId: workspace.outputs.workspaceResourceId
    actionGroupResourceId: actionGroup.outputs.actionGroupResourceId
    environment: environment
    alertNamePrefix: diskAlertNamePrefix
    expectedErpFolderCollectorHosts: expectedErpFolderCollectorHosts
    diskWarningFreePercent: diskWarningFreePercent
    diskCriticalFreePercent: diskCriticalFreePercent
    diskWarningFreeGb: diskWarningFreeGb
    diskCriticalFreeGb: diskCriticalFreeGb
    diskLatencyThresholdMs: diskLatencyThresholdMs
    erpFolderGrowthThresholdGb: erpFolderGrowthThresholdGb
  }
  dependsOn: [
    tables
    actionGroup
    windowsDiskPerformanceDcr
  ]
}
```

Pilot parameters:

```bicep name=azure/parameters/pilot.bicepparam
param expectedErpFolderCollectorHosts = [
  'ERP-APP-01'
]

param diskWarningFreePercent = 20
param diskCriticalFreePercent = 10

param diskWarningFreeGb = 50
param diskCriticalFreeGb = 20

param diskLatencyThresholdMs = 25

param erpFolderGrowthThresholdGb = 5
```

---

# 8. Disk and ERP Folder Workbook

Create:

```text
workbooks/
└── disk-erp-folder-monitoring.workbook.json
```

The Workbook should contain:

| Section | Primary data source | Purpose |
|---|---|---|
| Disk capacity heat map | `Perf` | Identify low free-space volumes. |
| Critical volumes | `Perf` | Identify immediate outage risk. |
| Disk latency | `Perf` | Identify read/write bottlenecks. |
| ERP folder size | `ErpFolderMetrics_CL` | Identify the largest ERP log locations. |
| ERP log growth | `ErpFolderMetrics_CL` | Detect abnormal growth in 24 hours. |
| Retention accumulation | `ErpFolderMetrics_CL` | Identify old files not cleared/archived. |
| Folder scan quality | `ErpFolderMetrics_CL` | Show timeouts/access errors. |
| Collector health | `CollectorHealth_CL` | Confirm telemetry remains trustworthy. |

## Workbook Starter Definition

```json name=workbooks/disk-erp-folder-monitoring.workbook.json
{
  "version": "Notebook/1.0",
  "items": [
    {
      "type": 1,
      "content": {
        "json": "# ERP Disk and Log Folder Monitoring\nThis workbook correlates Windows logical-disk capacity and performance with ERP log-directory size, growth, retention accumulation, and collector health."
      },
      "name": "DiskWorkbookTitle"
    },
    {
      "type": 9,
      "content": {
        "version": "KqlParameterItem/1.0",
        "parameters": [
          {
            "id": "ab7ef8ab-7c2d-42b7-9f93-5c98582142af",
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
      "name": "DiskWorkbookParameters"
    },
    {
      "type": 3,
      "content": {
        "version": "KqlItem/1.0",
        "query": "Perf\n| where TimeGenerated {TimeRange}\n| where ObjectName == \"LogicalDisk\"\n| where CounterName in (\"% Free Space\", \"Free Megabytes\")\n| where InstanceName != \"_Total\"\n| summarize arg_max(TimeGenerated, CounterValue) by Computer, InstanceName, CounterName\n| evaluate pivot(CounterName, max(CounterValue))\n| extend FreePercent = round(todouble([\"% Free Space\"]), 2)\n| extend FreeGB = round(todouble([\"Free Megabytes\"]) / 1024, 2)\n| project Computer, Drive = InstanceName, FreePercent, FreeGB\n| order by FreePercent asc, FreeGB asc",
        "size": 0,
        "title": "Current Disk Capacity",
        "timeContext": {
          "durationMs": 604800000
        },
        "visualization": "table"
      },
      "name": "CurrentDiskCapacity"
    },
    {
      "type": 3,
      "content": {
        "version": "KqlItem/1.0",
        "query": "ErpFolderMetrics_CL\n| where TimeGenerated {TimeRange}\n| summarize arg_max(TimeGenerated, *) by Computer, Application, FolderPath\n| project Computer, Application, FolderPath, TotalGB, FileCount, FilesOlderThanRetention, RetentionDays, AccessErrorCount, ScanDurationSeconds, CollectionStatus\n| order by TotalGB desc",
        "size": 0,
        "title": "Current ERP Log Folder Metrics",
        "timeContext": {
          "durationMs": 604800000
        },
        "visualization": "table"
      },
      "name": "CurrentErpFolderMetrics"
    },
    {
      "type": 3,
      "content": {
        "version": "KqlItem/1.0",
        "query": "let CurrentMetrics = ErpFolderMetrics_CL | where TimeGenerated > ago(2h) | summarize arg_max(TimeGenerated, TotalBytes, TotalGB, FileCount, CollectionStatus) by Computer, Application, FolderPath;\nlet PreviousMetrics = ErpFolderMetrics_CL | where TimeGenerated between (ago(26h) .. ago(22h)) | summarize arg_max(TimeGenerated, TotalBytes, TotalGB, FileCount) by Computer, Application, FolderPath;\nCurrentMetrics\n| join kind=leftouter PreviousMetrics on Computer, Application, FolderPath\n| extend GrowthGB24h = round(todouble(TotalBytes - TotalBytes1) / 1024 / 1024 / 1024, 2)\n| project Computer, Application, FolderPath, TotalGB, GrowthGB24h, FileCount, CollectionStatus\n| order by GrowthGB24h desc",
        "size": 0,
        "title": "ERP Log Folder Growth in 24 Hours",
        "timeContext": {
          "durationMs": 604800000
        },
        "visualization": "table"
      },
      "name": "ErpFolderGrowth24Hours"
    },
    {
      "type": 3,
      "content": {
        "version": "KqlItem/1.0",
        "query": "Perf\n| where TimeGenerated {TimeRange}\n| where ObjectName == \"LogicalDisk\"\n| where CounterName in (\"Avg. Disk sec/Read\", \"Avg. Disk sec/Write\", \"Current Disk Queue Length\")\n| where InstanceName != \"_Total\"\n| summarize AverageValue = avg(CounterValue) by Computer, InstanceName, CounterName\n| evaluate pivot(CounterName, max(AverageValue))\n| extend ReadLatencyMs = round(todouble([\"Avg. Disk sec/Read\"]) * 1000, 2)\n| extend WriteLatencyMs = round(todouble([\"Avg. Disk sec/Write\"]) * 1000, 2)\n| extend QueueLength = round(todouble([\"Current Disk Queue Length\"]), 2)\n| project Computer, Drive = InstanceName, ReadLatencyMs, WriteLatencyMs, QueueLength\n| order by WriteLatencyMs desc",
        "size": 0,
        "title": "Disk Latency and Queue Length",
        "timeContext": {
          "durationMs": 604800000
        },
        "visualization": "table"
      },
      "name": "DiskLatency"
    },
    {
      "type": 3,
      "content": {
        "version": "KqlItem/1.0",
        "query": "CollectorHealth_CL\n| where TimeGenerated {TimeRange}\n| where CollectorName == \"ErpFolderMetrics\"\n| summarize arg_max(TimeGenerated, *) by Computer, CollectorName\n| project TimeGenerated, Computer, Status, RecordsCollected, RecordsSubmitted, DurationSeconds, ErrorCode, ErrorMessage, CollectorVersion\n| order by TimeGenerated desc",
        "size": 0,
        "title": "ERP Folder Collector Health",
        "timeContext": {
          "durationMs": 604800000
        },
        "visualization": "table"
      },
      "name": "ErpFolderCollectorHealth"
    }
  ],
  "isLocked": false,
  "fallbackResourceIds": [
    "/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.OperationalInsights/workspaces/<workspace-name>"
  ]
}
```

---

# 9. Workbook Bicep Deployment

Reuse the generic Workbook Bicep module from Phase 3.

Add to `main.bicep`:

```bicep name=azure/main.bicep
@description('Enable deployment of the ERP disk and folder monitoring Workbook.')
param enableDiskAndFolderWorkbook bool = true

module diskAndFolderWorkbook './modules/certificate-workbook.bicep' = if (enableDiskAndFolderWorkbook) {
  name: 'deploy-disk-folder-workbook'
  params: {
    location: location
    workspaceResourceId: workspace.outputs.workspaceResourceId
    workbookDisplayName: 'ERP Disk and Log Folder Monitoring'
    workbookSerializedData: loadTextContent('../workbooks/disk-erp-folder-monitoring.workbook.json')
  }
  dependsOn: [
    tables
    windowsDiskPerformanceDcr
  ]
}
```

For a production repository, rename the generic module:

```text
certificate-workbook.bicep
```

to:

```text
monitoring-workbook.bicep
```

because it now deploys both certificate and disk Workbooks.

---

# 10. Disk and ERP Folder Operations Runbook

```markdown name=docs/runbooks/DiskAndErpLogCapacityResponse.md
# Disk and ERP Log Capacity Response Runbook

## Trigger Conditions

- Disk capacity warning or critical alert.
- Sustained disk read/write latency alert.
- ERP log-folder growth alert.
- ERP log retention accumulation alert.
- ERP folder collector failure or missing telemetry alert.

## Initial Triage

1. Open the **ERP Disk and Log Folder Monitoring** Workbook.
2. Identify:
   - Server.
   - Drive letter.
   - Available free percentage and free GB.
   - ERP log folder size.
   - 24-hour folder growth.
   - Old-file count and retention threshold.
   - Disk read/write latency.
3. Confirm whether the affected drive hosts:
   - ERP application logs.
   - IIS logs.
   - Middleware queues.
   - SQL data/log/tempdb.
   - Backup files.
4. Check active incident/change records before removing any files.
5. Confirm whether the growth is expected, such as during batch processing, upgrades, or diagnostics.

## Immediate Action for Critical Disk Capacity

1. Preserve logs required for incident investigation, audit, or legal hold.
2. Stop uncontrolled log generation only through approved application/vendor procedures.
3. Archive or remove only data approved by retention policy.
4. Confirm log rotation and cleanup jobs are operational.
5. Extend storage if approved and necessary.
6. Re-run the folder collector.
7. Verify that the disk-capacity alert resolves.

## High Disk Latency Investigation

1. Correlate with ERP response-time, CPU, memory, backup, antivirus, and storage events.
2. Identify competing processes or backup/maintenance windows.
3. Confirm storage subsystem health with infrastructure/storage support.
4. For SQL-related volumes, correlate with SQL I/O stall and wait statistics.
5. Do not restart ERP or storage services without approved incident/change procedure.

## Evidence Required

- Server and drive.
- Before/after free GB and free percentage.
- ERP folder growth rate.
- Cleanup/archive action.
- Change or incident reference.
- Root cause and prevention action.
```

---

# 11. Phase 4 Test Plan

## Test 1 — Native AMA Disk Counters

1. Confirm AMA is installed on `ERP-APP-01`.
2. Associate the Windows disk DCR.
3. Wait for disk samples to arrive.
4. Run `CurrentDiskCapacity.kql`.
5. Confirm drive-level values appear for the expected ERP server.

## Test 2 — Warning Capacity Threshold

In a non-production environment:

1. Use a test volume.
2. Create test files until the warning threshold is crossed.
3. Wait for AMA collection and alert evaluation.
4. Verify:
   - KQL result.
   - Alert activation.
   - Email delivery.
   - Teams delivery.
5. Remove approved test files.
6. Verify alert resolution.

## Test 3 — ERP Folder Growth

1. Add controlled test log files under the approved ERP test log directory.
2. Run the folder collector.
3. Validate `ErpFolderMetrics_CL`.
4. Compare current and previous metrics after the test interval.
5. Confirm the growth alert triggers when the configured threshold is exceeded.

## Test 4 — Retention Accumulation

1. Add a harmless test file with an old `LastWriteTime`.
2. Run the folder collector.
3. Confirm `FilesOlderThanRetention` increases.
4. Confirm the retention alert triggers.
5. Remove the test artifact and verify auto-resolution.

## Test 5 — Scan Access Failure

1. Create a restricted test subdirectory under the monitored path.
2. Run the collector under the scheduled-task account.
3. Confirm:
   - `AccessErrorCount` is non-zero.
   - `CollectionStatus` is `PartialSuccess`.
   - `CollectorHealth_CL` reflects the appropriate status.
4. Restore permissions.

## Test 6 — Scan Timeout

1. Configure a short test `MaximumScanDurationMinutes`.
2. Use a safely large test directory.
3. Run the collector.
4. Confirm:
   - `CollectionStatus` becomes `ScanTimedOut`.
   - Local collector log records the timeout.
   - The operational team receives the collector-health alert if configured.

---

# 12. Phase 4 Acceptance Criteria

| Area | Acceptance criterion |
|---|---|
| Native disk telemetry | AMA collects required `LogicalDisk` counters from the ERP server. |
| Disk capacity dashboard | Workbook shows current free percentage and free GB by drive. |
| Disk performance dashboard | Workbook shows read/write latency and queue length. |
| ERP folder telemetry | Folder collector reports size, file count, retention age, scan duration, and access errors. |
| Capacity alerts | Warning and critical disk alerts activate and resolve correctly. |
| Folder-growth alert | Rapid ERP log growth is detected using 24-hour trend comparison. |
| Retention alert | Old ERP logs are visible and alertable. |
| Collector health | Failed, timed-out, or missing ERP folder collection is detected. |
| Email and Teams | Alerts are routed to approved support recipients/channels. |
| Runbook | Capacity and log-retention response procedure is approved. |

### Want me to…
<!-- copilot:suggestions -->
1. Continue to Phase 5 with fault-injection testing, operational handover, cost governance, lifecycle management, and production rollout gates.
2. Add a custom PowerShell logical-disk collector and `WindowsDiskMetrics_CL` table for environments where AMA performance-counter collection is not approved.
3. Hand this disk and ERP folder monitoring implementation off to the coding agent to open a pull request in a repository you provide.
