# Phase 1 — PowerShell Collector Project and Windows Deployment Foundation

## 1. Objective

Build a reusable, secure PowerShell collector framework that can support:

- **Certificate inventory/expiry monitoring** from `Cert:\LocalMachine\WebHosting`.
- **ERP log-directory size and growth monitoring**.
- Future collectors, such as SQL DMV, REST API, service-account, and network-connection collectors.

Phase 1 creates the local project, deployment package, configuration model, logging, task scheduling, execution locking, and collector-health pattern. It does **not** create Azure custom tables/DCRs yet; that is Phase 2.

---

# 2. Source Repository Structure

Use a single source-controlled repository with Bicep, PowerShell, KQL, configuration templates, tests, and operational documentation.

```text
erp-hybrid-monitoring/
├── README.md
├── CHANGELOG.md
├── .gitignore
├── azure/
│   ├── main.bicep
│   ├── modules/
│   │   ├── log-analytics-workspace.bicep
│   │   ├── custom-tables.bicep
│   │   ├── data-collection-rules.bicep
│   │   ├── role-assignments.bicep
│   │   ├── action-groups.bicep
│   │   ├── scheduled-query-rules.bicep
│   │   └── workbooks.bicep
│   └── parameters/
│       ├── pilot.bicepparam
│       ├── test.bicepparam
│       └── production.bicepparam
│
├── collectors/
│   ├── modules/
│   │   ├── Collector.Common.psm1
│   │   ├── Collector.Logging.psm1
│   │   ├── Collector.Locking.psm1
│   │   ├── Collector.ArcIdentity.psm1
│   │   └── Collector.Ingestion.psm1
│   │
│   ├── certificate/
│   │   ├── Collect-CertificateInventory.ps1
│   │   └── CertificateInventory.config.json
│   │
│   ├── folder-metrics/
│   │   ├── Collect-ErpFolderMetrics.ps1
│   │   └── ErpFolderMetrics.config.json
│   │
│   ├── deployment/
│   │   ├── Install-CollectorPackage.ps1
│   │   ├── Register-CollectorScheduledTasks.ps1
│   │   ├── Uninstall-CollectorPackage.ps1
│   │   └── Test-CollectorPrerequisites.ps1
│   │
│   └── config/
│       └── CollectorSettings.template.json
│
├── kql/
│   ├── certificate/
│   ├── folder-metrics/
│   ├── collector-health/
│   └── alerts/
│
├── tests/
│   ├── unit/
│   ├── integration/
│   └── test-data/
│
└── docs/
    ├── architecture/
    ├── operations/
    ├── runbooks/
    └── security/
```

## Why this structure

| Folder | Purpose |
|---|---|
| `azure/` | Bicep infrastructure-as-code. |
| `collectors/modules/` | Shared code; no duplicate token, retry, logging, or lock logic. |
| `collectors/certificate/` | Certificate-specific business logic only. |
| `collectors/folder-metrics/` | ERP-folder-specific business logic only. |
| `collectors/deployment/` | Repeatable local installation, scheduled-task registration, testing, and removal. |
| `kql/` | Version-controlled Workbook and alert-query source. |
| `tests/` | Pester unit tests and controlled integration tests. |
| `docs/` | Operational ownership, access, deployment, and troubleshooting documentation. |

---

# 3. Windows Server Deployment Structure

Deploy only the required collector components to each server. Do not deploy the whole repository.

```text
C:\ProgramData\Contoso\AzureMonitorCollectors\
├── Current\
│   ├── Modules\
│   │   ├── Collector.Common.psm1
│   │   ├── Collector.Logging.psm1
│   │   ├── Collector.Locking.psm1
│   │   ├── Collector.ArcIdentity.psm1
│   │   └── Collector.Ingestion.psm1
│   │
│   ├── Collectors\
│   │   ├── Collect-CertificateInventory.ps1
│   │   └── Collect-ErpFolderMetrics.ps1
│   │
│   ├── Config\
│   │   ├── CollectorSettings.json
│   │   ├── CertificateInventory.config.json
│   │   └── ErpFolderMetrics.config.json
│   │
│   └── Version\
│       └── release.json
│
├── Logs\
│   ├── CertificateInventory\
│   └── ErpFolderMetrics\
│
├── Spool\
│   ├── CertificateInventory\
│   └── ErpFolderMetrics\
│
├── Locks\
└── Previous\
```

## Directory Responsibilities

| Path | Content | Access control |
|---|---|---|
| `Current\` | Active signed collector release. | Administrators and deployment service only modify; collector account read/execute only. |
| `Config\` | Environment and local collector settings. | Administrators/deployment service modify; collector account read only. |
| `Logs\` | Local operational logs. | Collector account modify; support team read; routine retention cleanup. |
| `Spool\` | Optional JSON batches awaiting retry after temporary Azure ingestion failure. | Collector account modify; support team read. Must not contain secrets. |
| `Locks\` | Prevents concurrent copies of the same collector. | Collector account modify. |
| `Previous\` | Previous signed release for rollback. | Deployment service only modify. |

> The `Spool` directory is optional for the first proof of ingestion. It is strongly recommended before production because temporary proxy/Azure connectivity failures should not silently discard critical collector records.

---

# 4. Runtime Requirements

## 4.1 Windows Prerequisites

| Requirement | Pilot baseline |
|---|---|
| Windows Server | Supported version for Azure Arc and PowerShell policy. |
| Azure Arc | Connected Machine agent installed and server onboarded. |
| Azure Arc identity | System-assigned managed identity enabled. |
| PowerShell | Windows PowerShell 5.1 minimum; PowerShell 7 may be adopted if enterprise standard. |
| Execution policy | `AllSigned` recommended for production. |
| Task Scheduler | Running and operational. |
| .NET/TLS | TLS 1.2 or later available and permitted. |
| Local disk | Sufficient working space for logs and short-lived spool files. |
| Network | Outbound TCP 443 to Arc, Entra ID, and Azure Monitor ingestion endpoints. |
| Local group | Scheduled-task account authorized to access Azure Arc HIMDS endpoint. |

## 4.2 Required PowerShell Capabilities

The initial Phase 1 framework should use built-in capabilities where possible:

- `ConvertTo-Json`
- `ConvertFrom-Json`
- `Invoke-RestMethod`
- `Get-ChildItem`
- `Get-Item`
- `Get-WebBinding` from `WebAdministration`, when IIS binding correlation is enabled
- `Get-ScheduledTask`
- `Register-ScheduledTask`

Avoid unnecessary external modules on ERP servers.

---

# 5. Configuration Model

Use JSON configuration files. Do not hard-code DCR IDs, endpoint URLs, folder paths, environment labels, or thresholds inside collector scripts.

## 5.1 Shared Collector Settings

```json name=CollectorSettings.template.json
{
  "SchemaVersion": "1.0",
  "Organization": "Contoso",
  "Environment": "Pilot",
  "Application": "ERP",
  "CollectorRoot": "C:\\ProgramData\\Contoso\\AzureMonitorCollectors",
  "ArcManagedIdentity": {
    "TokenResource": "https://monitor.azure.com/",
    "TokenApiVersion": "2020-06-01"
  },
  "Logging": {
    "MinimumLevel": "Information",
    "RetentionDays": 30,
    "MaximumFileSizeMB": 10
  },
  "Ingestion": {
    "MaximumRecordsPerBatch": 500,
    "MaximumPayloadBytes": 900000,
    "MaximumAttempts": 4,
    "RetryBaseDelaySeconds": 2,
    "EnableLocalSpool": true,
    "SpoolRetentionDays": 3
  }
}
```

### Configuration rules

- Keep environment-specific values outside scripts.
- Store DCR immutable ID and ingestion endpoint in role-specific configuration.
- Do not store authentication secrets; Arc managed identity supplies Azure authentication.
- Use separate configurations for certificate and folder collectors.
- Treat configuration as code: source controlled, reviewed, versioned, and deployed through a controlled process.

## 5.2 Certificate Collector Configuration

```json name=CertificateInventory.config.json
{
  "SchemaVersion": "1.0",
  "CollectorName": "CertificateInventory",
  "Enabled": true,
  "StreamName": "Custom-ServerCertificateInventoryRaw",
  "IngestionEndpoint": "https://<DCR-or-DCE-ingestion-endpoint>",
  "DcrImmutableId": "dcr-<immutable-id>",
  "CertificateStores": [
    "Cert:\\LocalMachine\\WebHosting"
  ],
  "EnableIisBindingDiscovery": true,
  "ExpiryWarningDays": 30,
  "ExpiryCriticalDays": 14
}
```

## 5.3 ERP Folder Metrics Configuration

```json name=ErpFolderMetrics.config.json
{
  "SchemaVersion": "1.0",
  "CollectorName": "ErpFolderMetrics",
  "Enabled": true,
  "StreamName": "Custom-ErpFolderMetricsRaw",
  "IngestionEndpoint": "https://<DCR-or-DCE-ingestion-endpoint>",
  "DcrImmutableId": "dcr-<immutable-id>",
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

The actual ERP path remains a deployment parameter until supplied by the ERP operations team.

---

# 6. Shared PowerShell Module Design

Use modular code. Each collector should import the same shared components.

```text
Collector.Common.psm1
    ├── Read-CollectorConfiguration
    ├── Get-CollectorContext
    └── New-CollectorHealthRecord

Collector.Logging.psm1
    ├── Write-CollectorLog
    └── Remove-ExpiredCollectorLogs

Collector.Locking.psm1
    ├── Enter-CollectorLock
    └── Exit-CollectorLock

Collector.ArcIdentity.psm1
    └── Get-ArcManagedIdentityAccessToken

Collector.Ingestion.psm1
    ├── Send-AzureMonitorRecords
    ├── Save-CollectorSpoolBatch
    └── Submit-CollectorSpoolBatches
```

---

## 6.1 Local Logging Module

Every execution needs a local troubleshooting log independent of Azure Monitor ingestion.

```powershell name=Collector.Logging.psm1
Set-StrictMode -Version Latest

function Write-CollectorLog {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)]
        [string] $LogDirectory,

        [Parameter(Mandatory)]
        [ValidateSet('Debug', 'Information', 'Warning', 'Error')]
        [string] $Level,

        [Parameter(Mandatory)]
        [string] $Message,

        [string] $ExecutionId
    )

    if (-not (Test-Path -LiteralPath $LogDirectory)) {
        New-Item -Path $LogDirectory -ItemType Directory -Force | Out-Null
    }

    $timestamp = (Get-Date).ToUniversalTime().ToString('o')
    $logFile = Join-Path $LogDirectory ("{0:yyyy-MM-dd}.log" -f (Get-Date))

    $record = [ordered]@{
        TimeGenerated = $timestamp
        Level         = $Level
        ExecutionId   = $ExecutionId
        Message       = $Message
    }

    # Keep messages sanitized: no tokens, passwords, private keys, or raw payloads.
    ($record | ConvertTo-Json -Compress) |
        Add-Content -LiteralPath $logFile -Encoding UTF8
}

Export-ModuleMember -Function Write-CollectorLog
```

Local logs should contain operational events only:

- Start/finish.
- Record counts.
- Duration.
- Target folder/store.
- HTTP status class.
- Retry occurrence.
- Sanitized failure reason.

They must not contain bearer tokens, JSON data with sensitive fields, private certificate material, or secrets.

---

## 6.2 Execution Locking Module

Without a lock, a long folder scan can overlap with the next scheduled task, duplicate records, and increase server load.

```powershell name=Collector.Locking.psm1
Set-StrictMode -Version Latest

function Enter-CollectorLock {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)]
        [string] $LockPath
    )

    $lockDirectory = Split-Path -Path $LockPath -Parent

    if (-not (Test-Path -LiteralPath $lockDirectory)) {
        New-Item -Path $lockDirectory -ItemType Directory -Force | Out-Null
    }

    try {
        $stream = [System.IO.File]::Open(
            $LockPath,
            [System.IO.FileMode]::OpenOrCreate,
            [System.IO.FileAccess]::ReadWrite,
            [System.IO.FileShare]::None
        )

        return $stream
    }
    catch {
        throw "Collector lock is already held: $LockPath"
    }
}

function Exit-CollectorLock {
    [CmdletBinding()]
    param(
        [Parameter()]
        [System.IO.FileStream] $LockHandle,

        [Parameter(Mandatory)]
        [string] $LockPath
    )

    if ($LockHandle) {
        $LockHandle.Dispose()
    }

    if (Test-Path -LiteralPath $LockPath) {
        Remove-Item -LiteralPath $LockPath -Force -ErrorAction SilentlyContinue
    }
}

Export-ModuleMember -Function Enter-CollectorLock, Exit-CollectorLock
```

Use both:

1. Task Scheduler setting: **Do not start a new instance**.
2. In-process lock file: defense in depth.

---

## 6.3 Azure Arc Managed Identity Module

This module obtains an Azure Monitor token through the Azure Arc local identity endpoint.

```powershell name=Collector.ArcIdentity.psm1
Set-StrictMode -Version Latest

function Get-ArcManagedIdentityAccessToken {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)]
        [string] $Resource = 'https://monitor.azure.com/',

        [string] $ApiVersion = '2020-06-01'
    )

    if ([string]::IsNullOrWhiteSpace($env:IDENTITY_ENDPOINT)) {
        throw (
            'IDENTITY_ENDPOINT is unavailable. Confirm that this server is Azure Arc-enabled ' +
            'and that its system-assigned managed identity is enabled.'
        )
    }

    $identityEndpoint = [uri]$env:IDENTITY_ENDPOINT

    if (-not $identityEndpoint.IsLoopback) {
        throw 'IDENTITY_ENDPOINT must resolve to a local loopback endpoint.'
    }

    $escapedResource = [uri]::EscapeDataString($Resource)
    $tokenUri = (
        '{0}?resource={1}&api-version={2}' -f
        $identityEndpoint.AbsoluteUri,
        $escapedResource,
        $ApiVersion
    )

    try {
        $response = Invoke-RestMethod `
            -Method Get `
            -Uri $tokenUri `
            -Headers @{ Metadata = 'True' } `
            -TimeoutSec 30 `
            -ErrorAction Stop
    }
    catch {
        throw "Unable to obtain Azure Arc managed identity token. $($_.Exception.Message)"
    }

    if ([string]::IsNullOrWhiteSpace($response.access_token)) {
        throw 'Azure Arc managed identity response did not contain an access token.'
    }

    return [string]$response.access_token
}

Export-ModuleMember -Function Get-ArcManagedIdentityAccessToken
```

The scheduled-task account must be authorized to access the Arc local token endpoint. Test this before collector deployment.

---

## 6.4 Common Context and Health-Record Module

```powershell name=Collector.Common.psm1
Set-StrictMode -Version Latest

function Read-CollectorConfiguration {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)]
        [string] $Path
    )

    if (-not (Test-Path -LiteralPath $Path -PathType Leaf)) {
        throw "Configuration file was not found: $Path"
    }

    try {
        return Get-Content -LiteralPath $Path -Raw -Encoding UTF8 |
            ConvertFrom-Json -ErrorAction Stop
    }
    catch {
        throw "Configuration file is invalid JSON: $Path. $($_.Exception.Message)"
    }
}

function New-CollectorHealthRecord {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)]
        [string] $CollectorName,

        [Parameter(Mandatory)]
        [string] $ExecutionId,

        [Parameter(Mandatory)]
        [datetime] $StartedUtc,

        [Parameter(Mandatory)]
        [ValidateSet('Success', 'PartialSuccess', 'Failed')]
        [string] $Status,

        [long] $RecordsCollected = 0,

        [long] $RecordsSubmitted = 0,

        [string] $TargetScope,

        [string] $ErrorCode,

        [string] $ErrorMessage,

        [string] $CollectorVersion = '1.0.0'
    )

    $completedUtc = (Get-Date).ToUniversalTime()

    return [pscustomobject]@{
        TimeGenerated     = $completedUtc.ToString('o')
        CollectorName     = $CollectorName
        Computer          = $env:COMPUTERNAME
        TargetScope       = $TargetScope
        ExecutionId       = $ExecutionId
        RecordsCollected  = $RecordsCollected
        RecordsSubmitted  = $RecordsSubmitted
        DurationSeconds   = [Math]::Round(
            ($completedUtc - $StartedUtc).TotalSeconds,
            3
        )
        Status            = $Status
        ErrorCode         = $ErrorCode
        ErrorMessage      = $ErrorMessage
        CollectorVersion  = $CollectorVersion
    }
}

Export-ModuleMember -Function Read-CollectorConfiguration, New-CollectorHealthRecord
```

`CollectorHealth_CL` will be created in Phase 2. Until then, local logging remains the primary evidence of task success/failure.

---

# 7. Deployment Package and Release Model

## 7.1 Release Manifest

Every deployed release should include a manifest.

```json name=release.json
{
  "PackageName": "Contoso.AzureMonitorCollectors",
  "Version": "1.0.0",
  "ReleaseDateUtc": "2026-09-22T00:00:00Z",
  "Collectors": [
    "CertificateInventory",
    "ErpFolderMetrics"
  ],
  "MinimumPowerShellVersion": "5.1",
  "RequiredArcManagedIdentity": true,
  "ReleaseNotes": "Initial pilot implementation."
}
```

## 7.2 Deployment Pattern

```text
Build/CI pipeline
    │
    ├── Lint PowerShell
    ├── Run Pester tests
    ├── Sign scripts/modules
    ├── Package release
    │
    ▼
Approved deployment share or software distribution platform
    │
    ▼
Target Windows server
    │
    ├── Validate prerequisites
    ├── Copy package to staging
    ├── Verify signatures/hashes
    ├── Move Current → Previous
    ├── Promote staging → Current
    ├── Deploy configuration
    ├── Register/update scheduled tasks
    └── Run validation execution
```

## 7.3 Recommended Deployment Methods

| Environment | Preferred deployment method |
|---|---|
| Pilot | Controlled PowerShell deployment script or enterprise software distribution system |
| Test | CI/CD-controlled deployment using approved automation |
| Production | CI/CD pipeline plus enterprise endpoint-management or configuration-management platform |

Avoid administrators manually copying scripts to production servers. Manual deployment causes version drift and prevents reliable rollback.

---

# 8. Script Signing and Execution Policy

## 8.1 Production Standard

Use:

```text
ExecutionPolicy: AllSigned
```

All `.ps1` and `.psm1` files must be signed by the organization’s code-signing certificate.

## 8.2 Certificate Requirements

The code-signing certificate should:

- Be issued by the enterprise PKI or approved public CA.
- Include Code Signing enhanced key usage.
- Be protected in the CI/CD signing process.
- Not be copied as an exportable private key to ERP servers.
- Have its issuing chain trusted on all collector hosts.
- Be monitored for expiry itself.

## 8.3 Pre-Execution Signature Validation

The deployment script should verify signatures:

```powershell name=Test-CollectorScriptSignatures.ps1
param(
    [Parameter(Mandatory)]
    [string] $CollectorRoot
)

$files = Get-ChildItem `
    -Path $CollectorRoot `
    -Recurse `
    -File `
    -Include '*.ps1', '*.psm1'

$invalid = foreach ($file in $files) {
    $signature = Get-AuthenticodeSignature -FilePath $file.FullName

    if ($signature.Status -ne 'Valid') {
        [pscustomobject]@{
            File   = $file.FullName
            Status = $signature.Status
            Detail = $signature.StatusMessage
        }
    }
}

if ($invalid) {
    $invalid | Format-Table -AutoSize | Out-String | Write-Error
    throw 'Collector package contains unsigned or invalidly signed scripts.'
}

Write-Output 'All collector scripts and modules have valid signatures.'
```

---

# 9. NTFS Permissions Model

Use explicit permissions rather than inheriting broad access from a shared parent folder.

| Folder | Deployment account | Collector account | Local administrators | Support analysts |
|---|---|---|---|---|
| `Current` | Modify | Read & Execute | Full control | Read & Execute |
| `Config` | Modify | Read | Full control | Read |
| `Logs` | Modify | Modify | Full control | Read |
| `Spool` | Modify | Modify | Full control | Read |
| `Locks` | Modify | Modify | Full control | No access required |
| `Previous` | Modify | Read & Execute | Full control | Read & Execute |

The collector account should never have write access to `Current\Modules` or `Current\Collectors`; otherwise it could alter its own signed executable content.

---

# 10. Task Scheduler Deployment Pattern

## Certificate Collector

- Runs daily.
- Starts at a staggered time to avoid all servers posting simultaneously.
- Typical pilot schedule: 02:15 local time.
- Maximum runtime: 15 minutes.
- Do not overlap execution.

## ERP Folder Collector

- Runs every 15–60 minutes according to log growth risk.
- Start with **hourly** if the directory is large or file count is unknown.
- Move to 15 minutes only after scan duration and server impact are validated.
- Maximum runtime: 20–30 minutes.
- Do not overlap execution.

## Task Naming

```text
Contoso-Monitor-CertificateInventory
Contoso-Monitor-ErpFolderMetrics
```

Use a predictable organization prefix so operations can locate and audit all monitoring tasks.

---

# 11. Local Logging, Spooling, and Failure Behavior

## 11.1 Expected Behavior

| Scenario | Required collector behavior |
|---|---|
| Certificate/folder data is collected and ingestion succeeds | Write local completion log; send primary records and collector-health record. |
| Azure token cannot be obtained | Write sanitized local failure log; return non-zero exit code; do not expose token details. |
| Azure endpoint temporarily unavailable | Retry with exponential backoff; spool batch if retries fail. |
| DCR rejects invalid payload | Write sanitized local error; do not retry indefinitely; flag configuration/schema incident. |
| Folder scan partially fails | Send available metrics with `PartialSuccess` and access-error count. |
| Collector already running | Exit safely with known exit code and local warning. |
| Task fails repeatedly | Task Scheduler records failure; future `CollectorHealth_CL` missing-data alert identifies the gap. |

## 11.2 Spool Controls

The optional spool feature must:

- Store only already-sanitized JSON payloads.
- Use restrictive NTFS permissions.
- Use one file per batch and collector execution.
- Delete spool data after confirmed successful submission.
- Delete stale spool files after the approved maximum retention window.
- Alert if spool-file count or size exceeds a threshold.
- Never spool bearer tokens or raw error responses.

---

# 12. Pre-Deployment Validation Script Requirements

Before installation, validate:

1. Windows version.
2. PowerShell version.
3. Azure Arc agent installed and connected.
4. Azure Arc system-assigned identity enabled.
5. `IDENTITY_ENDPOINT` available to the scheduled-task account.
6. Task Scheduler operational.
7. Certificate store available.
8. IIS module present if IIS binding discovery is enabled.
9. ERP log folder exists and is readable.
10. Required local directories can be created.
11. Required outbound HTTPS endpoints are reachable.
12. Script signatures are valid.
13. DCR endpoint, immutable ID, and stream-name configuration exists.

A design-level sample:

```powershell name=Test-CollectorPrerequisites.ps1
[CmdletBinding()]
param(
    [Parameter(Mandatory)]
    [string] $CollectorRoot,

    [Parameter(Mandatory)]
    [string] $ErpLogPath,

    [switch] $RequireIisBindingDiscovery
)

$failures = [System.Collections.Generic.List[string]]::new()

if ($PSVersionTable.PSVersion.Major -lt 5) {
    $failures.Add('PowerShell 5.1 or later is required.')
}

if (-not (Get-Service -Name 'himds' -ErrorAction SilentlyContinue)) {
    $failures.Add('Azure Arc HIMDS service is not detected. Confirm Azure Arc onboarding.')
}

if ([string]::IsNullOrWhiteSpace($env:IDENTITY_ENDPOINT)) {
    $failures.Add('IDENTITY_ENDPOINT is unavailable. Confirm Azure Arc managed identity.')
}

if (-not (Test-Path -LiteralPath 'Cert:\LocalMachine\WebHosting')) {
    $failures.Add('Certificate store Cert:\LocalMachine\WebHosting is unavailable.')
}

if (-not (Test-Path -LiteralPath $ErpLogPath -PathType Container)) {
    $failures.Add("ERP log directory was not found: $ErpLogPath")
}

if ($RequireIisBindingDiscovery -and
    -not (Get-Module -ListAvailable -Name WebAdministration)) {
    $failures.Add(
        'WebAdministration module is unavailable but IIS binding discovery is enabled.'
    )
}

try {
    New-Item -Path $CollectorRoot -ItemType Directory -Force -ErrorAction Stop | Out-Null
}
catch {
    $failures.Add("Unable to create or access collector root: $CollectorRoot")
}

if ($failures.Count -gt 0) {
    $failures | ForEach-Object { Write-Error $_ }
    exit 1
}

Write-Output 'Collector prerequisite validation succeeded.'
exit 0
```

The deployment process must run this script using the same identity as the planned scheduled task, not only as a privileged administrator.

---

# 13. Phase 1 Acceptance Criteria

Phase 1 is complete when all conditions below are met.

| Area | Acceptance criterion |
|---|---|
| Repository | Project structure exists in source control. |
| Shared modules | Logging, locking, Arc identity, configuration, and health-record modules exist and pass initial code review. |
| Windows package | Package deploys into the defined `ProgramData` structure. |
| Configuration | Certificate and ERP-folder configurations are externalized from scripts. |
| Security | Scripts/modules are signed; NTFS ACLs are applied; no Azure secrets exist locally. |
| Managed identity | Task run-as account can request an Azure Arc managed-identity token. |
| Scheduler | Certificate and folder Scheduled Tasks are registered, non-overlapping, and run under the approved account. |
| Logging | Each execution produces a sanitized local operational log. |
| Locks | A second simultaneous collector execution is prevented. |
| Validation | Prerequisite script passes on pilot servers. |
| Rollback | Current release can be replaced by the prior signed release. |
| Documentation | Deployment, upgrade, rollback, and local troubleshooting instructions are drafted. |

---

# 14. Phase 1 Deliverables

1. Source repository with approved structure.
2. Signed PowerShell module skeleton.
3. Configuration templates.
4. Windows local deployment layout.
5. Task Scheduler installation scripts.
6. Prerequisite-validation script.
7. Script-signature validation script.
8. NTFS permissions implementation guide.
9. Release manifest format.
10. Deployment, rollback, and local troubleshooting runbook.

---

# 15. Handoff to Phase 2

Phase 2 will implement the Azure-side infrastructure in Bicep:

- Dedicated Log Analytics Workspace.
- Custom tables:
  - `ServerCertificateInventory_CL`
  - `ErpFolderMetrics_CL`
  - `CollectorHealth_CL`
- DCR stream declarations and data flows.
- Public ingestion endpoint versus DCE validation.
- DCR-scope RBAC for each Azure Arc managed identity.
- Test ingestion of a single certificate and folder-metrics payload.
- Foundation for Teams and Outlook/email Action Groups.

### Want me to…
<!-- copilot:suggestions -->
1. Continue to Phase 2 and provide the Bicep design for Log Analytics custom tables, DCRs, identity role assignments, and test ingestion.
2. Expand Phase 1 into a complete copy-ready PowerShell package, including installer, rollback, spool/retry, and Scheduled Task scripts.
3. Hand this change off to the coding agent to open a pull request in a repository you provide.
