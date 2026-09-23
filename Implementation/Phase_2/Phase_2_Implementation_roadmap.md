# Phase 2 — Azure Monitor Custom Ingestion Infrastructure

This phase deploys the Azure-side platform required by the Phase 1 collectors:

- Pilot Log Analytics Workspace.
- Custom Log Analytics tables.
- Data Collection Endpoint (DCE).
- Three direct-ingestion Data Collection Rules (DCRs):
  - Certificate inventory.
  - ERP folder metrics.
  - Collector health.
- DCR-scoped RBAC for Azure Arc server managed identities.
- Action Group foundation for Outlook/email and Teams/Logic App integration.
- Direct-ingestion validation scripts.

This design uses a **DCE intentionally**, even though public ingestion may support a DCR endpoint in some configurations. A DCE makes the ingestion endpoint explicit, makes future Private Link migration clearer, and gives integrators a stable design boundary.

---

## 1. Phase 2 Target Architecture

```text
Azure Arc-enabled Windows server
    │
    ├── System-assigned managed identity
    ├── Windows Task Scheduler
    └── PowerShell collector
           │
           │ 1. Obtain token through Arc HIMDS
           │ 2. POST JSON to DCE logs-ingestion endpoint
           ▼
Data Collection Endpoint
    │
    ▼
Data Collection Rule
    ├── Validates custom input stream schema
    ├── Applies transformation
    └── Routes data
           │
           ▼
Log Analytics Workspace
    ├── ServerCertificateInventory_CL
    ├── ErpFolderMetrics_CL
    └── CollectorHealth_CL
           │
           ▼
KQL / Workbooks / Scheduled Query Alerts
```

---

# 2. Bicep Repository Structure

Add this structure to the Phase 1 repository:

```text
azure/
├── main.bicep
├── main.bicepparam
│
├── modules/
│   ├── log-analytics-workspace.bicep
│   ├── custom-tables.bicep
│   ├── data-collection-endpoint.bicep
│   ├── data-collection-rules.bicep
│   ├── dcr-role-assignments.bicep
│   └── action-group.bicep
│
└── parameters/
    ├── pilot.bicepparam
    ├── test.bicepparam
    └── production.bicepparam
```

The Phase 2 deployment should initially target a dedicated pilot resource group:

```text
rg-erpmon-pilot-<region>-001
```

Example:

```text
rg-erpmon-pilot-cac-001
```

`cac` is only an example abbreviation for Canada Central. The final region must be approved before deployment.

---

# 3. Design Standards

## 3.1 Custom Tables

| Table | Source stream | Use |
|---|---|---|
| `ServerCertificateInventory_CL` | `Custom-ServerCertificateInventoryRaw` | Certificate metadata from `Cert:\LocalMachine\WebHosting` and optional IIS binding information |
| `ErpFolderMetrics_CL` | `Custom-ErpFolderMetricsRaw` | ERP log-directory aggregate size, file count, age, scan status |
| `CollectorHealth_CL` | `Custom-CollectorHealthRaw` | Custom collector execution status, timing, and error status |

## 3.2 DCR Separation

Use one DCR per functional security boundary:

| DCR | Why it is separate |
|---|---|
| `dcr-erpmon-cert-pilot-001` | Certificate collector identity can ingest only certificate records. |
| `dcr-erpmon-folder-pilot-001` | Folder collector identity can ingest only folder metrics. |
| `dcr-erpmon-health-pilot-001` | Health telemetry is isolated for monitoring collector execution. |

For the pilot, an Arc server identity may receive access to both its applicable functional DCR and the shared health DCR. In production, scope access by role and server.

---

# 4. Main Bicep Template

```bicep name=azure/main.bicep
targetScope = 'resourceGroup'

@description('Azure region for all pilot monitoring resources.')
param location string = resourceGroup().location

@description('Log Analytics Workspace name.')
param logAnalyticsWorkspaceName string

@description('Data Collection Endpoint name.')
param dataCollectionEndpointName string

@description('Certificate inventory DCR name.')
param certificateDcrName string

@description('ERP folder metrics DCR name.')
param folderMetricsDcrName string

@description('Collector health DCR name.')
param collectorHealthDcrName string

@description('Action Group name.')
param actionGroupName string

@description('Short environment label, such as pilot, test, or prod.')
param environment string

@description('Email address for initial Azure Monitor alert notification.')
param operationsEmailAddress string

@description('Azure Arc managed identity principal IDs allowed to submit certificate inventory data.')
param certificateCollectorPrincipalIds array = []

@description('Azure Arc managed identity principal IDs allowed to submit ERP folder metric data.')
param folderCollectorPrincipalIds array = []

@description('Azure Arc managed identity principal IDs allowed to submit collector health data.')
param healthCollectorPrincipalIds array = []

module workspace './modules/log-analytics-workspace.bicep' = {
  name: 'deploy-log-analytics-workspace'
  params: {
    location: location
    workspaceName: logAnalyticsWorkspaceName
    environment: environment
  }
}

module tables './modules/custom-tables.bicep' = {
  name: 'deploy-custom-log-analytics-tables'
  params: {
    workspaceName: logAnalyticsWorkspaceName
    environment: environment
  }
  dependsOn: [
    workspace
  ]
}

module dce './modules/data-collection-endpoint.bicep' = {
  name: 'deploy-data-collection-endpoint'
  params: {
    location: location
    dceName: dataCollectionEndpointName
    environment: environment
  }
}

module dcrs './modules/data-collection-rules.bicep' = {
  name: 'deploy-direct-ingestion-dcrs'
  params: {
    location: location
    workspaceResourceId: workspace.outputs.workspaceResourceId
    dataCollectionEndpointResourceId: dce.outputs.dataCollectionEndpointResourceId
    certificateDcrName: certificateDcrName
    folderMetricsDcrName: folderMetricsDcrName
    collectorHealthDcrName: collectorHealthDcrName
    environment: environment
  }
  dependsOn: [
    workspace
    tables
    dce
  ]
}

module roles './modules/dcr-role-assignments.bicep' = {
  name: 'assign-dcr-ingestion-roles'
  params: {
    certificateDcrResourceId: dcrs.outputs.certificateDcrResourceId
    folderMetricsDcrResourceId: dcrs.outputs.folderMetricsDcrResourceId
    collectorHealthDcrResourceId: dcrs.outputs.collectorHealthDcrResourceId
    certificateCollectorPrincipalIds: certificateCollectorPrincipalIds
    folderCollectorPrincipalIds: folderCollectorPrincipalIds
    healthCollectorPrincipalIds: healthCollectorPrincipalIds
  }
  dependsOn: [
    dcrs
  ]
}

module actionGroup './modules/action-group.bicep' = {
  name: 'deploy-operations-action-group'
  params: {
    location: location
    actionGroupName: actionGroupName
    environment: environment
    operationsEmailAddress: operationsEmailAddress
  }
}

output logAnalyticsWorkspaceResourceId string = workspace.outputs.workspaceResourceId
output logAnalyticsWorkspaceCustomerId string = workspace.outputs.workspaceCustomerId

output dataCollectionEndpointResourceId string = dce.outputs.dataCollectionEndpointResourceId
output logsIngestionEndpoint string = dce.outputs.logsIngestionEndpoint

output certificateDcrResourceId string = dcrs.outputs.certificateDcrResourceId
output certificateDcrImmutableId string = dcrs.outputs.certificateDcrImmutableId

output folderMetricsDcrResourceId string = dcrs.outputs.folderMetricsDcrResourceId
output folderMetricsDcrImmutableId string = dcrs.outputs.folderMetricsDcrImmutableId

output collectorHealthDcrResourceId string = dcrs.outputs.collectorHealthDcrResourceId
output collectorHealthDcrImmutableId string = dcrs.outputs.collectorHealthDcrImmutableId

output actionGroupResourceId string = actionGroup.outputs.actionGroupResourceId
```

---

# 5. Log Analytics Workspace Module

```bicep name=azure/modules/log-analytics-workspace.bicep
@description('Azure region for the Log Analytics Workspace.')
param location string

@description('Log Analytics Workspace name.')
param workspaceName string

@description('Deployment environment.')
param environment string

resource workspace 'Microsoft.OperationalInsights/workspaces@2023-09-01' = {
  name: workspaceName
  location: location
  tags: {
    Application: 'ERP-Monitoring'
    Environment: environment
    ManagedBy: 'Bicep'
    DataClassification: 'Internal'
    SupportTeam: 'ERP-Platform-Support'
  }
  properties: {
    sku: {
      name: 'PerGB2018'
    }
    retentionInDays: 90
    features: {
      enableLogAccessUsingOnlyResourcePermissions: true
      legacy: 0
      searchVersion: 1
    }
  }
}

output workspaceResourceId string = workspace.id
output workspaceCustomerId string = workspace.properties.customerId
```

## Workspace Design Notes

- `PerGB2018` is the standard Log Analytics consumption SKU.
- Initial 90-day retention is appropriate for pilot analysis. Adjust after cost review.
- `enableLogAccessUsingOnlyResourcePermissions` supports Azure RBAC-based data access rather than legacy workspace shared-key access.
- The Phase 1 collectors do **not** use workspace shared keys.

---

# 6. Custom Log Analytics Tables Module

The `outputStream` in each DCR maps to the respective custom table stream:

```text
Custom-ServerCertificateInventory_CL
Custom-ErpFolderMetrics_CL
Custom-CollectorHealth_CL
```

The table names queried in KQL are:

```text
ServerCertificateInventory_CL
ErpFolderMetrics_CL
CollectorHealth_CL
```

```bicep name=azure/modules/custom-tables.bicep
@description('Log Analytics Workspace name.')
param workspaceName string

@description('Deployment environment.')
param environment string

resource workspace 'Microsoft.OperationalInsights/workspaces@2023-09-01' existing = {
  name: workspaceName
}

resource certificateTable 'Microsoft.OperationalInsights/workspaces/tables@2022-10-01' = {
  parent: workspace
  name: 'ServerCertificateInventory_CL'
  properties: {
    plan: 'Analytics'
    retentionInDays: 90
    totalRetentionInDays: 90
    schema: {
      name: 'ServerCertificateInventory_CL'
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
          name: 'StoreLocation'
          type: 'string'
        }
        {
          name: 'StoreName'
          type: 'string'
        }
        {
          name: 'Subject'
          type: 'string'
        }
        {
          name: 'DnsNames'
          type: 'string'
        }
        {
          name: 'Issuer'
          type: 'string'
        }
        {
          name: 'Thumbprint'
          type: 'string'
        }
        {
          name: 'SerialNumber'
          type: 'string'
        }
        {
          name: 'NotBefore'
          type: 'datetime'
        }
        {
          name: 'NotAfter'
          type: 'datetime'
        }
        {
          name: 'DaysUntilExpiry'
          type: 'long'
        }
        {
          name: 'HasPrivateKey'
          type: 'boolean'
        }
        {
          name: 'EnhancedKeyUsage'
          type: 'string'
        }
        {
          name: 'IisBindings'
          type: 'string'
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

resource folderMetricsTable 'Microsoft.OperationalInsights/workspaces/tables@2022-10-01' = {
  parent: workspace
  name: 'ErpFolderMetrics_CL'
  properties: {
    plan: 'Analytics'
    retentionInDays: 90
    totalRetentionInDays: 90
    schema: {
      name: 'ErpFolderMetrics_CL'
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
          name: 'FolderPath'
          type: 'string'
        }
        {
          name: 'TotalBytes'
          type: 'long'
        }
        {
          name: 'TotalGB'
          type: 'real'
        }
        {
          name: 'FileCount'
          type: 'long'
        }
        {
          name: 'OldestFileUtc'
          type: 'datetime'
        }
        {
          name: 'NewestFileUtc'
          type: 'datetime'
        }
        {
          name: 'FilesOlderThanRetention'
          type: 'long'
        }
        {
          name: 'RetentionDays'
          type: 'long'
        }
        {
          name: 'AccessErrorCount'
          type: 'long'
        }
        {
          name: 'ScanDurationSeconds'
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

resource collectorHealthTable 'Microsoft.OperationalInsights/workspaces/tables@2022-10-01' = {
  parent: workspace
  name: 'CollectorHealth_CL'
  properties: {
    plan: 'Analytics'
    retentionInDays: 90
    totalRetentionInDays: 90
    schema: {
      name: 'CollectorHealth_CL'
      columns: [
        {
          name: 'TimeGenerated'
          type: 'datetime'
        }
        {
          name: 'CollectorName'
          type: 'string'
        }
        {
          name: 'Computer'
          type: 'string'
        }
        {
          name: 'TargetScope'
          type: 'string'
        }
        {
          name: 'ExecutionId'
          type: 'string'
        }
        {
          name: 'RecordsCollected'
          type: 'long'
        }
        {
          name: 'RecordsSubmitted'
          type: 'long'
        }
        {
          name: 'DurationSeconds'
          type: 'real'
        }
        {
          name: 'Status'
          type: 'string'
        }
        {
          name: 'ErrorCode'
          type: 'string'
        }
        {
          name: 'ErrorMessage'
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

output certificateTableName string = certificateTable.name
output folderMetricsTableName string = folderMetricsTable.name
output collectorHealthTableName string = collectorHealthTable.name
```

## Table-Design Notes

1. `TimeGenerated` is an explicit field sent by the collectors and stored as a `datetime`.
2. Field names are intentionally aligned with the Phase 1 JSON records.
3. Do not add full file paths per file, certificate private keys, secrets, or raw exceptions.
4. The `ErrorMessage` field must receive only sanitized error output.
5. Schema changes should be additive wherever possible. Avoid renaming/removing columns after production rollout.

---

# 7. Data Collection Endpoint Module

```bicep name=azure/modules/data-collection-endpoint.bicep
@description('Azure region for the Data Collection Endpoint.')
param location string

@description('Data Collection Endpoint name.')
param dceName string

@description('Deployment environment.')
param environment string

resource dce 'Microsoft.Insights/dataCollectionEndpoints@2023-03-11' = {
  name: dceName
  location: location
  tags: {
    Application: 'ERP-Monitoring'
    Environment: environment
    ManagedBy: 'Bicep'
    DataClassification: 'Internal'
    SupportTeam: 'ERP-Platform-Support'
  }
  properties: {
    networkAcls: {
      publicNetworkAccess: 'Enabled'
    }
  }
}

output dataCollectionEndpointResourceId string = dce.id
output logsIngestionEndpoint string = dce.properties.logsIngestion.endpoint
```

## DCE Notes

The pilot uses:

```text
publicNetworkAccess: Enabled
```

This aligns with the approved outbound public HTTPS model.

For a later Private Link design, the DCE can become part of a private monitoring architecture, but do not add Private Link complexity to the initial proof of concept unless required by policy.

---

# 8. Data Collection Rules Module

The DCR has three functions:

1. **Stream declaration** — defines the accepted JSON schema.
2. **Data flow** — identifies the input stream, applies a transformation, and routes it to Log Analytics.
3. **Destination** — points to the target Log Analytics Workspace.

The input stream contains `Raw` in its name:

```text
Custom-ServerCertificateInventoryRaw
```

The DCR output stream maps to the custom table:

```text
Custom-ServerCertificateInventory_CL
```

```bicep name=azure/modules/data-collection-rules.bicep
@description('Azure region for DCR resources.')
param location string

@description('Resource ID of the target Log Analytics Workspace.')
param workspaceResourceId string

@description('Resource ID of the Data Collection Endpoint.')
param dataCollectionEndpointResourceId string

@description('Certificate inventory DCR name.')
param certificateDcrName string

@description('ERP folder metrics DCR name.')
param folderMetricsDcrName string

@description('Collector health DCR name.')
param collectorHealthDcrName string

@description('Deployment environment.')
param environment string

resource certificateDcr 'Microsoft.Insights/dataCollectionRules@2023-03-11' = {
  name: certificateDcrName
  location: location
  kind: 'Direct'
  tags: {
    Application: 'ERP-Monitoring'
    Environment: environment
    ManagedBy: 'Bicep'
    DataClassification: 'Internal'
    Collector: 'CertificateInventory'
  }
  properties: {
    dataCollectionEndpointId: dataCollectionEndpointResourceId
    streamDeclarations: {
      'Custom-ServerCertificateInventoryRaw': {
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
            name: 'StoreLocation'
            type: 'string'
          }
          {
            name: 'StoreName'
            type: 'string'
          }
          {
            name: 'Subject'
            type: 'string'
          }
          {
            name: 'DnsNames'
            type: 'string'
          }
          {
            name: 'Issuer'
            type: 'string'
          }
          {
            name: 'Thumbprint'
            type: 'string'
          }
          {
            name: 'SerialNumber'
            type: 'string'
          }
          {
            name: 'NotBefore'
            type: 'datetime'
          }
          {
            name: 'NotAfter'
            type: 'datetime'
          }
          {
            name: 'DaysUntilExpiry'
            type: 'long'
          }
          {
            name: 'HasPrivateKey'
            type: 'boolean'
          }
          {
            name: 'EnhancedKeyUsage'
            type: 'string'
          }
          {
            name: 'IisBindings'
            type: 'string'
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
          name: 'certificateWorkspace'
          workspaceResourceId: workspaceResourceId
        }
      ]
    }
    dataFlows: [
      {
        streams: [
          'Custom-ServerCertificateInventoryRaw'
        ]
        destinations: [
          'certificateWorkspace'
        ]
        transformKql: 'source'
        outputStream: 'Custom-ServerCertificateInventory_CL'
      }
    ]
  }
}

resource folderMetricsDcr 'Microsoft.Insights/dataCollectionRules@2023-03-11' = {
  name: folderMetricsDcrName
  location: location
  kind: 'Direct'
  tags: {
    Application: 'ERP-Monitoring'
    Environment: environment
    ManagedBy: 'Bicep'
    DataClassification: 'Internal'
    Collector: 'ErpFolderMetrics'
  }
  properties: {
    dataCollectionEndpointId: dataCollectionEndpointResourceId
    streamDeclarations: {
      'Custom-ErpFolderMetricsRaw': {
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
            name: 'FolderPath'
            type: 'string'
          }
          {
            name: 'TotalBytes'
            type: 'long'
          }
          {
            name: 'TotalGB'
            type: 'real'
          }
          {
            name: 'FileCount'
            type: 'long'
          }
          {
            name: 'OldestFileUtc'
            type: 'datetime'
          }
          {
            name: 'NewestFileUtc'
            type: 'datetime'
          }
          {
            name: 'FilesOlderThanRetention'
            type: 'long'
          }
          {
            name: 'RetentionDays'
            type: 'long'
          }
          {
            name: 'AccessErrorCount'
            type: 'long'
          }
          {
            name: 'ScanDurationSeconds'
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
          name: 'folderMetricsWorkspace'
          workspaceResourceId: workspaceResourceId
        }
      ]
    }
    dataFlows: [
      {
        streams: [
          'Custom-ErpFolderMetricsRaw'
        ]
        destinations: [
          'folderMetricsWorkspace'
        ]
        transformKql: 'source'
        outputStream: 'Custom-ErpFolderMetrics_CL'
      }
    ]
  }
}

resource collectorHealthDcr 'Microsoft.Insights/dataCollectionRules@2023-03-11' = {
  name: collectorHealthDcrName
  location: location
  kind: 'Direct'
  tags: {
    Application: 'ERP-Monitoring'
    Environment: environment
    ManagedBy: 'Bicep'
    DataClassification: 'Internal'
    Collector: 'CollectorHealth'
  }
  properties: {
    dataCollectionEndpointId: dataCollectionEndpointResourceId
    streamDeclarations: {
      'Custom-CollectorHealthRaw': {
        columns: [
          {
            name: 'TimeGenerated'
            type: 'datetime'
          }
          {
            name: 'CollectorName'
            type: 'string'
          }
          {
            name: 'Computer'
            type: 'string'
          }
          {
            name: 'TargetScope'
            type: 'string'
          }
          {
            name: 'ExecutionId'
            type: 'string'
          }
          {
            name: 'RecordsCollected'
            type: 'long'
          }
          {
            name: 'RecordsSubmitted'
            type: 'long'
          }
          {
            name: 'DurationSeconds'
            type: 'real'
          }
          {
            name: 'Status'
            type: 'string'
          }
          {
            name: 'ErrorCode'
            type: 'string'
          }
          {
            name: 'ErrorMessage'
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
          name: 'collectorHealthWorkspace'
          workspaceResourceId: workspaceResourceId
        }
      ]
    }
    dataFlows: [
      {
        streams: [
          'Custom-CollectorHealthRaw'
        ]
        destinations: [
          'collectorHealthWorkspace'
        ]
        transformKql: 'source'
        outputStream: 'Custom-CollectorHealth_CL'
      }
    ]
  }
}

output certificateDcrResourceId string = certificateDcr.id
output certificateDcrImmutableId string = certificateDcr.properties.immutableId

output folderMetricsDcrResourceId string = folderMetricsDcr.id
output folderMetricsDcrImmutableId string = folderMetricsDcr.properties.immutableId

output collectorHealthDcrResourceId string = collectorHealthDcr.id
output collectorHealthDcrImmutableId string = collectorHealthDcr.properties.immutableId
```

---

# 9. DCR Role Assignment Module

The collector needs permission to submit telemetry to the DCR. Use the **Monitoring Metrics Publisher** built-in Azure role at the DCR scope.

> Although its name says “Metrics Publisher,” this built-in role is the role Microsoft documents for authorizing identity-based Logs Ingestion API submission through a DCR. Scope it to the individual DCR—not the full resource group, subscription, or workspace.

```bicep name=azure/modules/dcr-role-assignments.bicep
@description('Certificate DCR resource ID.')
param certificateDcrResourceId string

@description('Folder metrics DCR resource ID.')
param folderMetricsDcrResourceId string

@description('Collector health DCR resource ID.')
param collectorHealthDcrResourceId string

@description('Azure Arc managed identity principal IDs for certificate collector hosts.')
param certificateCollectorPrincipalIds array

@description('Azure Arc managed identity principal IDs for ERP folder collector hosts.')
param folderCollectorPrincipalIds array

@description('Azure Arc managed identity principal IDs for collector-health submitters.')
param healthCollectorPrincipalIds array

// Built-in role: Monitoring Metrics Publisher
var monitoringMetricsPublisherRoleDefinitionId = subscriptionResourceId(
  'Microsoft.Authorization/roleDefinitions'
  '3913510d-42f4-4e42-8a64-420c390055eb'
)

resource certificateDcrRoleAssignments 'Microsoft.Authorization/roleAssignments@2022-04-01' = [
  for principalId in certificateCollectorPrincipalIds: {
    name: guid(
      certificateDcrResourceId
      principalId
      monitoringMetricsPublisherRoleDefinitionId
    )
    scope: resourceId(
      'Microsoft.Insights/dataCollectionRules'
      last(split(certificateDcrResourceId, '/'))
    )
    properties: {
      roleDefinitionId: monitoringMetricsPublisherRoleDefinitionId
      principalId: principalId
      principalType: 'ServicePrincipal'
    }
  }
]

resource folderMetricsDcrRoleAssignments 'Microsoft.Authorization/roleAssignments@2022-04-01' = [
  for principalId in folderCollectorPrincipalIds: {
    name: guid(
      folderMetricsDcrResourceId
      principalId
      monitoringMetricsPublisherRoleDefinitionId
    )
    scope: resourceId(
      'Microsoft.Insights/dataCollectionRules'
      last(split(folderMetricsDcrResourceId, '/'))
    )
    properties: {
      roleDefinitionId: monitoringMetricsPublisherRoleDefinitionId
      principalId: principalId
      principalType: 'ServicePrincipal'
    }
  }
]

resource collectorHealthDcrRoleAssignments 'Microsoft.Authorization/roleAssignments@2022-04-01' = [
  for principalId in healthCollectorPrincipalIds: {
    name: guid(
      collectorHealthDcrResourceId
      principalId
      monitoringMetricsPublisherRoleDefinitionId
    )
    scope: resourceId(
      'Microsoft.Insights/dataCollectionRules'
      last(split(collectorHealthDcrResourceId, '/'))
    )
    properties: {
      roleDefinitionId: monitoringMetricsPublisherRoleDefinitionId
      principalId: principalId
      principalType: 'ServicePrincipal'
    }
  }
]
```

## Important Implementation Note

The `scope` expression assumes the DCRs are deployed into the same resource group as the parent deployment. This is the recommended pilot pattern.

If future production DCRs are deployed to a centralized monitoring resource group, revise this module to use an `existing` DCR resource scoped to that resource group.

---

# 10. Action Group Module

For the initial pilot, configure email directly. Teams should be connected through an approved Logic App workflow or the organization’s approved Teams notification mechanism.

```bicep name=azure/modules/action-group.bicep
@description('Azure region for the Action Group resource.')
param location string

@description('Action Group name.')
param actionGroupName string

@description('Deployment environment.')
param environment string

@description('Email address or distribution list for operations alerts.')
param operationsEmailAddress string

resource actionGroup 'Microsoft.Insights/actionGroups@2023-01-01' = {
  name: actionGroupName
  location: 'global'
  tags: {
    Application: 'ERP-Monitoring'
    Environment: environment
    ManagedBy: 'Bicep'
    SupportTeam: 'ERP-Platform-Support'
  }
  properties: {
    groupShortName: 'ERPMonOps'
    enabled: true
    emailReceivers: [
      {
        name: 'OperationsEmail'
        emailAddress: operationsEmailAddress
        useCommonAlertSchema: true
      }
    ]
  }
}

output actionGroupResourceId string = actionGroup.id
```

Phase 5 will add:

```text
Azure Monitor Alert
    → Action Group
        → Logic App
            → Microsoft Teams channel
```

This avoids hard-coding obsolete Teams webhook patterns into the monitoring design.

---

# 11. Pilot Parameter File

Update the region once approved. The principal IDs must be taken from the Azure Arc-enabled server identity pages or retrieved by Azure CLI/PowerShell.

```bicep name=azure/parameters/pilot.bicepparam
using '../main.bicep'

param location = 'canadacentral'

param environment = 'pilot'

param logAnalyticsWorkspaceName = 'law-erpmon-pilot-cac-001'

param dataCollectionEndpointName = 'dce-erpmon-pilot-cac-001'

param certificateDcrName = 'dcr-erpmon-cert-pilot-001'

param folderMetricsDcrName = 'dcr-erpmon-folder-pilot-001'

param collectorHealthDcrName = 'dcr-erpmon-health-pilot-001'

param actionGroupName = 'ag-erpmon-ops-pilot-001'

param operationsEmailAddress = 'erp-platform-support@example.com'

// Replace placeholders with object/principal IDs of the Azure Arc system-assigned
// managed identities for the actual collector hosts.
param certificateCollectorPrincipalIds = [
  '00000000-0000-0000-0000-000000000001'
]

param folderCollectorPrincipalIds = [
  '00000000-0000-0000-0000-000000000002'
]

param healthCollectorPrincipalIds = [
  '00000000-0000-0000-0000-000000000001'
  '00000000-0000-0000-0000-000000000002'
]
```

---

# 12. Retrieve Azure Arc Managed-Identity Principal IDs

Run from an administrator workstation with Azure CLI and rights to read Azure Arc resources:

```powershell name=Get-ArcManagedIdentityPrincipalIds.ps1
param(
    [Parameter(Mandatory)]
    [string] $ResourceGroupName,

    [Parameter(Mandatory)]
    [string[]] $ArcMachineNames
)

foreach ($machineName in $ArcMachineNames) {
    $machine = az connectedmachine show `
        --resource-group $ResourceGroupName `
        --name $machineName `
        --query '{Name:name, PrincipalId:identity.principalId, TenantId:identity.tenantId}' `
        --output json |
        ConvertFrom-Json

    [pscustomobject]@{
        ArcMachineName = $machine.Name
        PrincipalId    = $machine.PrincipalId
        TenantId       = $machine.TenantId
    }
}
```

Expected result:

```text
ArcMachineName     PrincipalId
--------------     ------------------------------------
ERP-MW-01          <certificate-server-principal-id>
ERP-APP-01         <erp-folder-server-principal-id>
```

Use the `PrincipalId` values in the Bicep parameter file.

---

# 13. Validate and Deploy Bicep

## 13.1 Validate

```powershell name=Deploy-Phase2AzureMonitoring.ps1
param(
    [Parameter(Mandatory)]
    [string] $SubscriptionId,

    [Parameter(Mandatory)]
    [string] $ResourceGroupName,

    [Parameter(Mandatory)]
    [string] $TemplateFile,

    [Parameter(Mandatory)]
    [string] $ParameterFile
)

az account set --subscription $SubscriptionId

az deployment group validate `
    --resource-group $ResourceGroupName `
    --template-file $TemplateFile `
    --parameters $ParameterFile
```

## 13.2 What-if Review

```powershell name=WhatIf-Phase2AzureMonitoring.ps1
param(
    [Parameter(Mandatory)]
    [string] $ResourceGroupName,

    [Parameter(Mandatory)]
    [string] $TemplateFile,

    [Parameter(Mandatory)]
    [string] $ParameterFile
)

az deployment group what-if `
    --resource-group $ResourceGroupName `
    --template-file $TemplateFile `
    --parameters $ParameterFile `
    --result-format FullResourcePayloads
```

Review that only intended resources are created or updated:

- Log Analytics Workspace.
- Three custom tables.
- One DCE.
- Three DCRs.
- DCR-scoped role assignments.
- One Action Group.

## 13.3 Deploy

```powershell name=Invoke-Phase2AzureMonitoringDeployment.ps1
param(
    [Parameter(Mandatory)]
    [string] $ResourceGroupName,

    [Parameter(Mandatory)]
    [string] $TemplateFile,

    [Parameter(Mandatory)]
    [string] $ParameterFile
)

az deployment group create `
    --name "erpmon-phase2-$(Get-Date -Format 'yyyyMMddHHmmss')" `
    --resource-group $ResourceGroupName `
    --template-file $TemplateFile `
    --parameters $ParameterFile `
    --output json
```

Capture deployment outputs:

```powershell name=Get-Phase2DeploymentOutputs.ps1
param(
    [Parameter(Mandatory)]
    [string] $ResourceGroupName,

    [Parameter(Mandatory)]
    [string] $DeploymentName
)

az deployment group show `
    --resource-group $ResourceGroupName `
    --name $DeploymentName `
    --query properties.outputs `
    --output json
```

---

# 14. Update Phase 1 Collector Configurations

After deployment, use the Bicep outputs.

## Certificate Configuration

```json name=CertificateInventory.config.json
{
  "SchemaVersion": "1.0",
  "CollectorName": "CertificateInventory",
  "Enabled": true,
  "IngestionEndpoint": "https://<logs-ingestion-endpoint-from-bicep-output>",
  "DcrImmutableId": "dcr-<certificate-dcr-immutable-id-from-bicep-output>",
  "StreamName": "Custom-ServerCertificateInventoryRaw",
  "HealthIngestionEndpoint": "https://<logs-ingestion-endpoint-from-bicep-output>",
  "HealthDcrImmutableId": "dcr-<collector-health-dcr-immutable-id-from-bicep-output>",
  "HealthStreamName": "Custom-CollectorHealthRaw",
  "CertificateStores": [
    "Cert:\\LocalMachine\\WebHosting"
  ],
  "EnableIisBindingDiscovery": true,
  "ExpiryWarningDays": 30,
  "ExpiryCriticalDays": 14
}
```

## Folder Metrics Configuration

```json name=ErpFolderMetrics.config.json
{
  "SchemaVersion": "1.0",
  "CollectorName": "ErpFolderMetrics",
  "Enabled": true,
  "IngestionEndpoint": "https://<logs-ingestion-endpoint-from-bicep-output>",
  "DcrImmutableId": "dcr-<folder-metrics-dcr-immutable-id-from-bicep-output>",
  "StreamName": "Custom-ErpFolderMetricsRaw",
  "HealthIngestionEndpoint": "https://<logs-ingestion-endpoint-from-bicep-output>",
  "HealthDcrImmutableId": "dcr-<collector-health-dcr-immutable-id-from-bicep-output>",
  "HealthStreamName": "Custom-CollectorHealthRaw",
  "DefaultRetentionDays": 30,
  "MaximumScanDurationMinutes": 20,
  "Folders": [
    {
      "Application": "ERP",
      "Path": "D:\\ERP\\Logs",
      "RetentionDays": 30,
      "Enabled": true
    }
  ]
}
```

---

# 15. Direct Ingestion Validation Script

Run this on the actual Azure Arc-enabled certificate collector server under the same account that Task Scheduler will use.

This validates:

1. HIMDS access.
2. Arc managed-identity token retrieval.
3. DCR-scoped RBAC.
4. DCE HTTPS connectivity.
5. DCR stream schema.
6. Table routing.

```powershell name=Test-CertificateDirectIngestion.ps1
[CmdletBinding()]
param(
    [Parameter(Mandatory)]
    [string] $IngestionEndpoint,

    [Parameter(Mandatory)]
    [string] $DcrImmutableId,

    [string] $StreamName = 'Custom-ServerCertificateInventoryRaw'
)

Set-StrictMode -Version Latest
$ErrorActionPreference = 'Stop'

function Get-ArcAccessToken {
    param(
        [string] $Resource = 'https://monitor.azure.com/',
        [string] $ApiVersion = '2020-06-01'
    )

    if ([string]::IsNullOrWhiteSpace($env:IDENTITY_ENDPOINT)) {
        throw 'IDENTITY_ENDPOINT is missing. Azure Arc managed identity is not available.'
    }

    $resourceEncoded = [uri]::EscapeDataString($Resource)

    $tokenUri = '{0}?resource={1}&api-version={2}' -f `
        $env:IDENTITY_ENDPOINT,
        $resourceEncoded,
        $ApiVersion

    $secretFilePath = $null

    try {
        Invoke-WebRequest `
            -Method Get `
            -Uri $tokenUri `
            -Headers @{ Metadata = 'True' } `
            -UseBasicParsing `
            -ErrorAction Stop | Out-Null
    }
    catch {
        $challenge = $_.Exception.Response.Headers['WWW-Authenticate']

        if ($challenge -notmatch 'Basic\s+realm=(.+)$') {
            throw 'The expected Azure Arc HIMDS authentication challenge was not received.'
        }

        $secretFilePath = $Matches[1].Trim('"')
    }

    if (-not (Test-Path -LiteralPath $secretFilePath -PathType Leaf)) {
        throw "HIMDS secret challenge file was not found: $secretFilePath"
    }

    $secret = Get-Content -LiteralPath $secretFilePath -Raw -Encoding UTF8

    $response = Invoke-RestMethod `
        -Method Get `
        -Uri $tokenUri `
        -Headers @{
            Metadata      = 'True'
            Authorization = "Basic $secret"
        } `
        -ErrorAction Stop

    return $response.access_token
}

$record = @{
    TimeGenerated    = (Get-Date).ToUniversalTime().ToString('o')
    Computer         = $env:COMPUTERNAME
    Environment      = 'Pilot'
    Application      = 'ERP'
    StoreLocation    = 'LocalMachine'
    StoreName        = 'WebHosting'
    Subject          = 'CN=AzureMonitorIngestionValidation'
    DnsNames         = 'validation.invalid'
    Issuer           = 'CN=AzureMonitorIngestionValidation'
    Thumbprint       = ('VALIDATION-' + [guid]::NewGuid().ToString('N'))
    SerialNumber     = 'VALIDATION'
    NotBefore        = (Get-Date).ToUniversalTime().ToString('o')
    NotAfter         = (Get-Date).ToUniversalTime().AddDays(30).ToString('o')
    DaysUntilExpiry  = 30
    HasPrivateKey    = $false
    EnhancedKeyUsage = 'ValidationOnly'
    IisBindings      = $null
    CollectionStatus = 'Validation'
    CollectorVersion = '1.0.0'
}

$uri = '{0}/dataCollectionRules/{1}/streams/{2}?api-version=2023-01-01' -f `
    $IngestionEndpoint.TrimEnd('/'),
    [uri]::EscapeDataString($DcrImmutableId),
    [uri]::EscapeDataString($StreamName)

$token = Get-ArcAccessToken
$payload = @($record) | ConvertTo-Json -Depth 8 -Compress

Invoke-RestMethod `
    -Method Post `
    -Uri $uri `
    -Headers @{ Authorization = "Bearer $token" } `
    -ContentType 'application/json; charset=utf-8' `
    -Body ([Text.Encoding]::UTF8.GetBytes($payload)) `
    -ErrorAction Stop | Out-Null

Write-Output 'Direct ingestion request was accepted successfully.'
Write-Output "Validation thumbprint: $($record.Thumbprint)"
```

> A successful `POST` means the Azure Monitor ingestion service accepted the request. It can take several minutes before the record becomes queryable in Log Analytics.

---

# 16. KQL Validation Queries

## 16.1 Validate Certificate Ingestion

```kusto name=kql/validation/Validate-CertificateIngestion.kql
ServerCertificateInventory_CL
| where TimeGenerated > ago(30m)
| where CollectionStatus == "Validation"
| project
    TimeGenerated,
    Computer,
    Thumbprint,
    Subject,
    CollectionStatus,
    CollectorVersion
| order by TimeGenerated desc
```

## 16.2 Validate Folder Metrics Ingestion

```kusto name=kql/validation/Validate-FolderMetricsIngestion.kql
ErpFolderMetrics_CL
| where TimeGenerated > ago(30m)
| project
    TimeGenerated,
    Computer,
    Application,
    FolderPath,
    TotalGB,
    FileCount,
    CollectionStatus,
    ScanDurationSeconds
| order by TimeGenerated desc
```

## 16.3 Validate Collector Health Ingestion

```kusto name=kql/validation/Validate-CollectorHealthIngestion.kql
CollectorHealth_CL
| where TimeGenerated > ago(30m)
| project
    TimeGenerated,
    CollectorName,
    Computer,
    Status,
    RecordsCollected,
    RecordsSubmitted,
    DurationSeconds,
    ErrorCode
| order by TimeGenerated desc
```

---

# 17. Troubleshooting Matrix

| Symptom | Likely cause | Technical response |
|---|---|---|
| `IDENTITY_ENDPOINT` missing | Arc identity not enabled, or environment unavailable to task account | Confirm Arc machine identity enabled; run validation under task account. |
| HIMDS returns challenge but secret file unavailable | Task account lacks local HIMDS authorization | Validate membership in `Hybrid Agent Extension Applications`; confirm local group policy. |
| HTTP 401 from Logs Ingestion API | Invalid token audience or token issue | Confirm token uses `https://monitor.azure.com/`; verify server clock. |
| HTTP 403 from Logs Ingestion API | Arc identity lacks DCR-scoped role | Confirm principal ID and `Monitoring Metrics Publisher` assignment at correct DCR scope. |
| HTTP 404 | Incorrect endpoint, DCR immutable ID, stream name, or API path | Use deployment outputs; do not use Azure Resource Manager DCR ID instead of immutable ID. |
| HTTP 400 | Payload type/schema mismatch | Compare JSON properties and types against DCR `streamDeclarations`. |
| POST succeeds but no data appears | Ingestion delay, wrong workspace/table, DCR transformation mismatch | Wait several minutes; inspect DCR destination/output stream; run validation KQL. |
| Records appear in wrong table | Incorrect `outputStream` | Verify DCR data flow uses `Custom-<TableName>_CL`. |
| Certificate task completes but no IIS bindings | IIS module unavailable or account lacks IIS configuration access | Validate `WebAdministration`; test `Get-WebBinding` under run-as identity. |
| Folder collector causes long execution | Too many files, large archive tree, access failures | Increase interval; configure exclusions; set timeout; consider directory-level storage metrics. |

---

# 18. Phase 2 Acceptance Criteria

Phase 2 is complete when:

| Requirement | Acceptance evidence |
|---|---|
| Workspace deployed | `law-erpmon-pilot-...` exists with intended retention. |
| Tables deployed | All three `_CL` tables exist with correct typed columns. |
| DCE deployed | Public logs-ingestion endpoint is available. |
| DCRs deployed | Three Direct DCRs have expected stream declarations and workspace destinations. |
| Role assignment deployed | Each Arc server identity has DCR-scoped `Monitoring Metrics Publisher` authorization. |
| Certificate test POST accepted | Direct ingestion validation script returns success. |
| KQL validation succeeds | Validation record is queryable in `ServerCertificateInventory_CL`. |
| Folder ingestion test succeeds | Folder record appears in `ErpFolderMetrics_CL`. |
| Health ingestion test succeeds | Collector health record appears in `CollectorHealth_CL`. |
| No secrets used | No workspace shared key, client secret, or certificate credential exists in collector configurations. |
| Bicep deployment repeatable | `validate` and `what-if` produce expected results on redeployment. |

---

# 19. Next Phase

**Phase 3** will finalize the certificate-expiration monitoring use case:

- Harden the certificate collector.
- Validate IIS binding correlation.
- Deploy and test the certificate Scheduled Task.
- Add KQL inventory, expired, 60/30/14/7-day expiry, and stale-data queries.
- Deploy certificate scheduled-query alerts.
- Build certificate Azure Workbook sections.
- Wire alerts to Outlook/email and Teams through the approved notification architecture.

### Want me to…
<!-- copilot:suggestions -->
1. Continue to Phase 3 with the complete certificate monitoring implementation, KQL queries, alerts, and Workbook dashboard design.
2. Provide a companion Azure CLI/PowerShell deployment script that packages, validates, and deploys this Phase 2 Bicep solution.
3. Hand this Bicep and PowerShell implementation off to the coding agent to open a pull request in a repository you provide.
