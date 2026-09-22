# A.5 — Custom Log Analytics Tables: Implementation and Schema

## 1. Where Custom Tables Reside

The custom tables reside in an **Azure Log Analytics Workspace**, not on the Windows servers and not inside the Azure Data Collection Endpoint.

```text
On-premises collector
        │
        │ JSON records over HTTPS
        ▼
Data Collection Endpoint, if used
        │
        ▼
Data Collection Rule
        │
        │ stream mapping and transformation
        ▼
Log Analytics Workspace
        │
        └── Custom tables
              ├── ServerCertificateInventory_CL
              ├── ServiceAccountStatus_CL
              ├── LapsDeviceStatus_CL
              └── ErpFolderMetrics_CL
```

The custom table is physically part of the selected Log Analytics Workspace’s Azure Monitor Logs data store.

The `_CL` suffix means the table is a **custom log table**. For example:

```text
ServerCertificateInventory_CL
```

The table is logically scoped to the workspace:

```text
Subscription
└── Resource group
    └── Log Analytics Workspace
        └── ServerCertificateInventory_CL
```

The table is not a traditional relational SQL table and does not have primary keys, foreign keys, indexes managed by the integrator, or a manually administered database server behind it. Azure Monitor manages the storage, indexing, retention, and query engine.

---

# 2. The Three Schemas You Must Design

For Logs Ingestion API collection, distinguish among three related schemas.

## 2.1 Collector payload schema

This is the JSON produced by PowerShell.

Example:

```json
{
  "TimeGenerated": "2026-09-21T15:30:00Z",
  "Computer": "ERP-APP-01",
  "FolderPath": "D:\\ERP\\Logs",
  "TotalBytes": 5368709120,
  "TotalGB": 5.0,
  "FileCount": 12500,
  "CollectionStatus": "Success"
}
```

## 2.2 DCR input stream schema

This is the schema expected by the DCR stream.

The stream name is normally custom, for example:

```text
Custom-ErpFolderMetrics
```

The DCR declares the fields and their data types.

Conceptual stream definition:

```text
Custom-ErpFolderMetrics
├── TimeGenerated: datetime
├── Computer: string
├── FolderPath: string
├── TotalBytes: long
├── TotalGB: real
├── FileCount: long
└── CollectionStatus: string
```

The JSON property names and types submitted by the collector must match the DCR input schema, unless the DCR transformation explicitly maps or converts them.

## 2.3 Destination table schema

This is the schema stored in Log Analytics.

For a direct mapping, the destination table can have the same fields as the stream:

```text
ErpFolderMetrics_CL
├── TimeGenerated: datetime
├── Computer: string
├── FolderPath: string
├── TotalBytes: long
├── TotalGB: real
├── FileCount: long
└── CollectionStatus: string
```

A DCR can also transform the input into a different output schema. For example:

```text
Input:
  Computer

Output:
  ComputerName
```

However, for this pilot, use **matching input and destination field names wherever possible**. It simplifies development, testing, KQL, and support.

---

# 3. Recommended Table Design

The following tables are recommended for the pilot:

| Custom table | Purpose |
|---|---|
| `ServerCertificateInventory_CL` | Certificate-store and IIS certificate-binding inventory |
| `ServiceAccountStatus_CL` | Windows service, IIS app-pool, scheduled-task, and directory-account status |
| `LapsDeviceStatus_CL` | Windows LAPS metadata from Microsoft Entra ID without password values |
| `ErpFolderMetrics_CL` | ERP, IIS, middleware, and SQL-related directory size metrics |
| `CollectorHealth_CL` | Health of the custom collector jobs themselves |

---

# 4. Table 1 — `ServerCertificateInventory_CL`

## 4.1 Recommended schema

| Column | Azure Monitor type | Description |
|---|---|---|
| `TimeGenerated` | `datetime` | UTC collection timestamp; required operational field |
| `Computer` | `string` | Windows server name |
| `Environment` | `string` | Development, test, production, etc. |
| `Application` | `string` | ERP, middleware, IIS, SQL, or other application |
| `StoreLocation` | `string` | Usually `LocalMachine` |
| `StoreName` | `string` | `My`, `WebHosting`, etc. |
| `Subject` | `string` | Certificate subject |
| `DnsNames` | `string` | Subject Alternative Names, preferably semicolon-delimited |
| `Issuer` | `string` | Certificate issuer |
| `Thumbprint` | `string` | Certificate thumbprint |
| `SerialNumber` | `string` | Certificate serial number |
| `NotBefore` | `datetime` | Certificate validity start |
| `NotAfter` | `datetime` | Certificate expiry |
| `DaysUntilExpiry` | `long` | Calculated days remaining |
| `HasPrivateKey` | `boolean` | Indicates whether the certificate has a private key |
| `EnhancedKeyUsage` | `string` | Certificate usages |
| `IisBindings` | `string` | Associated IIS bindings |
| `CollectionStatus` | `string` | `Success`, `PartialSuccess`, or error state |
| `CollectorVersion` | `string` | Collector version |

## 4.2 Example table row

```json
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
```

## 4.3 Example KQL

```kusto
ServerCertificateInventory_CL
| where TimeGenerated > ago(2d)
| summarize arg_max(TimeGenerated, *) by Computer, Thumbprint
| where CollectionStatus == "Success"
| project
    Computer,
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

# 5. Table 2 — `ServiceAccountStatus_CL`

This table contains **identity metadata**, not passwords.

## 5.1 Recommended schema

| Column | Azure Monitor type | Description |
|---|---|---|
| `TimeGenerated` | `datetime` | UTC collection timestamp |
| `Computer` | `string` | Server where the identity is configured |
| `UsageType` | `string` | `WindowsService`, `ScheduledTask`, `IISApplicationPool`, etc. |
| `UsageName` | `string` | Service name, task name, or app-pool name |
| `DisplayName` | `string` | Friendly display name |
| `ConfiguredAccount` | `string` | Account configured for the workload |
| `AccountType` | `string` | `DomainUser`, `gMSA`, `BuiltInOrVirtual`, `UnknownOrLocal` |
| `SamAccountName` | `string` | Normalized account name |
| `DistinguishedName` | `string` | AD distinguished name, if appropriate |
| `Enabled` | `boolean` | Account enabled state |
| `LockedOut` | `boolean` | Account locked state |
| `PasswordLastSet` | `datetime` | Last password-set time, where applicable |
| `PasswordExpires` | `datetime` | Calculated expiry time, where applicable |
| `DaysUntilExpiry` | `long` | Days until calculated expiry |
| `PasswordNeverExpires` | `boolean` | Directory setting |
| `ManagedPassword` | `boolean` | Whether password is managed, such as a gMSA |
| `ResolutionStatus` | `string` | `Resolved`, `NotResolved`, or `NotApplicable` |
| `CollectionStatus` | `string` | Collector result |
| `CollectorVersion` | `string` | Collector version |

## 5.2 Data that must not be stored

Do not include:

- Password values.
- LAPS password values.
- Access tokens.
- Client secrets.
- Private keys.
- Password hashes.
- Security answers.

## 5.3 Example KQL

```kusto
ServiceAccountStatus_CL
| where TimeGenerated > ago(2d)
| summarize arg_max(TimeGenerated, *) by Computer, UsageType, UsageName
| where
    Enabled == false
    or LockedOut == true
    or (
        AccountType == "DomainUser"
        and PasswordNeverExpires == false
        and DaysUntilExpiry <= 14
    )
| project
    Computer,
    UsageType,
    UsageName,
    ConfiguredAccount,
    AccountType,
    Enabled,
    LockedOut,
    PasswordExpires,
    DaysUntilExpiry,
    ResolutionStatus
```

---

# 6. Table 3 — `LapsDeviceStatus_CL`

This table is for **Windows LAPS metadata only**.

## 6.1 Recommended schema

| Column | Azure Monitor type | Description |
|---|---|---|
| `TimeGenerated` | `datetime` | Collection timestamp |
| `DeviceId` | `string` | Entra device identifier |
| `DeviceName` | `string` | Device name |
| `Computer` | `string` | Optional normalized hostname |
| `PasswordExpirationTime` | `datetime` | LAPS-managed password expiry metadata |
| `BackupTimestamp` | `datetime` | Last successful backup timestamp |
| `DaysUntilExpiry` | `long` | Calculated days until rotation |
| `BackupPresent` | `boolean` | Whether backup metadata exists |
| `CollectionStatus` | `string` | Collection result |
| `CollectorVersion` | `string` | Collector version |

## 6.2 Important distinction

The field name `PasswordExpirationTime` describes the **metadata about the LAPS-managed password**. It is not the password itself.

The collector must use metadata-only permissions and must not request the sensitive password property.

## 6.3 Example KQL

```kusto
LapsDeviceStatus_CL
| where TimeGenerated > ago(2d)
| summarize arg_max(TimeGenerated, *) by DeviceId
| where
    BackupPresent == false
    or DaysUntilExpiry <= 3
    or CollectionStatus != "Success"
| project
    DeviceId,
    DeviceName,
    PasswordExpirationTime,
    BackupTimestamp,
    DaysUntilExpiry,
    BackupPresent,
    CollectionStatus
```

---

# 7. Table 4 — `ErpFolderMetrics_CL`

## 7.1 Recommended schema

| Column | Azure Monitor type | Description |
|---|---|---|
| `TimeGenerated` | `datetime` | Scan start time |
| `Computer` | `string` | Windows server |
| `Environment` | `string` | Pilot, test, production |
| `Application` | `string` | ERP, IIS, middleware, SQL backup, etc. |
| `FolderPath` | `string` | Monitored folder |
| `TotalBytes` | `long` | Total file bytes |
| `TotalGB` | `real` | Total size in GB |
| `FileCount` | `long` | Number of files |
| `OldestFileUtc` | `datetime` | Oldest file timestamp |
| `NewestFileUtc` | `datetime` | Newest file timestamp |
| `FilesOlderThanRetention` | `long` | Count beyond retention |
| `RetentionDays` | `long` | Configured retention threshold |
| `AccessErrorCount` | `long` | Files/directories skipped due to access errors |
| `ScanDurationSeconds` | `real` | Collection duration |
| `CollectionStatus` | `string` | `Success`, `PartialSuccess`, or error |
| `CollectorVersion` | `string` | Collector version |

## 7.2 Example KQL

```kusto
ErpFolderMetrics_CL
| where TimeGenerated > ago(2d)
| summarize arg_max(TimeGenerated, *) by Computer, Application, FolderPath
| project
    Computer,
    Application,
    FolderPath,
    TotalGB,
    FileCount,
    OldestFileUtc,
    FilesOlderThanRetention,
    AccessErrorCount,
    ScanDurationSeconds,
    CollectionStatus
```

---

# 8. Table 5 — `CollectorHealth_CL`

This table is important because a missing custom table record may mean either:

- The source is healthy and produced no issue, or
- The collector itself failed.

## 8.1 Recommended schema

| Column | Azure Monitor type | Description |
|---|---|---|
| `TimeGenerated` | `datetime` | Collector execution time |
| `CollectorName` | `string` | Certificate, service account, LAPS, or folder collector |
| `Computer` | `string` | Execution host |
| `TargetScope` | `string` | Server, folder, domain, or device scope |
| `ExecutionId` | `string` | Unique run ID |
| `RecordsCollected` | `long` | Number of records collected |
| `RecordsSubmitted` | `long` | Number submitted to Azure |
| `DurationSeconds` | `real` | Execution duration |
| `Status` | `string` | `Success`, `PartialSuccess`, or `Failed` |
| `ErrorCode` | `string` | Normalized error code |
| `ErrorMessage` | `string` | Sanitized error message |
| `CollectorVersion` | `string` | Version |

## 8.2 Collector health alert

```kusto
CollectorHealth_CL
| where TimeGenerated > ago(2h)
| summarize arg_max(TimeGenerated, *) by CollectorName, Computer
| where Status != "Success"
| project
    TimeGenerated,
    CollectorName,
    Computer,
    TargetScope,
    Status,
    ErrorCode,
    ErrorMessage,
    DurationSeconds
```

Also create a **missing-data alert**:

```kusto
let expectedCollectors = datatable(CollectorName:string)
[
    "CertificateInventory",
    "ServiceAccountStatus",
    "LapsDeviceStatus",
    "ErpFolderMetrics"
];
expectedCollectors
| join kind=leftouter (
    CollectorHealth_CL
    | where TimeGenerated > ago(26h)
    | summarize LastRun=max(TimeGenerated) by CollectorName
) on CollectorName
| where isempty(LastRun) or LastRun < ago(25h)
```

---

# 9. Creating the Custom Tables

There are three common implementation approaches.

## 9.1 Azure portal

Use this for the pilot if the team wants a controlled manual setup.

Typical sequence:

1. Open the target Log Analytics Workspace.
2. Open the **Tables** blade.
3. Select the option to create a custom table.
4. Choose the custom-log or DCR-based table option.
5. Define the table name.
6. Define columns and data types.
7. Create or associate the DCR.
8. Define the custom input stream.
9. Map the stream to the table.
10. Test with a sample payload.
11. Query the resulting table in Logs.

Portal terminology can change, so the implementation team should confirm the current table-creation experience for the selected Azure region and API version.

## 9.2 Azure CLI or ARM/Bicep

Use infrastructure as code for repeatable deployment.

The deployment should define:

- Log Analytics Workspace.
- Custom table schema.
- DCR.
- DCE, if required.
- DCR association.
- Managed identity or service principal role assignment.
- Alert rules.
- Workbook resources.

This is recommended for promotion from pilot to test and production.

## 9.3 Azure REST API or PowerShell

Use this for automated provisioning pipelines or when the portal does not expose the exact schema requirement.

The provisioning process should be version-controlled and should make table and DCR changes auditable.

---

# 10. Conceptual DCR Structure

A DCR for `ErpFolderMetrics_CL` contains four important parts:

```json
{
  "dataSources": {
    "dataImports": [
      {
        "name": "erpFolderMetricsInput",
        "streams": [
          "Custom-ErpFolderMetrics"
        ]
      }
    ]
  },
  "streamDeclarations": {
    "Custom-ErpFolderMetrics": {
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
          "name": "CollectionStatus",
          "type": "string"
        }
      ]
    }
  },
  "dataFlows": [
    {
      "streams": [
        "Custom-ErpFolderMetrics"
      ],
      "destinations": [
        "logAnalyticsDestination"
      ],
      "transformKql": "source",
      "outputStream": "Custom-ErpFolderMetrics"
    }
  ],
  "destinations": {
    "logAnalytics": [
      {
        "name": "logAnalyticsDestination",
        "workspaceResourceId": "<workspace-resource-id>"
      }
    ]
  }
}
```

This is a conceptual structure. The exact ARM/Bicep/API property names and supported `dataSources` definition depend on the selected Azure Monitor Logs Ingestion API version. The implementation team should validate the final template against the current Azure API specification before deployment.

---

# 11. DCR Input and Table Mapping

The data path for one record is:

```text
PowerShell property       DCR stream field       Workspace table column
------------------        ----------------       -----------------------
Computer                  Computer               Computer
FolderPath                FolderPath             FolderPath
TotalBytes                TotalBytes             TotalBytes
TotalGB                   TotalGB                TotalGB
TimeGenerated             TimeGenerated          TimeGenerated
```

For a direct mapping:

```kusto
transformKql = "source"
```

For a transformation:

```kusto
source
| extend DaysUntilExpiry = tolong((NotAfter - now()) / 1d)
| project-away SensitiveField
```

Use transformations to remove data that should not enter the workspace. Do not rely solely on a workbook or query to hide sensitive data after ingestion.

---

# 12. Schema Design Rules

## Use typed columns

Prefer:

```text
DaysUntilExpiry: long
TotalBytes: long
TotalGB: real
HasPrivateKey: boolean
NotAfter: datetime
```

Avoid storing all values as strings. String-only schemas make alerting and aggregation less reliable.

## Use UTC

All timestamps should be emitted in UTC and use ISO 8601 format:

```text
2026-09-21T15:30:00Z
```

## Keep stable names

Changing a field from:

```text
DaysUntilExpiry
```

to:

```text
CertificateDaysRemaining
```

requires DCR, collector, workbook, and alert changes. Establish naming standards first.

## Avoid high-cardinality fields

Do not include unnecessary:

- Full filenames.
- Complete SQL statements.
- Process command lines containing secrets.
- File content.
- Long exception text.
- Dynamic nested objects for every record.

## Include collection status

A record should indicate whether the collection was:

- `Success`
- `PartialSuccess`
- `FolderNotFound`
- `AccessDenied`
- `NotResolved`
- `Error`

This makes operational diagnosis easier.

---

# 13. Table Retention and Cost

Custom tables inherit the workspace’s retention model unless configured otherwise. For this pilot:

- Use 30–90 days of interactive retention initially.
- Archive longer-term data only when there is a compliance or trend-analysis requirement.
- Avoid storing raw high-volume data if an aggregated record is sufficient.
- Use 15–60-minute collection intervals for folder metrics.
- Use daily collection for certificates and account inventory.
- Use a separate table for high-volume SQL or connection-state telemetry.

The data-ingestion volume is driven by:

```text
record size × records per execution × execution frequency × number of servers
```

The pilot should measure actual ingestion volume before setting production retention.

---

# 14. Recommended Implementation Sequence

1. Create the Log Analytics Workspace.
2. Define table schemas in a data dictionary.
3. Create custom tables.
4. Create DCR stream declarations.
5. Create data flows and workspace destinations.
6. Create the DCE if the selected ingestion architecture requires it.
7. Assign the collector identity its minimum ingestion permission.
8. Deploy the shared PowerShell ingestion module.
9. Test with one manually submitted record.
10. Confirm the record appears in the expected custom table.
11. Deploy each collector independently.
12. Create KQL validation queries.
13. Create workbooks.
14. Create alert rules only after data quality is validated.
15. Test failure, recovery, deduplication, and missing-data conditions.
16. Convert the deployment to ARM/Bicep or an equivalent infrastructure-as-code process.

The most important design principle is:

> The custom table is the final storage and query destination in Log Analytics; the DCR defines how the incoming stream is shaped and routed; and the PowerShell script is only responsible for collecting and submitting correctly structured telemetry.
