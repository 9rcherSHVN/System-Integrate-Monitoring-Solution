# 1. JSON Payload Schema for Custom Data Streams

For Azure Monitor **Logs Ingestion API**, the PowerShell collector submits a **JSON array of records** to a DCR custom input stream.

Conceptually:

```text
PowerShell object(s)
   → ConvertTo-Json
   → JSON array payload
   → Logs Ingestion API
   → DCR input stream
   → DCR transformation
   → Log Analytics custom table
```

The submitted JSON must match the **DCR input stream declaration**, not necessarily the final table schema. However, for this pilot, use a direct one-to-one mapping where possible:

```text
PowerShell JSON field
        =
DCR stream field
        =
Log Analytics custom-table column
```

This keeps the solution supportable.

---

## 1.1 General API Payload Structure

The Logs Ingestion API requires an array, including where there is only one record.

```json name=generic-ingestion-payload.json
[
  {
    "TimeGenerated": "2026-09-21T15:00:00Z",
    "Computer": "ERP-APP-01",
    "CollectionStatus": "Success",
    "CollectorVersion": "1.0.0"
  }
]
```

### Key rules

| Rule | Requirement |
|---|---|
| Root JSON type | Must be an array: `[...]` |
| Individual record | JSON object: `{...}` |
| Timestamp format | ISO 8601 UTC, for example `2026-09-21T15:00:00Z` |
| Numeric fields | Send as JSON numbers, not quoted strings |
| Boolean fields | Send as `true` or `false`, not `"true"` or `"false"` |
| Nullable fields | Use `null` where the field is optional or unavailable |
| Property names | Must match DCR stream field names exactly in spelling and case conventions used by the implementation |
| Payload size | Batch payloads conservatively; do not exceed Azure Monitor Logs Ingestion API request limits |
| Sensitive data | Do not submit passwords, LAPS passwords, secrets, access tokens, private keys, or raw sensitive business payloads |

---

## 1.2 Data-Type Mapping

| Azure Monitor / DCR type | JSON representation | Example |
|---|---|---|
| `string` | JSON string | `"ERP-APP-01"` |
| `datetime` | ISO 8601 string | `"2026-09-21T15:00:00Z"` |
| `long` | JSON integer | `14` |
| `real` | JSON decimal number | `5.25` |
| `boolean` | JSON Boolean | `true` |
| `dynamic` | JSON object or array | `{"key":"value"}` |

For this pilot, use `dynamic` sparingly. Explicit columns are easier to query, alert on, document, and govern.

---

# 2. Recommended DCR Stream and Table Naming

Use a distinct **input stream** and **destination table**.

| Collector | DCR input stream | Log Analytics table |
|---|---|---|
| Certificate inventory | `Custom-ServerCertificateInventoryRaw` | `ServerCertificateInventory_CL` |
| Service account status | `Custom-ServiceAccountStatusRaw` | `ServiceAccountStatus_CL` |
| Entra-backed LAPS metadata | `Custom-LapsDeviceStatusRaw` | `LapsDeviceStatus_CL` |
| ERP folder metrics | `Custom-ErpFolderMetricsRaw` | `ErpFolderMetrics_CL` |
| Collector health | `Custom-CollectorHealthRaw` | `CollectorHealth_CL` |

The `Raw` suffix is optional but recommended because it distinguishes the **submitted source stream** from the final Log Analytics table.

The destination output stream convention is normally:

```text
Custom-<TableName>_CL
```

For example:

```text
Custom-ErpFolderMetrics_CL
```

---

# 3. Certificate Inventory Payload and Schema

## 3.1 JSON Payload Example

```json name=server-certificate-inventory-payload.json
[
  {
    "TimeGenerated": "2026-09-21T15:00:00Z",
    "Computer": "ERP-MW-01",
    "Environment": "Pilot",
    "Application": "Middleware",
    "StoreLocation": "LocalMachine",
    "StoreName": "WebHosting",
    "Subject": "CN=api.erp.example.com",
    "DnsNames": "api.erp.example.com;middleware.erp.example.com",
    "Issuer": "CN=Enterprise Issuing CA",
    "Thumbprint": "AABBCCDDEEFF00112233445566778899AABBCCDD",
    "SerialNumber": "1234567890",
    "NotBefore": "2026-01-01T00:00:00Z",
    "NotAfter": "2026-10-15T23:59:59Z",
    "DaysUntilExpiry": 24,
    "HasPrivateKey": true,
    "EnhancedKeyUsage": "Server Authentication",
    "IisBindings": "ERP API|*:443:api.erp.example.com|My",
    "CollectionStatus": "Success",
    "CollectorVersion": "1.0.0"
  }
]
```

## 3.2 DCR Input Stream Schema

```json name=certificate-stream-declaration.json
{
  "Custom-ServerCertificateInventoryRaw": {
    "columns": [
      { "name": "TimeGenerated", "type": "datetime" },
      { "name": "Computer", "type": "string" },
      { "name": "Environment", "type": "string" },
      { "name": "Application", "type": "string" },
      { "name": "StoreLocation", "type": "string" },
      { "name": "StoreName", "type": "string" },
      { "name": "Subject", "type": "string" },
      { "name": "DnsNames", "type": "string" },
      { "name": "Issuer", "type": "string" },
      { "name": "Thumbprint", "type": "string" },
      { "name": "SerialNumber", "type": "string" },
      { "name": "NotBefore", "type": "datetime" },
      { "name": "NotAfter", "type": "datetime" },
      { "name": "DaysUntilExpiry", "type": "long" },
      { "name": "HasPrivateKey", "type": "boolean" },
      { "name": "EnhancedKeyUsage", "type": "string" },
      { "name": "IisBindings", "type": "string" },
      { "name": "CollectionStatus", "type": "string" },
      { "name": "CollectorVersion", "type": "string" }
    ]
  }
}
```

---

# 4. Service-Account Status Payload and Schema

This collector inventories the account used by a Windows service, IIS application pool, or scheduled task. It does **not** retrieve or submit password values.

## 4.1 JSON Payload Example

```json name=service-account-status-payload.json
[
  {
    "TimeGenerated": "2026-09-21T15:00:00Z",
    "Computer": "ERP-APP-01",
    "UsageType": "WindowsService",
    "UsageName": "ErpApplicationService",
    "DisplayName": "ERP Business Application Service",
    "ConfiguredAccount": "CONTOSO\\svc_erpapp",
    "AccountType": "DomainUser",
    "SamAccountName": "svc_erpapp",
    "DistinguishedName": "CN=svc_erpapp,OU=Service Accounts,DC=contoso,DC=com",
    "Enabled": true,
    "LockedOut": false,
    "PasswordLastSet": "2026-08-22T09:00:00Z",
    "PasswordExpires": "2026-10-21T09:00:00Z",
    "DaysUntilExpiry": 30,
    "PasswordNeverExpires": false,
    "ManagedPassword": false,
    "ResolutionStatus": "Resolved",
    "CollectionStatus": "Success",
    "CollectorVersion": "1.0.0"
  },
  {
    "TimeGenerated": "2026-09-21T15:00:00Z",
    "Computer": "ERP-MW-01",
    "UsageType": "IISApplicationPool",
    "UsageName": "ErpMiddlewarePool",
    "DisplayName": "ErpMiddlewarePool",
    "ConfiguredAccount": "CONTOSO\\gmsa_erpweb$",
    "AccountType": "gMSA",
    "SamAccountName": "gmsa_erpweb$",
    "DistinguishedName": "CN=gmsa_erpweb,OU=Managed Service Accounts,DC=contoso,DC=com",
    "Enabled": true,
    "LockedOut": false,
    "PasswordLastSet": null,
    "PasswordExpires": null,
    "DaysUntilExpiry": null,
    "PasswordNeverExpires": true,
    "ManagedPassword": true,
    "ResolutionStatus": "Resolved",
    "CollectionStatus": "Success",
    "CollectorVersion": "1.0.0"
  }
]
```

## 4.2 DCR Input Stream Schema

```json name=service-account-stream-declaration.json
{
  "Custom-ServiceAccountStatusRaw": {
    "columns": [
      { "name": "TimeGenerated", "type": "datetime" },
      { "name": "Computer", "type": "string" },
      { "name": "UsageType", "type": "string" },
      { "name": "UsageName", "type": "string" },
      { "name": "DisplayName", "type": "string" },
      { "name": "ConfiguredAccount", "type": "string" },
      { "name": "AccountType", "type": "string" },
      { "name": "SamAccountName", "type": "string" },
      { "name": "DistinguishedName", "type": "string" },
      { "name": "Enabled", "type": "boolean" },
      { "name": "LockedOut", "type": "boolean" },
      { "name": "PasswordLastSet", "type": "datetime" },
      { "name": "PasswordExpires", "type": "datetime" },
      { "name": "DaysUntilExpiry", "type": "long" },
      { "name": "PasswordNeverExpires", "type": "boolean" },
      { "name": "ManagedPassword", "type": "boolean" },
      { "name": "ResolutionStatus", "type": "string" },
      { "name": "CollectionStatus", "type": "string" },
      { "name": "CollectorVersion", "type": "string" }
    ]
  }
}
```

---

# 5. Windows LAPS Metadata Payload and Schema

This stream must contain **metadata only**. Do not include a LAPS password field.

## 5.1 JSON Payload Example

```json name=laps-device-status-payload.json
[
  {
    "TimeGenerated": "2026-09-21T15:00:00Z",
    "DeviceId": "11111111-2222-3333-4444-555555555555",
    "DeviceName": "ERP-APP-01",
    "Computer": "ERP-APP-01",
    "PasswordExpirationTime": "2026-10-20T12:00:00Z",
    "BackupTimestamp": "2026-09-20T12:00:00Z",
    "DaysUntilExpiry": 29,
    "BackupPresent": true,
    "CollectionStatus": "Success",
    "CollectorVersion": "1.0.0"
  }
]
```

## 5.2 DCR Input Stream Schema

```json name=laps-device-status-stream-declaration.json
{
  "Custom-LapsDeviceStatusRaw": {
    "columns": [
      { "name": "TimeGenerated", "type": "datetime" },
      { "name": "DeviceId", "type": "string" },
      { "name": "DeviceName", "type": "string" },
      { "name": "Computer", "type": "string" },
      { "name": "PasswordExpirationTime", "type": "datetime" },
      { "name": "BackupTimestamp", "type": "datetime" },
      { "name": "DaysUntilExpiry", "type": "long" },
      { "name": "BackupPresent", "type": "boolean" },
      { "name": "CollectionStatus", "type": "string" },
      { "name": "CollectorVersion", "type": "string" }
    ]
  }
}
```

---

# 6. ERP Folder-Metrics Payload and Schema

## 6.1 JSON Payload Example

```json name=erp-folder-metrics-payload.json
[
  {
    "TimeGenerated": "2026-09-21T15:00:00Z",
    "Computer": "ERP-APP-01",
    "Environment": "Pilot",
    "Application": "ERP",
    "FolderPath": "D:\\ERP\\Logs",
    "TotalBytes": 5368709120,
    "TotalGB": 5.0,
    "FileCount": 12500,
    "OldestFileUtc": "2025-08-18T03:22:00Z",
    "NewestFileUtc": "2026-09-21T14:58:32Z",
    "FilesOlderThanRetention": 4200,
    "RetentionDays": 30,
    "AccessErrorCount": 0,
    "ScanDurationSeconds": 8.54,
    "CollectionStatus": "Success",
    "CollectorVersion": "1.0.0"
  }
]
```

## 6.2 DCR Input Stream Schema

```json name=erp-folder-metrics-stream-declaration.json
{
  "Custom-ErpFolderMetricsRaw": {
    "columns": [
      { "name": "TimeGenerated", "type": "datetime" },
      { "name": "Computer", "type": "string" },
      { "name": "Environment", "type": "string" },
      { "name": "Application", "type": "string" },
      { "name": "FolderPath", "type": "string" },
      { "name": "TotalBytes", "type": "long" },
      { "name": "TotalGB", "type": "real" },
      { "name": "FileCount", "type": "long" },
      { "name": "OldestFileUtc", "type": "datetime" },
      { "name": "NewestFileUtc", "type": "datetime" },
      { "name": "FilesOlderThanRetention", "type": "long" },
      { "name": "RetentionDays", "type": "long" },
      { "name": "AccessErrorCount", "type": "long" },
      { "name": "ScanDurationSeconds", "type": "real" },
      { "name": "CollectionStatus", "type": "string" },
      { "name": "CollectorVersion", "type": "string" }
    ]
  }
}
```

---

# 7. Collector-Health Payload and Schema

Collector health is required to distinguish:

- “No problem found,” from
- “The collector did not run,” or
- “The collector failed before it could submit data.”

## 7.1 JSON Payload Example

```json name=collector-health-payload.json
[
  {
    "TimeGenerated": "2026-09-21T15:00:12Z",
    "CollectorName": "CertificateInventory",
    "Computer": "ERP-MW-01",
    "TargetScope": "Cert:\\LocalMachine\\My;Cert:\\LocalMachine\\WebHosting",
    "ExecutionId": "6688746a-1b86-4f56-8b41-cf54b5d17782",
    "RecordsCollected": 12,
    "RecordsSubmitted": 12,
    "DurationSeconds": 3.21,
    "Status": "Success",
    "ErrorCode": null,
    "ErrorMessage": null,
    "CollectorVersion": "1.0.0"
  }
]
```

## 7.2 DCR Input Stream Schema

```json name=collector-health-stream-declaration.json
{
  "Custom-CollectorHealthRaw": {
    "columns": [
      { "name": "TimeGenerated", "type": "datetime" },
      { "name": "CollectorName", "type": "string" },
      { "name": "Computer", "type": "string" },
      { "name": "TargetScope", "type": "string" },
      { "name": "ExecutionId", "type": "string" },
      { "name": "RecordsCollected", "type": "long" },
      { "name": "RecordsSubmitted", "type": "long" },
      { "name": "DurationSeconds", "type": "real" },
      { "name": "Status", "type": "string" },
      { "name": "ErrorCode", "type": "string" },
      { "name": "ErrorMessage", "type": "string" },
      { "name": "CollectorVersion", "type": "string" }
    ]
  }
}
```

Do not send a raw exception containing passwords, tokens, connection strings, customer data, or SQL query values. Sanitize error messages before ingesting them.

---

# 8. How Custom DCR Configuration Works

## 8.1 DCR Purpose

A custom **Data Collection Rule** is the controlled contract between your custom collector and the Log Analytics Workspace.

Its purposes are to:

1. Define the **expected incoming JSON schema**.
2. Define the custom **input stream name**.
3. Define the **Log Analytics Workspace destination**.
4. Map the input stream to the destination custom table.
5. Optionally apply an ingestion-time **KQL transformation**.
6. Reject or expose schema problems early.
7. Separate ingestion configuration from PowerShell logic.
8. Provide an Azure resource on which ingestion permissions can be assigned.
9. Support repeatable deployment through ARM, Bicep, Terraform, Azure CLI, or REST API.

A DCR does **not** execute PowerShell. It does not scan folders, query Active Directory, access certificate stores, or run SQL. It receives already collected JSON records.

---

## 8.2 Core DCR Components

A custom Logs Ingestion API DCR includes these logical elements:

```text
DCR
├── streamDeclarations
│    └── Defines expected JSON fields and types
│
├── destinations
│    └── Identifies the Log Analytics Workspace
│
├── dataFlows
│    ├── Accepts the input stream
│    ├── Applies optional KQL transformation
│    └── Routes output to a custom table stream
│
└── dataCollectionEndpointId (optional/architecture-dependent)
     └── Associates DCR with a Data Collection Endpoint where required
```

---

## 8.3 Data Flow Example

For ERP folder metrics:

```text
Custom PowerShell collector
    │
    │ POST JSON array
    ▼
Custom-ErpFolderMetricsRaw
    │
    │ DCR validates fields and types
    ▼
transformKql = source
    │
    │ DCR routes records
    ▼
Custom-ErpFolderMetrics_CL
    │
    ▼
ErpFolderMetrics_CL in Log Analytics Workspace
```

---

# 9. Custom DCR Example: ERP Folder Metrics

The following is a representative DCR fragment. It focuses on the custom stream declaration, workspace destination, and data flow.

```json name=erp-folder-metrics-dcr.json
{
  "location": "canadacentral",
  "properties": {
    "description": "Ingest ERP, middleware, and IIS folder-size telemetry.",
    "kind": "Direct",
    "streamDeclarations": {
      "Custom-ErpFolderMetricsRaw": {
        "columns": [
          {
            "name": "TimeGenerated",
            "type": "datetime"
          },
          {
            "name": "Computer",
            "type": "string"
          },
          {
            "name": "Environment",
            "type": "string"
          },
          {
            "name": "Application",
            "type": "string"
          },
          {
            "name": "FolderPath",
            "type": "string"
          },
          {
            "name": "TotalBytes",
            "type": "long"
          },
          {
            "name": "TotalGB",
            "type": "real"
          },
          {
            "name": "FileCount",
            "type": "long"
          },
          {
            "name": "OldestFileUtc",
            "type": "datetime"
          },
          {
            "name": "NewestFileUtc",
            "type": "datetime"
          },
          {
            "name": "FilesOlderThanRetention",
            "type": "long"
          },
          {
            "name": "RetentionDays",
            "type": "long"
          },
          {
            "name": "AccessErrorCount",
            "type": "long"
          },
          {
            "name": "ScanDurationSeconds",
            "type": "real"
          },
          {
            "name": "CollectionStatus",
            "type": "string"
          },
          {
            "name": "CollectorVersion",
            "type": "string"
          }
        ]
      }
    },
    "destinations": {
      "logAnalytics": [
        {
          "name": "pilotLogAnalytics",
          "workspaceResourceId": "/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.OperationalInsights/workspaces/<workspace-name>"
        }
      ]
    },
    "dataFlows": [
      {
        "streams": [
          "Custom-ErpFolderMetricsRaw"
        ],
        "destinations": [
          "pilotLogAnalytics"
        ],
        "transformKql": "source",
        "outputStream": "Custom-ErpFolderMetrics_CL"
      }
    ]
  }
}
```

## Interpretation

| DCR element | Meaning |
|---|---|
| `kind: "Direct"` | Indicates the DCR is intended for direct ingestion through the Logs Ingestion API. |
| `streamDeclarations` | Defines the custom incoming JSON contract. |
| `Custom-ErpFolderMetricsRaw` | The stream name used in the API request. |
| `destinations.logAnalytics` | Identifies the destination Log Analytics Workspace. |
| `workspaceResourceId` | Azure Resource Manager ID of the workspace. |
| `dataFlows.streams` | Identifies which custom input stream enters this flow. |
| `transformKql: "source"` | Passes input fields through unchanged. |
| `outputStream` | Maps the data flow to the custom destination table stream. |
| `Custom-ErpFolderMetrics_CL` | Corresponds to the `ErpFolderMetrics_CL` workspace table. |

---

# 10. Why Use DCR Transformations

A transformation is an ingestion-time KQL expression in the DCR.

Use transformations when you need to:

- Rename an input field.
- Convert a string to a typed value.
- Normalize server names.
- Add fixed classification values.
- Calculate simple derived fields.
- Drop unneeded fields.
- Remove sensitive fields **before storage**.

## Example: Normalize Hostname and Drop Unwanted Field

Assume the collector emits:

```json
{
  "Host": "erp-app-01.contoso.com",
  "SecretDiagnosticValue": "DoNotStore",
  "TotalBytes": 5368709120
}
```

The DCR transformation could be:

```kusto
source
| extend Computer = toupper(split(Host, ".")[0])
| project-away Host, SecretDiagnosticValue
```

The final table receives `Computer` and `TotalBytes`, but not the sensitive diagnostic value.

## Example: Convert a String Field to Integer

```kusto
source
| extend DaysUntilExpiry = tolong(DaysUntilExpiryText)
| project-away DaysUntilExpiryText
```

For the pilot, avoid unnecessary transformations. Emit correctly typed JSON in PowerShell and use:

```kusto
source
```

This is simpler, easier to test, and minimizes ingestion failures.

---

# 11. API Invocation Parameters

The PowerShell collector needs four principal values:

| Parameter | Description |
|---|---|
| DCE or DCR ingestion endpoint | The endpoint URL to which the collector posts data |
| DCR immutable ID | Stable DCR ingestion identifier, not merely the Azure Resource Manager resource ID |
| Stream name | For example, `Custom-ErpFolderMetricsRaw` |
| JSON payload | Array of records matching that stream declaration |

Conceptual HTTP request:

```text
POST https://<ingestion-endpoint>/dataCollectionRules/<DCR-immutable-ID>/streams/Custom-ErpFolderMetricsRaw?api-version=<supported-api-version>
Authorization: Bearer <Entra-ID-token>
Content-Type: application/json

[
  {
    "TimeGenerated": "2026-09-21T15:00:00Z",
    "Computer": "ERP-APP-01",
    "Application": "ERP",
    "FolderPath": "D:\\ERP\\Logs",
    "TotalBytes": 5368709120,
    "TotalGB": 5.0,
    "FileCount": 12500,
    "CollectionStatus": "Success",
    "CollectorVersion": "1.0.0"
  }
]
```

The exact endpoint is obtained from Azure after creating the DCR and, where used, DCE. Do not construct an endpoint manually based only on region naming conventions.

---

# 12. Custom DCR Configuration Purposes by Pilot Requirement

| Requirement | PowerShell collector responsibility | Custom DCR responsibility | Custom table |
|---|---|---|---|
| Certificate inventory | Read certificate store and IIS bindings | Validate certificate metadata schema; route to workspace; optionally remove fields | `ServerCertificateInventory_CL` |
| Service-account lifecycle | Read service/task/app-pool identity; resolve directory metadata | Validate account-status fields; route records | `ServiceAccountStatus_CL` |
| LAPS metadata | Query permitted LAPS metadata without password retrieval | Validate device/expiry/backup fields; route records | `LapsDeviceStatus_CL` |
| ERP folder size | Measure approved directory trees | Validate capacity metrics; route records | `ErpFolderMetrics_CL` |
| Collector health | Record success/failure, duration, counts | Validate health fields; route records | `CollectorHealth_CL` |

---

# 13. DCR Does Not Replace the Collector

The division of responsibility is important.

| Function | PowerShell collector | Custom DCR |
|---|---:|---:|
| Read Windows certificate store | Yes | No |
| Query IIS binding configuration | Yes | No |
| Query Active Directory | Yes | No |
| Query Entra/LAPS metadata | Yes | No |
| Measure ERP log directory | Yes | No |
| Authenticate to Logs Ingestion API | Yes | No |
| Send JSON records | Yes | No |
| Define accepted stream schema | No | Yes |
| Route data to Log Analytics | No | Yes |
| Apply ingestion transformation | No | Yes |
| Store/query data | No | No — Log Analytics Workspace does this |
| Create alert/workbook | No | No — Azure Monitor uses workspace data |

---

# 14. Recommended DCR Deployment Model

Use separate DCRs based on security boundary and operational purpose:

| DCR | Streams |
|---|---|
| `dcr-pilot-certificate-inventory` | `Custom-ServerCertificateInventoryRaw` |
| `dcr-pilot-service-account-status` | `Custom-ServiceAccountStatusRaw` |
| `dcr-pilot-laps-status` | `Custom-LapsDeviceStatusRaw` |
| `dcr-pilot-folder-metrics` | `Custom-ErpFolderMetricsRaw` |
| `dcr-pilot-collector-health` | `Custom-CollectorHealthRaw` |

This is preferable to placing all streams into one DCR because it enables:

- Separate least-privilege role assignments.
- Cleaner change control.
- Independent schema evolution.
- Easier troubleshooting.
- Distinct data owners.
- Safer rollback.
- Clearer ingestion-cost analysis.

For example, the certificate-inventory collector should have permission only to ingest into the certificate DCR. It should not be able to submit SQL, account, or network telemetry.

---

# 15. Schema Versioning and Change Management

A DCR and table schema should be treated as an interface contract.

When adding fields:

1. Add the new table column.
2. Update the DCR stream declaration.
3. Update the DCR transformation if applicable.
4. Release the updated PowerShell collector.
5. Update KQL queries, workbook visualizations, and alerts.
6. Test with non-production data.
7. Increment `CollectorVersion`.

Avoid removing or renaming fields during the pilot. Add a replacement field first, migrate KQL/alerts, and retire the old field only after validation.

A suitable version record is:

```json
{
  "CollectorVersion": "1.1.0"
}
```

This helps support personnel determine whether unexpected results came from an old collector deployment or a schema change.

---

# 16. Validation Checklist

Before enabling scheduled execution, validate each stream end to end:

1. Custom table exists in the designated Log Analytics Workspace.
2. Table columns have the correct types.
3. DCR `streamDeclarations` match the collector JSON field names and types.
4. DCR `dataFlows` map to the correct workspace destination.
5. Output stream maps to the intended `_CL` table.
6. Collector identity has the minimum role needed to ingest through that DCR.
7. A manually posted single-record JSON payload succeeds.
8. The record appears in the expected table.
9. `TimeGenerated` is correctly interpreted as UTC.
10. Null values are accepted where expected.
11. No secrets or passwords are present in table records.
12. An intentional invalid payload is rejected and logged.
13. Collector-health data is emitted even if the primary collection fails.
14. KQL queries, Workbooks, and alert rules return expected results.

The overall design principle is:

> The PowerShell collector gathers and submits structured facts; the custom DCR validates, optionally transforms, and routes those facts; and the Log Analytics custom table is the durable, searchable destination for dashboards, alerting, correlation, and ServiceNow workflow.
