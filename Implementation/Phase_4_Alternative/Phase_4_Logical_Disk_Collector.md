# Custom PowerShell Logical-Disk Collector Alternative

Use this design only where **AMA performance-counter collection is unavailable or not approved**. It uses the same Phase 1 collector framework:

```text
Windows Task Scheduler
    → Collect-WindowsDiskMetrics.ps1
    → Azure Arc managed identity
    → Logs Ingestion API
    → DCE / DCR
    → WindowsDiskMetrics_CL
    → KQL / Workbook / alerts
```

This collector complements—not replaces—the ERP folder collector:

| Collector | Scope |
|---|---|
| `Collect-WindowsDiskMetrics.ps1` | Logical volume capacity and selected disk performance counters |
| `Collect-ErpFolderMetrics.ps1` | ERP log-directory size, file count, retention, growth, and scan status |

---

## 1. Add Package Files

Add these files to the Phase 1 package:

```text
AzureMonitorCollectors/
├── Collectors/
│   └── Collect-WindowsDiskMetrics.ps1
│
└── Config/
    └── WindowsDiskMetrics.config.json
```

The deployed server structure becomes:

```text
C:\ProgramData\Contoso\AzureMonitorCollectors\
└── Current\
    ├── Collectors\
    │   ├── Collect-CertificateInventory.ps1
    │   ├── Collect-ErpFolderMetrics.ps1
    │   ├── Collect-WindowsDiskMetrics.ps1
    │   └── Submit-CollectorSpool.ps1
    │
    └── Config\
        ├── CollectorSettings.json
        ├── CertificateInventory.config.json
        ├── ErpFolderMetrics.config.json
        └── WindowsDiskMetrics.config.json
```

---

# 2. `WindowsDiskMetrics_CL` Custom Table

Add this table to `azure/modules/custom-tables.bicep`.

```bicep name=azure/modules/custom-tables.bicep
resource windowsDiskMetricsTable 'Microsoft.OperationalInsights/workspaces/tables@2022-10-01' = {
  parent: workspace
  name: 'WindowsDiskMetrics_CL'
  properties: {
    plan: 'Analytics'
    retentionInDays: 90
    totalRetentionInDays: 90
    schema: {
      name: 'WindowsDiskMetrics_CL'
      columns: [
        {
          name: 'TimeGenerated'
          type: 'datetime'
        }
        {
          name: 'Computer'
          type: 'string'
        }
        {
          name: 'Environment'
          type: 'string'
        }
        {
          name: 'Application'
          type: 'string'
        }
        {
          name: 'Drive'
          type: 'string'
        }
        {
          name: 'VolumeLabel'
          type: 'string'
        }
        {
          name: 'FileSystem'
          type: 'string'
        }
        {
          name: 'TotalBytes'
          type: 'long'
        }
        {
          name: 'FreeBytes'
          type: 'long'
        }
        {
          name: 'UsedBytes'
          type: 'long'
        }
        {
          name: 'TotalGB'
          type: 'real'
        }
        {
          name: 'FreeGB'
          type: 'real'
        }
        {
          name: 'UsedGB'
          type: 'real'
        }
        {
          name: 'FreePercent'
          type: 'real'
        }
        {
          name: 'UsedPercent'
          type: 'real'
        }
        {
          name: 'ReadLatencyMs'
          type: 'real'
        }
        {
          name: 'WriteLatencyMs'
          type: 'real'
        }
        {
          name: 'DiskQueueLength'
          type: 'real'
        }
        {
          name: 'DiskTransfersPerSecond'
          type: 'real'
        }
        {
          name: 'CollectionStatus'
          type: 'string'
        }
        {
          name: 'CollectorVersion'
          type: 'string'
        }
      ]
    }
  }
}
```

Add this output at the end of the same Bicep file:

```bicep name=azure/modules/custom-tables.bicep
output windowsDiskMetricsTableName string = windowsDiskMetricsTable.name
```

---

## 2.1 Table Schema Rationale

| Column group | Purpose |
|---|---|
| `TotalBytes`, `FreeBytes`, `UsedBytes` | Precise capacity information without rounding loss. |
| `TotalGB`, `FreeGB`, `UsedGB` | Dashboard-friendly values. |
| `FreePercent`, `UsedPercent` | Relative capacity risk. |
| `ReadLatencyMs`, `WriteLatencyMs` | Disk performance health. |
| `DiskQueueLength` | Potential saturation indicator. |
| `DiskTransfersPerSecond` | Indicates activity level during performance diagnosis. |
| `CollectionStatus` | `Success`, `PartialSuccess`, `CounterUnavailable`, or `DriveNotFound`. |
| `CollectorVersion` | Supports code/schema troubleshooting and upgrade control. |

---

# 3. Add the Disk Metrics DCR

Add a fourth direct-ingestion DCR to the `data-collection-rules.bicep` module.

## 3.1 New Parameter

```bicep name=azure/modules/data-collection-rules.bicep
@description('Windows logical-disk metrics DCR name.')
param windowsDiskMetricsDcrName string
```

## 3.2 Disk Metrics DCR Resource

```bicep name=azure/modules/data-collection-rules.bicep
resource windowsDiskMetricsDcr 'Microsoft.Insights/dataCollectionRules@2023-03-11' = {
  name: windowsDiskMetricsDcrName
  location: location
  kind: 'Direct'
  tags: {
    Application: 'ERP-Monitoring'
    Environment: environment
    ManagedBy: 'Bicep'
    DataClassification: 'Internal'
    Collector: 'WindowsDiskMetrics'
  }
  properties: {
    dataCollectionEndpointId: dataCollectionEndpointResourceId
    streamDeclarations: {
      'Custom-WindowsDiskMetricsRaw': {
        columns: [
          {
            name: 'TimeGenerated'
            type: 'datetime'
          }
          {
            name: 'Computer'
            type: 'string'
          }
          {
            name: 'Environment'
            type: 'string'
          }
          {
            name: 'Application'
            type: 'string'
          }
          {
            name: 'Drive'
            type: 'string'
          }
          {
            name: 'VolumeLabel'
            type: 'string'
          }
          {
            name: 'FileSystem'
            type: 'string'
          }
          {
            name: 'TotalBytes'
            type: 'long'
          }
          {
            name: 'FreeBytes'
            type: 'long'
          }
          {
            name: 'UsedBytes'
            type: 'long'
          }
          {
            name: 'TotalGB'
            type: 'real'
          }
          {
            name: 'FreeGB'
            type: 'real'
          }
          {
            name: 'UsedGB'
            type: 'real'
          }
          {
            name: 'FreePercent'
            type: 'real'
          }
          {
            name: 'UsedPercent'
            type: 'real'
          }
          {
            name: 'ReadLatencyMs'
            type: 'real'
          }
          {
            name: 'WriteLatencyMs'
            type: 'real'
          }
          {
            name: 'DiskQueueLength'
            type: 'real'
          }
          {
            name: 'DiskTransfersPerSecond'
            type: 'real'
          }
          {
            name: 'CollectionStatus'
            type: 'string'
          }
          {
            name: 'CollectorVersion'
            type: 'string'
          }
        ]
      }
    }
    destinations: {
      logAnalytics: [
        {
          name: 'windowsDiskMetricsWorkspace'
          workspaceResourceId: workspaceResourceId
        }
      ]
    }
    dataFlows: [
      {
        streams: [
          'Custom-WindowsDiskMetricsRaw'
        ]
        destinations: [
          'windowsDiskMetricsWorkspace'
        ]
        transformKql: 'source'
        outputStream: 'Custom-WindowsDiskMetrics_CL'
      }
    ]
  }
}
```

## 3.3 Add Output

```bicep name=azure/modules/data-collection-rules.bicep
output windowsDiskMetricsDcrResourceId string = windowsDiskMetricsDcr.id
output windowsDiskMetricsDcrImmutableId string = windowsDiskMetricsDcr.properties.immutableId
```

---

# 4. Add DCR RBAC Assignment

## 4.1 Add Parameter to `dcr-role-assignments.bicep`

```bicep name=azure/modules/dcr-role-assignments.bicep
@description('Windows disk metrics DCR resource ID.')
param windowsDiskMetricsDcrResourceId string

@description('Azure Arc managed identity principal IDs for Windows disk metrics collector hosts.')
param windowsDiskMetricsCollectorPrincipalIds array
```

## 4.2 Add Role Assignment Resource

```bicep name=azure/modules/dcr-role-assignments.bicep
resource windowsDiskMetricsDcrRoleAssignments 'Microsoft.Authorization/roleAssignments@2022-04-01' = [
  for principalId in windowsDiskMetricsCollectorPrincipalIds: {
    name: guid(
      windowsDiskMetricsDcrResourceId
      principalId
      monitoringMetricsPublisherRoleDefinitionId
    )
    scope: resourceId(
      'Microsoft.Insights/dataCollectionRules'
      last(split(windowsDiskMetricsDcrResourceId, '/'))
    )
    properties: {
      roleDefinitionId: monitoringMetricsPublisherRoleDefinitionId
      principalId: principalId
      principalType: 'ServicePrincipal'
    }
  }
]
```

The Azure Arc server managed identity should receive the **Monitoring Metrics Publisher** built-in role only at this DCR scope.

---

# 5. Update `main.bicep`

## 5.1 Add Parameters

```bicep name=azure/main.bicep
@description('Windows logical-disk custom collector DCR name.')
param windowsDiskMetricsDcrName string

@description('Azure Arc managed identity principal IDs allowed to submit logical-disk metric data.')
param windowsDiskMetricsCollectorPrincipalIds array = []
```

## 5.2 Pass the New DCR Parameter

```bicep name=azure/main.bicep
module dcrs './modules/data-collection-rules.bicep' = {
  name: 'deploy-direct-ingestion-dcrs'
  params: {
    location: location
    workspaceResourceId: workspace.outputs.workspaceResourceId
    dataCollectionEndpointResourceId: dce.outputs.dataCollectionEndpointResourceId
    certificateDcrName: certificateDcrName
    folderMetricsDcrName: folderMetricsDcrName
    collectorHealthDcrName: collectorHealthDcrName
    windowsDiskMetricsDcrName: windowsDiskMetricsDcrName
    environment: environment
  }
}
```

## 5.3 Pass Principal IDs to the RBAC Module

```bicep name=azure/main.bicep
module roles './modules/dcr-role-assignments.bicep' = {
  name: 'assign-dcr-ingestion-roles'
  params: {
    certificateDcrResourceId: dcrs.outputs.certificateDcrResourceId
    folderMetricsDcrResourceId: dcrs.outputs.folderMetricsDcrResourceId
    collectorHealthDcrResourceId: dcrs.outputs.collectorHealthDcrResourceId
    windowsDiskMetricsDcrResourceId: dcrs.outputs.windowsDiskMetricsDcrResourceId
    certificateCollectorPrincipalIds: certificateCollectorPrincipalIds
    folderCollectorPrincipalIds: folderCollectorPrincipalIds
    healthCollectorPrincipalIds: healthCollectorPrincipalIds
    windowsDiskMetricsCollectorPrincipalIds: windowsDiskMetricsCollectorPrincipalIds
  }
}
```

## 5.4 Add Outputs

```bicep name=azure/main.bicep
output windowsDiskMetricsDcrResourceId string = dcrs.outputs.windowsDiskMetricsDcrResourceId
output windowsDiskMetricsDcrImmutableId string = dcrs.outputs.windowsDiskMetricsDcrImmutableId
```

---

# 6. Pilot Parameter Updates

```bicep name=azure/parameters/pilot.bicepparam
param windowsDiskMetricsDcrName = 'dcr-erpmon-windows-disk-custom-pilot-001'

param windowsDiskMetricsCollectorPrincipalIds = [
  '00000000-0000-0000-0000-000000000002'
]
```

Use the principal ID of the Azure Arc system-assigned managed identity for the ERP application server hosting:

```text
Contoso-Monitor-WindowsDiskMetrics
```

---

# 7. Disk Collector Configuration

```json name=Config/WindowsDiskMetrics.config.json
{
  "SchemaVersion": "1.0",
  "CollectorName": "WindowsDiskMetrics",
  "Enabled": true,
  "IngestionEndpoint": "https://<logs-ingestion-endpoint-from-bicep-output>",
  "DcrImmutableId": "dcr-<windows-disk-metrics-dcr-immutable-id>",
  "StreamName": "Custom-WindowsDiskMetricsRaw",
  "HealthIngestionEndpoint": "https://<logs-ingestion-endpoint-from-bicep-output>",
  "HealthDcrImmutableId": "dcr-<collector-health-dcr-immutable-id>",
  "HealthStreamName": "Custom-CollectorHealthRaw",
  "SampleIntervalSeconds": 1,
  "Drives": [
    {
      "Drive": "C:",
      "Application": "WindowsOS",
      "Enabled": true
    },
    {
      "Drive": "D:",
      "Application": "ERP",
      "Enabled": true
    }
  ]
}
```

## Configuration Guidance

- Monitor only approved drives initially.
- Add the ERP log drive, operating-system drive, and any dedicated application-data drive.
- Do not monitor removable drives, CD-ROM drives, or all mounted volumes unless there is a defined requirement.
- Use `Application: "ERP"` for the volume containing the ERP log directory so Workbooks can correlate the drive and folder data.

---

# 8. Custom Logical-Disk Collector

The collector uses:

- `Get-CimInstance Win32_LogicalDisk` for capacity.
- `Get-Counter` for logical-disk latency, queue length, and disk-transfer measurements.
- The shared Phase 1 `AzureMonitorCollector.Common.psm1` module for:
  - Arc managed-identity token acquisition.
  - Logs Ingestion API submission.
  - Spool/retry.
  - Local logging.
  - Execution locking.
  - Health records.

> **Localization caution:** Windows Performance Counter names can be localized. The initial sample uses English counter paths. For non-English Windows Server installations, validate the counter paths with `Get-Counter -ListSet LogicalDisk` and replace the configured names or implement a localized counter-name mapping.

```powershell name=Collectors/Collect-WindowsDiskMetrics.ps1
[CmdletBinding()]
param()

Set-StrictMode -Version Latest
$ErrorActionPreference = 'Stop'

$collectorRoot = Split-Path -Path $PSScriptRoot -Parent
$modulePath = Join-Path $collectorRoot 'Modules\AzureMonitorCollector.Common.psm1'
$configRoot = Join-Path $collectorRoot 'Config'

Import-Module $modulePath -Force

$sharedSettings = Read-CollectorJsonConfiguration `
    -Path (Join-Path $configRoot 'CollectorSettings.json')

$collectorSettings = Read-CollectorJsonConfiguration `
    -Path (Join-Path $configRoot 'WindowsDiskMetrics.config.json')

if (-not [bool]$collectorSettings.Enabled) {
    Write-Output 'Windows logical-disk collector is disabled by configuration.'
    exit 0
}

$collectorName = [string]$collectorSettings.CollectorName
$executionId = [guid]::NewGuid().ToString()
$startedUtc = (Get-Date).ToUniversalTime()
$logDirectory = Join-Path $sharedSettings.CollectorRoot "Logs\$collectorName"
$spoolDirectory = Join-Path $sharedSettings.CollectorRoot "Spool\$collectorName"
$lockPath = Join-Path $sharedSettings.CollectorRoot "Locks\$collectorName.lock"
$lockHandle = $null

$recordsCollected = 0
$recordsSubmitted = 0
$status = 'Success'
$errorCode = $null
$errorMessage = $null

function Get-LogicalDiskPerformanceSamples {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)]
        [string[]] $Drives,

        [ValidateRange(1, 10)]
        [int] $SampleIntervalSeconds = 1
    )

    $counterPaths = [System.Collections.Generic.List[string]]::new()

    foreach ($drive in $Drives) {
        $normalizedDrive = $drive.TrimEnd('\').TrimEnd(':')

        foreach ($counterName in @(
            'Avg. Disk sec/Read',
            'Avg. Disk sec/Write',
            'Current Disk Queue Length',
            'Disk Transfers/sec'
        )) {
            $counterPaths.Add("\LogicalDisk($normalizedDrive`:)\$counterName")
        }
    }

    try {
        # Two samples provide a more meaningful rate-based sample for Disk Transfers/sec.
        $counterResult = Get-Counter `
            -Counter $counterPaths.ToArray() `
            -SampleInterval $SampleIntervalSeconds `
            -MaxSamples 2 `
            -ErrorAction Stop

        $latestSample = $counterResult[-1]

        $results = @{}

        foreach ($sample in $latestSample.CounterSamples) {
            $instance = $sample.InstanceName
            $counter = $sample.Path.Split('\')[-1]

            if (-not $results.ContainsKey($instance)) {
                $results[$instance] = @{}
            }

            $results[$instance][$counter] = [double]$sample.CookedValue
        }

        return $results
    }
    catch {
        Write-Warning (
            "Unable to collect one or more LogicalDisk performance counters. " +
            "$($_.Exception.Message)"
        )

        return @{}
    }
}

function Get-PerformanceValue {
    param(
        [hashtable] $PerformanceData,

        [string] $Drive,

        [string] $CounterName
    )

    $instance = $Drive.TrimEnd('\').TrimEnd(':') + ':'

    if ($PerformanceData.ContainsKey($instance) -and
        $PerformanceData[$instance].ContainsKey($CounterName)) {
        return [double]$PerformanceData[$instance][$CounterName]
    }

    return $null
}

try {
    $lockHandle = Enter-CollectorLock -LockPath $lockPath

    Write-CollectorLog `
        -LogDirectory $logDirectory `
        -Level Information `
        -ExecutionId $executionId `
        -Message 'Windows logical-disk metrics collection started.'

    Remove-CollectorExpiredFiles `
        -Directory $logDirectory `
        -RetentionDays ([int]$sharedSettings.Logging.RetentionDays)

    Remove-CollectorExpiredFiles `
        -Directory $spoolDirectory `
        -RetentionDays ([int]$sharedSettings.Ingestion.SpoolRetentionDays)

    Submit-CollectorSpoolBatches `
        -SpoolDirectory $spoolDirectory `
        -SharedSettings $sharedSettings `
        -LogDirectory $logDirectory `
        -ExecutionId $executionId | Out-Null

    $configuredDrives = @(
        $collectorSettings.Drives |
            Where-Object { $_.Enabled } |
            ForEach-Object { [string]$_.Drive }
    )

    if ($configuredDrives.Count -eq 0) {
        throw 'No enabled logical drives are defined in WindowsDiskMetrics.config.json.'
    }

    $performanceData = Get-LogicalDiskPerformanceSamples `
        -Drives $configuredDrives `
        -SampleIntervalSeconds ([int]$collectorSettings.SampleIntervalSeconds)

    $logicalDisks = Get-CimInstance `
        -ClassName Win32_LogicalDisk `
        -Filter 'DriveType = 3' `
        -ErrorAction Stop

    $collectionTime = (Get-Date).ToUniversalTime()
    $records = [System.Collections.Generic.List[object]]::new()

    foreach ($driveDefinition in $collectorSettings.Drives) {
        if (-not [bool]$driveDefinition.Enabled) {
            continue
        }

        $driveName = ([string]$driveDefinition.Drive).TrimEnd('\').TrimEnd(':') + ':'

        $logicalDisk = $logicalDisks |
            Where-Object { $_.DeviceID -eq $driveName } |
            Select-Object -First 1

        if ($null -eq $logicalDisk) {
            $records.Add([pscustomobject]@{
                TimeGenerated          = $collectionTime.ToString('o')
                Computer               = $env:COMPUTERNAME
                Environment            = $sharedSettings.Environment
                Application            = $driveDefinition.Application
                Drive                  = $driveName
                VolumeLabel            = $null
                FileSystem             = $null
                TotalBytes             = [Int64]0
                FreeBytes              = [Int64]0
                UsedBytes              = [Int64]0
                TotalGB                = [double]0
                FreeGB                 = [double]0
                UsedGB                 = [double]0
                FreePercent            = $null
                UsedPercent            = $null
                ReadLatencyMs          = $null
                WriteLatencyMs         = $null
                DiskQueueLength        = $null
                DiskTransfersPerSecond = $null
                CollectionStatus       = 'DriveNotFound'
                CollectorVersion       = '1.0.0'
            })

            $status = 'PartialSuccess'
            continue
        }

        [Int64]$totalBytes = $logicalDisk.Size
        [Int64]$freeBytes = $logicalDisk.FreeSpace
        [Int64]$usedBytes = $totalBytes - $freeBytes

        $freePercent = if ($totalBytes -gt 0) {
            [Math]::Round(($freeBytes / $totalBytes) * 100, 2)
        }
        else {
            $null
        }

        $usedPercent = if ($null -ne $freePercent) {
            [Math]::Round(100 - $freePercent, 2)
        }
        else {
            $null
        }

        $readLatencySeconds = Get-PerformanceValue `
            -PerformanceData $performanceData `
            -Drive $driveName `
            -CounterName 'Avg. Disk sec/Read'

        $writeLatencySeconds = Get-PerformanceValue `
            -PerformanceData $performanceData `
            -Drive $driveName `
            -CounterName 'Avg. Disk sec/Write'

        $queueLength = Get-PerformanceValue `
            -PerformanceData $performanceData `
            -Drive $driveName `
            -CounterName 'Current Disk Queue Length'

        $diskTransfers = Get-PerformanceValue `
            -PerformanceData $performanceData `
            -Drive $driveName `
            -CounterName 'Disk Transfers/sec'

        $performanceAvailable = (
            $null -ne $readLatencySeconds -or
            $null -ne $writeLatencySeconds -or
            $null -ne $queueLength -or
            $null -ne $diskTransfers
        )

        $collectionStatus = if ($performanceAvailable) {
            'Success'
        }
        else {
            'CounterUnavailable'
        }

        if ($collectionStatus -ne 'Success') {
            $status = 'PartialSuccess'
        }

        $records.Add([pscustomobject]@{
            TimeGenerated          = $collectionTime.ToString('o')
            Computer               = $env:COMPUTERNAME
            Environment            = $sharedSettings.Environment
            Application            = $driveDefinition.Application
            Drive                  = $driveName
            VolumeLabel            = $logicalDisk.VolumeName
            FileSystem             = $logicalDisk.FileSystem
            TotalBytes             = $totalBytes
            FreeBytes              = $freeBytes
            UsedBytes              = $usedBytes
            TotalGB                = [Math]::Round($totalBytes / 1GB, 3)
            FreeGB                 = [Math]::Round($freeBytes / 1GB, 3)
            UsedGB                 = [Math]::Round($usedBytes / 1GB, 3)
            FreePercent            = $freePercent
            UsedPercent            = $usedPercent
            ReadLatencyMs          = if ($null -ne $readLatencySeconds) {
                [Math]::Round($readLatencySeconds * 1000, 2)
            } else {
                $null
            }
            WriteLatencyMs         = if ($null -ne $writeLatencySeconds) {
                [Math]::Round($writeLatencySeconds * 1000, 2)
            } else {
                $null
            }
            DiskQueueLength        = if ($null -ne $queueLength) {
                [Math]::Round($queueLength, 2)
            } else {
                $null
            }
            DiskTransfersPerSecond = if ($null -ne $diskTransfers) {
                [Math]::Round($diskTransfers, 2)
            } else {
                $null
            }
            CollectionStatus       = $collectionStatus
            CollectorVersion       = '1.0.0'
        })
    }

    $recordsCollected = $records.Count

    $recordsSubmitted = Send-AzureMonitorRecords `
        -CollectorName $collectorName `
        -IngestionEndpoint $collectorSettings.IngestionEndpoint `
        -DcrImmutableId $collectorSettings.DcrImmutableId `
        -StreamName $collectorSettings.StreamName `
        -Records $records.ToArray() `
        -SharedSettings $sharedSettings `
        -SpoolDirectory $spoolDirectory `
        -LogDirectory $logDirectory `
        -ExecutionId $executionId

    Write-CollectorLog `
        -LogDirectory $logDirectory `
        -Level Information `
        -ExecutionId $executionId `
        -Message (
            "Windows disk metrics completed. Collected=$recordsCollected; " +
            "submitted=$recordsSubmitted."
        )
}
catch {
    $status = 'Failed'
    $errorCode = 'WindowsDiskMetricsFailure'
    $errorMessage = $_.Exception.Message

    Write-CollectorLog `
        -LogDirectory $logDirectory `
        -Level Error `
        -ExecutionId $executionId `
        -Message "Windows disk metrics collector failed. Error: $errorMessage"
}
finally {
    $healthRecord = New-CollectorHealthRecord `
        -CollectorName $collectorName `
        -ExecutionId $executionId `
        -StartedUtc $startedUtc `
        -Status $status `
        -RecordsCollected $recordsCollected `
        -RecordsSubmitted $recordsSubmitted `
        -TargetScope (
            ($collectorSettings.Drives |
                Where-Object { $_.Enabled } |
                ForEach-Object { $_.Drive }) -join ';'
        ) `
        -ErrorCode $errorCode `
        -ErrorMessage $errorMessage `
        -CollectorVersion '1.0.0'

    try {
        Send-AzureMonitorRecords `
            -CollectorName $collectorName `
            -IngestionEndpoint $collectorSettings.HealthIngestionEndpoint `
            -DcrImmutableId $collectorSettings.HealthDcrImmutableId `
            -StreamName $collectorSettings.HealthStreamName `
            -Records @($healthRecord) `
            -SharedSettings $sharedSettings `
            -SpoolDirectory $spoolDirectory `
            -LogDirectory $logDirectory `
            -ExecutionId $executionId | Out-Null
    }
    catch {
        Write-CollectorLog `
            -LogDirectory $logDirectory `
            -Level Error `
            -ExecutionId $executionId `
            -Message (
                "Unable to submit disk collector health record. " +
                "Error: $($_.Exception.Message)"
            )
    }

    if ($null -ne $lockHandle) {
        Exit-CollectorLock -LockHandle $lockHandle -LockPath $lockPath
    }
}

if ($status -eq 'Failed') {
    exit 1
}

exit 0
```

---

# 9. Scheduled Task Registration Update

Add a logical-disk task to `Register-CollectorScheduledTasks.ps1`.

## 9.1 Add the Script Path

```powershell name=Register-CollectorScheduledTasks.ps1
$windowsDiskMetricsScript = Join-Path $collectorPath 'Collect-WindowsDiskMetrics.ps1'
```

## 9.2 Add a Trigger

For capacity monitoring, collect every 15 minutes initially.

```powershell name=Register-CollectorScheduledTasks.ps1
$diskMetricsTrigger = New-ScheduledTaskTrigger -Once -At (Get-Date).Date.AddMinutes(5)
$diskMetricsTrigger.Repetition.Interval = New-TimeSpan -Minutes 15
$diskMetricsTrigger.Repetition.Duration = New-TimeSpan -Days 1
```

## 9.3 Register the Task

```powershell name=Register-CollectorScheduledTasks.ps1
Register-CollectorTask `
    -TaskName 'Contoso-Monitor-WindowsDiskMetrics' `
    -ScriptPath $windowsDiskMetricsScript `
    -Triggers @($diskMetricsTrigger) `
    -Description (
        'Collects Windows logical-disk capacity and performance metrics ' +
        'for Azure Monitor custom ingestion.'
    )
```

## 9.4 Add Spool Replay Task

Add disk collector spool replay action:

```powershell name=Register-CollectorScheduledTasks.ps1
$diskSpoolAction = New-ScheduledTaskAction `
    -Execute $powerShell `
    -Argument (
        '-NoLogo -NoProfile -NonInteractive -ExecutionPolicy AllSigned ' +
        "-File `"$spoolScript`" -CollectorName WindowsDiskMetrics"
    ) `
    -WorkingDirectory $currentPath
```

Register it:

```powershell name=Register-CollectorScheduledTasks.ps1
Register-ScheduledTask `
    -TaskName 'Contoso-Monitor-WindowsDiskMetrics-SpoolReplay' `
    -Action $diskSpoolAction `
    -Trigger $spoolTrigger `
    -Settings $settings `
    -User $CollectorRunAsAccount `
    -Password (
        [Runtime.InteropServices.Marshal]::PtrToStringBSTR(
            [Runtime.InteropServices.Marshal]::SecureStringToBSTR(
                $CollectorRunAsPassword
            )
        )
    ) `
    -RunLevel Limited `
    -Description 'Replays spooled Windows logical-disk telemetry to Azure Monitor.' `
    -Force | Out-Null
```

---

# 10. Disk Metric KQL Queries

When the custom collector is enabled, replace `Perf`-based capacity and latency queries with `WindowsDiskMetrics_CL`.

## Current Disk Capacity

```kusto name=kql/disk-custom/CurrentDiskCapacity.kql
WindowsDiskMetrics_CL
| where TimeGenerated > ago(45m)
| summarize arg_max(TimeGenerated, *) by Computer, Drive
| where CollectionStatus in ("Success", "CounterUnavailable")
| project
    TimeGenerated,
    Computer,
    Environment,
    Application,
    Drive,
    VolumeLabel,
    FileSystem,
    TotalGB,
    FreeGB,
    UsedGB,
    FreePercent,
    UsedPercent,
    CollectionStatus
| order by FreePercent asc, FreeGB asc
```

## Critical Capacity

```kusto name=kql/disk-custom/DiskCapacityCritical.kql
WindowsDiskMetrics_CL
| where TimeGenerated > ago(45m)
| summarize arg_max(TimeGenerated, *) by Computer, Drive
| where CollectionStatus in ("Success", "CounterUnavailable")
| where FreePercent < 10 or FreeGB < 20
| project
    Computer,
    Application,
    Drive,
    VolumeLabel,
    TotalGB,
    FreeGB,
    FreePercent,
    CollectionStatus
| order by FreePercent asc, FreeGB asc
```

## Warning Capacity

```kusto name=kql/disk-custom/DiskCapacityWarning.kql
WindowsDiskMetrics_CL
| where TimeGenerated > ago(45m)
| summarize arg_max(TimeGenerated, *) by Computer, Drive
| where CollectionStatus in ("Success", "CounterUnavailable")
| where FreePercent >= 10 and FreeGB >= 20
| where FreePercent < 20 or FreeGB < 50
| project
    Computer,
    Application,
    Drive,
    VolumeLabel,
    TotalGB,
    FreeGB,
    FreePercent,
    CollectionStatus
| order by FreePercent asc, FreeGB asc
```

## Disk Latency

```kusto name=kql/disk-custom/DiskLatency.kql
WindowsDiskMetrics_CL
| where TimeGenerated > ago(45m)
| summarize arg_max(TimeGenerated, *) by Computer, Drive
| where CollectionStatus == "Success"
| where ReadLatencyMs > 25 or WriteLatencyMs > 25
| project
    Computer,
    Application,
    Drive,
    ReadLatencyMs,
    WriteLatencyMs,
    DiskQueueLength,
    DiskTransfersPerSecond,
    TimeGenerated
| order by WriteLatencyMs desc, ReadLatencyMs desc
```

## Disk Collector Health

```kusto name=kql/disk-custom/DiskCollectorHealth.kql
CollectorHealth_CL
| where TimeGenerated > ago(2h)
| where CollectorName == "WindowsDiskMetrics"
| summarize arg_max(TimeGenerated, *) by Computer, CollectorName
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

# 11. Alert Query Changes

If using the custom logical-disk collector, update the Bicep alert queries from `Perf` to `WindowsDiskMetrics_CL`.

Example critical alert query:

```bicep name=azure/modules/disk-folder-alerts.bicep
var diskCapacityCriticalQuery = '''
WindowsDiskMetrics_CL
| where TimeGenerated > ago(45m)
| summarize arg_max(TimeGenerated, *) by Computer, Drive
| where CollectionStatus in ("Success", "CounterUnavailable")
| where FreePercent < ${diskCriticalFreePercent} or FreeGB < ${diskCriticalFreeGb}
| project Computer, Drive, VolumeLabel, TotalGB, FreeGB, FreePercent, CollectionStatus
'''
```

Example latency alert query:

```bicep name=azure/modules/disk-folder-alerts.bicep
var diskLatencyQuery = '''
WindowsDiskMetrics_CL
| where TimeGenerated > ago(45m)
| summarize arg_max(TimeGenerated, *) by Computer, Drive
| where CollectionStatus == "Success"
| where ReadLatencyMs > ${diskLatencyThresholdMs} or WriteLatencyMs > ${diskLatencyThresholdMs}
| project Computer, Drive, ReadLatencyMs, WriteLatencyMs, DiskQueueLength, DiskTransfersPerSecond
'''
```

Set the alert evaluation schedule to:

| Alert | Evaluation | Window |
|---|---:|---:|
| Capacity critical | 15 minutes | 45 minutes |
| Capacity warning | 15 minutes | 45 minutes |
| Disk latency | 15 minutes | 45 minutes |
| Disk collector failure | 30 minutes | 2 hours |
| Disk metrics missing | 30 minutes | 1 hour |

The longer 45-minute window accounts for the collector’s 15-minute frequency, transient ingestion delay, and one potentially missed scheduled execution.

---

# 12. Workbook Query Changes

Replace the `Perf` query sections of the Phase 4 Workbook with this custom-table query:

```kusto name=workbooks/WindowsDiskMetricsWorkbookQuery.kql
WindowsDiskMetrics_CL
| where TimeGenerated {TimeRange}
| summarize arg_max(TimeGenerated, *) by Computer, Drive
| where CollectionStatus in ("Success", "CounterUnavailable")
| project
    Computer,
    Application,
    Drive,
    VolumeLabel,
    FileSystem,
    TotalGB,
    FreeGB,
    FreePercent,
    ReadLatencyMs,
    WriteLatencyMs,
    DiskQueueLength,
    DiskTransfersPerSecond,
    CollectionStatus,
    TimeGenerated
| order by FreePercent asc, FreeGB asc
```

Add a separate Workbook grid to identify incomplete performance collection:

```kusto name=workbooks/WindowsDiskCounterAvailabilityWorkbookQuery.kql
WindowsDiskMetrics_CL
| where TimeGenerated {TimeRange}
| summarize arg_max(TimeGenerated, *) by Computer, Drive
| where CollectionStatus != "Success"
| project
    TimeGenerated,
    Computer,
    Drive,
    CollectionStatus,
    FreeGB,
    FreePercent,
    ReadLatencyMs,
    WriteLatencyMs,
    DiskQueueLength
| order by TimeGenerated desc
```

---

# 13. Direct-Ingestion Validation Payload

Use this after the new DCR is deployed and the Azure Arc identity has DCR-scoped permissions.

```powershell name=Test-WindowsDiskMetricsDirectIngestion.ps1
[CmdletBinding()]
param(
    [Parameter(Mandatory)]
    [string] $IngestionEndpoint,

    [Parameter(Mandatory)]
    [string] $DcrImmutableId,

    [string] $StreamName = 'Custom-WindowsDiskMetricsRaw'
)

Set-StrictMode -Version Latest
$ErrorActionPreference = 'Stop'

$collectorRoot = 'C:\ProgramData\Contoso\AzureMonitorCollectors\Current'
$modulePath = Join-Path $collectorRoot 'Modules\AzureMonitorCollector.Common.psm1'

Import-Module $modulePath -Force

$record = [pscustomobject]@{
    TimeGenerated          = (Get-Date).ToUniversalTime().ToString('o')
    Computer               = $env:COMPUTERNAME
    Environment            = 'Pilot'
    Application            = 'ERP'
    Drive                  = 'D:'
    VolumeLabel            = 'Validation'
    FileSystem             = 'NTFS'
    TotalBytes             = [Int64]107374182400
    FreeBytes              = [Int64]53687091200
    UsedBytes              = [Int64]53687091200
    TotalGB                = [double]100
    FreeGB                 = [double]50
    UsedGB                 = [double]50
    FreePercent            = [double]50
    UsedPercent            = [double]50
    ReadLatencyMs          = [double]2.1
    WriteLatencyMs         = [double]4.2
    DiskQueueLength        = [double]0.3
    DiskTransfersPerSecond = [double]75
    CollectionStatus       = 'Validation'
    CollectorVersion       = '1.0.0'
}

$settings = [pscustomobject]@{
    ArcManagedIdentity = [pscustomobject]@{
        TokenResource = 'https://monitor.azure.com/'
        TokenApiVersion = '2020-06-01'
    }
    Ingestion = [pscustomobject]@{
        MaximumRecordsPerBatch = 500
        MaximumPayloadBytes = 900000
        MaximumAttempts = 1
        RetryBaseDelaySeconds = 1
        EnableLocalSpool = $false
        SpoolRetentionDays = 1
    }
}

Send-AzureMonitorRecords `
    -CollectorName 'WindowsDiskMetricsValidation' `
    -IngestionEndpoint $IngestionEndpoint `
    -DcrImmutableId $DcrImmutableId `
    -StreamName $StreamName `
    -Records @($record) `
    -SharedSettings $settings `
    -SpoolDirectory 'C:\Temp' `
    -LogDirectory 'C:\Temp' `
    -ExecutionId ([guid]::NewGuid().ToString()) | Out-Null

Write-Output 'Windows disk metrics validation record accepted by Azure Monitor.'
```

Validate it in Log Analytics:

```kusto name=kql/validation/Validate-WindowsDiskMetricsIngestion.kql
WindowsDiskMetrics_CL
| where TimeGenerated > ago(30m)
| where CollectionStatus == "Validation"
| project
    TimeGenerated,
    Computer,
    Drive,
    TotalGB,
    FreeGB,
    FreePercent,
    ReadLatencyMs,
    WriteLatencyMs,
    CollectorVersion
| order by TimeGenerated desc
```

---

# 14. Custom Collector Operational Considerations

## Advantages

- No AMA performance-counter DCR dependency.
- Uses the same custom collector identity, deployment, DCE, DCR, table, and alerting architecture.
- Can be limited to specific drives.
- Allows custom data-model fields such as application ownership and volume role.
- Suitable for tightly controlled environments that approve only one custom telemetry pathway.

## Limitations

| Limitation | Mitigation |
|---|---|
| PowerShell `Get-Counter` adds local execution overhead. | Use 15-minute schedule initially; restrict to approved drives. |
| Performance counter names can be localized. | Validate `Get-Counter -ListSet LogicalDisk` during deployment. |
| A 15-minute collector is not equivalent to continuous performance telemetry. | Use AMA or a dedicated performance-monitoring tool when high-resolution analysis is required. |
| Short point samples can miss transient latency spikes. | Use repeated collection, aggregate trend analysis, and tune schedule/thresholds. |
| Scheduled task may fail or be delayed. | Monitor `CollectorHealth_CL` and missing-data conditions. |
| Custom ingestion has cost/maintenance overhead. | Use only when AMA is not approved; collect aggregate records only. |

---

# 15. Recommended Decision Rule

Use the following selection model:

```text
Is AMA approved for Windows performance counters?
    │
    ├── Yes
    │     └── Use AMA + Microsoft-Perf for disk capacity and latency.
    │         Use custom PowerShell only for ERP folder metrics.
    │
    └── No
          └── Use Collect-WindowsDiskMetrics.ps1
              + WindowsDiskMetrics_CL
              + direct-ingestion DCR.
```

Do **not** run both AMA and the custom disk collector for the same alerting purpose in production unless there is a defined comparison/transition period. Otherwise, duplicate telemetry and duplicate alerts can occur.

### Want me to…
<!-- copilot:suggestions -->
1. Continue to Phase 5 with fault-injection testing, production hardening, operations handover, cost governance, and rollout gates.
2. Add Pester tests for the custom logical-disk collector, including capacity calculation, missing-drive, counter-unavailable, and JSON schema tests.
3. Hand this custom disk collector implementation off to the coding agent to open a pull request in a repository you provide.
