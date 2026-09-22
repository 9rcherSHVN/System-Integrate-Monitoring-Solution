
# Custom PowerShell Collector Design for the Pilot

The three requirements are appropriate for custom collection, but they should be implemented as **small, independent collectors**, not one large script:

1. **Certificate inventory collector** — runs locally on each Windows server.
2. **Service-account lifecycle collector** — normally runs centrally from a domain-connected management server or Hybrid Runbook Worker.
3. **ERP log-directory capacity collector** — runs locally on ERP/web servers.

Each collector should produce structured records and send them through:

```text
PowerShell collector
    → JSON records
    → Azure Monitor Logs Ingestion API
    → Custom DCR stream
    → Log Analytics custom table
    → KQL / Workbook / Alert
```

A DCR does not execute the scripts. Use Windows Task Scheduler, Azure Automation Hybrid Runbook Worker, or another approved orchestration platform to run them.

---

# A. Common Design Guidelines

## A.1 Separate Collection from Ingestion

Use two logical layers:

- **Collection functions** gather and normalize local or directory data.
- A shared **ingestion function** sends records to Azure Monitor.

This separation allows the collection logic to be tested locally without sending data to Azure.

Recommended files:

```text
Collectors/
├── AzureMonitorIngestion.psm1
├── Collect-CertificateInventory.ps1
├── Collect-ServiceAccountStatus.ps1
└── Collect-ErpFolderMetrics.ps1
```

## A.2 Authentication Design

Preferred authentication order:

1. **Managed identity** when running under Azure Automation Hybrid Worker or another supported managed execution context.
2. **Certificate-based service principal** if managed identity is unavailable.
3. Client secret only as a temporary pilot method, stored in Azure Key Vault or an approved enterprise secret vault.

Never embed these in scripts:

- Passwords
- Client secrets
- Access tokens
- LAPS passwords
- Private certificate keys

The collector identity requires only the Azure permission necessary to submit data through its DCR. It should not have permission to read all Log Analytics data or administer the workspace.

## A.3 Scheduling Recommendations

| Collector | Pilot frequency |
|---|---:|
| Certificate inventory | Every 12–24 hours |
| Domain service-account status | Every 6–24 hours |
| Entra-backed LAPS metadata | Every 12–24 hours |
| ERP log-folder size | Every 15–60 minutes |
| Collector health record | Every execution |

These are inventory and capacity datasets; collecting them every minute creates unnecessary cost.

## A.4 Required Operational Controls

Every collector should implement:

- UTC timestamps.
- Stable server and resource identifiers.
- Structured error handling.
- Non-zero process exit code on failure.
- Local operational logging.
- Retry with exponential backoff for transient Azure errors.
- Payload batching.
- Collection duration measurement.
- A collector-version field.
- No sensitive values in logs.
- Idempotent execution.
- Code signing for production deployment.
- Least-privilege filesystem, Active Directory, Graph, and Azure permissions.

## A.5 Recommended Custom Tables

| Table | Purpose |
|---|---|
| `ServerCertificateInventory_CL` | Windows certificate-store and IIS binding inventory |
| `ServiceAccountStatus_CL` | Services, scheduled tasks, IIS app-pool accounts, AD account status |
| `LapsDeviceStatus_CL` | Entra-backed Windows LAPS metadata without password values |
| `ErpFolderMetrics_CL` | ERP/IIS/middleware directory size, file count, and age |
| `CollectorHealth_CL` | Collector execution success, duration, and error status |

---

# B. Common Azure Monitor Ingestion Module

The following sample uses the Logs Ingestion REST API. The calling script must first authenticate using `Connect-AzAccount`, preferably with managed identity or certificate-based service-principal authentication.

The DCR must define a custom stream whose schema matches the submitted records.

```powershell name=AzureMonitorIngestion.psm1
Set-StrictMode -Version Latest

function ConvertFrom-SecureStringToPlainText {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)]
        [SecureString] $SecureString
    )

    $bstr = [Runtime.InteropServices.Marshal]::SecureStringToBSTR($SecureString)

    try {
        [Runtime.InteropServices.Marshal]::PtrToStringBSTR($bstr)
    }
    finally {
        [Runtime.InteropServices.Marshal]::ZeroFreeBSTR($bstr)
    }
}

function Get-AzureMonitorBearerToken {
    [CmdletBinding()]
    param()

    $tokenResult = Get-AzAccessToken -ResourceUrl 'https://monitor.azure.com/'

    if ($tokenResult.Token -is [SecureString]) {
        return ConvertFrom-SecureStringToPlainText -SecureString $tokenResult.Token
    }

    return [string]$tokenResult.Token
}

function Send-AzureMonitorRecords {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)]
        [ValidatePattern('^https://')]
        [string] $Endpoint,

        [Parameter(Mandatory)]
        [string] $DcrImmutableId,

        [Parameter(Mandatory)]
        [ValidatePattern('^Custom-')]
        [string] $StreamName,

        [Parameter(Mandatory)]
        [object[]] $Records,

        [ValidateRange(1, 10)]
        [int] $MaximumAttempts = 4,

        [ValidateRange(1, 1000)]
        [int] $MaximumRecordsPerBatch = 500,

        [ValidateRange(100000, 950000)]
        [int] $MaximumPayloadBytes = 900000
    )

    if ($Records.Count -eq 0) {
        return
    }

    $normalizedEndpoint = $Endpoint.TrimEnd('/')
    $escapedDcr = [Uri]::EscapeDataString($DcrImmutableId)
    $escapedStream = [Uri]::EscapeDataString($StreamName)

    $uri = (
        '{0}/dataCollectionRules/{1}/streams/{2}?api-version=2023-01-01' -f
        $normalizedEndpoint,
        $escapedDcr,
        $escapedStream
    )

    $token = Get-AzureMonitorBearerToken
    $headers = @{
        Authorization = "Bearer $token"
    }

    $batches = [System.Collections.Generic.List[object]]::new()
    $currentBatch = [System.Collections.Generic.List[object]]::new()
    $currentBytes = 2

    foreach ($record in $Records) {
        $recordJson = $record | ConvertTo-Json -Depth 12 -Compress
        $recordBytes = [Text.Encoding]::UTF8.GetByteCount($recordJson) + 1

        if ($recordBytes -gt $MaximumPayloadBytes) {
            throw "A single telemetry record exceeds the configured payload limit."
        }

        $batchWouldOverflow =
            $currentBatch.Count -ge $MaximumRecordsPerBatch -or
            ($currentBytes + $recordBytes) -gt $MaximumPayloadBytes

        if ($batchWouldOverflow -and $currentBatch.Count -gt 0) {
            $batches.Add($currentBatch.ToArray())
            $currentBatch = [System.Collections.Generic.List[object]]::new()
            $currentBytes = 2
        }

        $currentBatch.Add($record)
        $currentBytes += $recordBytes
    }

    if ($currentBatch.Count -gt 0) {
        $batches.Add($currentBatch.ToArray())
    }

    foreach ($batch in $batches) {
        $body = ConvertTo-Json -InputObject @($batch) -Depth 12 -Compress
        $attempt = 0

        while ($true) {
            $attempt++

            try {
                Invoke-RestMethod `
                    -Method Post `
                    -Uri $uri `
                    -Headers $headers `
                    -ContentType 'application/json; charset=utf-8' `
                    -Body ([Text.Encoding]::UTF8.GetBytes($body)) `
                    -ErrorAction Stop | Out-Null

                break
            }
            catch {
                $statusCode = $null

                if ($_.Exception.Response) {
                    try {
                        $statusCode = [int]$_.Exception.Response.StatusCode
                    }
                    catch {
                        $statusCode = $null
                    }
                }

                $retryable = $statusCode -in @(408, 429, 500, 502, 503, 504)

                if (-not $retryable -or $attempt -ge $MaximumAttempts) {
                    throw
                }

                $delaySeconds = [Math]::Min(60, [Math]::Pow(2, $attempt))
                Start-Sleep -Seconds $delaySeconds
            }
        }
    }
}

Export-ModuleMember -Function Send-AzureMonitorRecords
```

## Example Authentication

For a supported managed-identity execution environment:

```powershell name=Connect-CollectorWithManagedIdentity.ps1
Import-Module Az.Accounts
Connect-AzAccount -Identity -ErrorAction Stop | Out-Null
```

For certificate-based service-principal authentication:

```powershell name=Connect-CollectorWithCertificate.ps1
param(
    [Parameter(Mandatory)]
    [string] $TenantId,

    [Parameter(Mandatory)]
    [string] $ApplicationId,

    [Parameter(Mandatory)]
    [string] $CertificateThumbprint
)

Import-Module Az.Accounts

Connect-AzAccount `
    -ServicePrincipal `
    -Tenant $TenantId `
    -ApplicationId $ApplicationId `
    -CertificateThumbprint $CertificateThumbprint `
    -ErrorAction Stop | Out-Null
```

Do not use the certificate being inventoried as the collector authentication certificate unless that design has been explicitly approved.

---

# 1. Windows Certificate-Store Inventory

## 1.1 Technical Scope

Inventory certificates from selected stores, normally:

- `Cert:\LocalMachine\My`
- `Cert:\LocalMachine\WebHosting`
- Optionally application-specific stores

Do not collect every certificate from `Root`, `CA`, and `TrustedPublisher` unless there is a defined compliance requirement. Those stores can create significant noise.

For application-facing certificates, correlate certificate inventory with:

- IIS HTTPS bindings.
- ERP application configuration.
- Middleware/reverse-proxy configuration.
- SQL Server TLS configuration.
- Certificate owner and renewal process.

## 1.2 Fields to Collect

| Field | Purpose |
|---|---|
| `TimeGenerated` | Collection timestamp in UTC |
| `Computer` | Server name |
| `StoreLocation` / `StoreName` | Certificate-store source |
| `Subject` | Certificate subject |
| `DnsNames` | Subject Alternative Names where available |
| `Issuer` | Issuing authority |
| `Thumbprint` | Stable certificate identifier |
| `SerialNumber` | Issuer-assigned serial |
| `NotBefore` / `NotAfter` | Validity period |
| `DaysUntilExpiry` | Alert-friendly calculated value |
| `HasPrivateKey` | Whether a private key is associated |
| `EnhancedKeyUsage` | Intended usages |
| `IisBindings` | IIS sites/endpoints using the certificate |
| `CollectionStatus` | Success or error |

Do not export:

- Private key material
- PFX content
- PINs/passwords
- Authentication secrets

## 1.3 Permissions

Reading `LocalMachine` certificate stores normally requires appropriate local rights. IIS binding discovery may require administrator privileges or delegated IIS configuration access.

The production design should prefer a narrowly delegated collector account instead of unrestricted Domain Admin privileges.

## 1.4 Certificate Inventory Script

```powershell name=Collect-CertificateInventory.ps1
[CmdletBinding()]
param(
    [string[]] $CertificateStores = @(
        'Cert:\LocalMachine\My',
        'Cert:\LocalMachine\WebHosting'
    ),

    [Parameter(Mandatory)]
    [string] $IngestionEndpoint,

    [Parameter(Mandatory)]
    [string] $DcrImmutableId,

    [string] $StreamName = 'Custom-ServerCertificateInventory'
)

Set-StrictMode -Version Latest
$ErrorActionPreference = 'Stop'

Import-Module "$PSScriptRoot\AzureMonitorIngestion.psm1" -Force

function Get-CertificateDnsNames {
    param(
        [Parameter(Mandatory)]
        [System.Security.Cryptography.X509Certificates.X509Certificate2] $Certificate
    )

    $names = [System.Collections.Generic.List[string]]::new()

    if ($Certificate.PSObject.Properties.Name -contains 'DnsNameList') {
        foreach ($dnsName in $Certificate.DnsNameList) {
            if ($dnsName.Unicode) {
                $names.Add([string]$dnsName.Unicode)
            }
        }
    }

    return ($names | Sort-Object -Unique) -join ';'
}

function Get-IisCertificateBindingMap {
    $map = @{}

    if (-not (Get-Module -ListAvailable -Name WebAdministration)) {
        return $map
    }

    Import-Module WebAdministration -ErrorAction Stop

    foreach ($site in Get-Website) {
        foreach ($binding in Get-WebBinding -Name $site.Name -Protocol https) {
            $hash = $binding.certificateHash

            if ($hash -is [byte[]]) {
                $thumbprint = ([BitConverter]::ToString($hash)).Replace('-', '')
            }
            else {
                $thumbprint = ([string]$hash).Replace(' ', '').ToUpperInvariant()
            }

            if ([string]::IsNullOrWhiteSpace($thumbprint)) {
                continue
            }

            $description = '{0}|{1}|{2}' -f `
                $site.Name,
                $binding.bindingInformation,
                $binding.certificateStoreName

            if (-not $map.ContainsKey($thumbprint)) {
                $map[$thumbprint] = [System.Collections.Generic.List[string]]::new()
            }

            $map[$thumbprint].Add($description)
        }
    }

    return $map
}

$collectionTime = (Get-Date).ToUniversalTime()
$iisBindings = @{}

try {
    $iisBindings = Get-IisCertificateBindingMap
}
catch {
    Write-Warning "IIS binding discovery failed: $($_.Exception.Message)"
}

$records = [System.Collections.Generic.List[object]]::new()

foreach ($storePath in $CertificateStores) {
    if (-not (Test-Path -LiteralPath $storePath)) {
        $records.Add([pscustomobject]@{
            TimeGenerated   = $collectionTime.ToString('o')
            Computer        = $env:COMPUTERNAME
            StoreLocation   = 'LocalMachine'
            StoreName       = Split-Path $storePath -Leaf
            Subject         = $null
            DnsNames        = $null
            Issuer          = $null
            Thumbprint      = $null
            SerialNumber    = $null
            NotBefore       = $null
            NotAfter        = $null
            DaysUntilExpiry = $null
            HasPrivateKey   = $null
            EnhancedKeyUsage = $null
            IisBindings     = $null
            CollectionStatus = 'StoreNotFound'
            CollectorVersion = '1.0.0'
        })

        continue
    }

    foreach ($certificate in Get-ChildItem -LiteralPath $storePath -ErrorAction Stop) {
        $thumbprint = $certificate.Thumbprint.Replace(' ', '').ToUpperInvariant()
        $bindingList = $null

        if ($iisBindings.ContainsKey($thumbprint)) {
            $bindingList = ($iisBindings[$thumbprint] | Sort-Object -Unique) -join ';'
        }

        $eku = @(
            foreach ($usage in $certificate.EnhancedKeyUsageList) {
                if ($usage.FriendlyName) {
                    $usage.FriendlyName
                }
                else {
                    $usage.ObjectId.Value
                }
            }
        ) -join ';'

        $records.Add([pscustomobject]@{
            TimeGenerated    = $collectionTime.ToString('o')
            Computer         = $env:COMPUTERNAME
            StoreLocation    = 'LocalMachine'
            StoreName        = Split-Path $storePath -Leaf
            Subject          = $certificate.Subject
            DnsNames         = Get-CertificateDnsNames -Certificate $certificate
            Issuer           = $certificate.Issuer
            Thumbprint       = $thumbprint
            SerialNumber     = $certificate.SerialNumber
            NotBefore        = $certificate.NotBefore.ToUniversalTime().ToString('o')
            NotAfter         = $certificate.NotAfter.ToUniversalTime().ToString('o')
            DaysUntilExpiry  = [Math]::Floor(
                ($certificate.NotAfter.ToUniversalTime() - $collectionTime).TotalDays
            )
            HasPrivateKey    = [bool]$certificate.HasPrivateKey
            EnhancedKeyUsage = $eku
            IisBindings      = $bindingList
            CollectionStatus = 'Success'
            CollectorVersion = '1.0.0'
        })
    }
}

Send-AzureMonitorRecords `
    -Endpoint $IngestionEndpoint `
    -DcrImmutableId $DcrImmutableId `
    -StreamName $StreamName `
    -Records $records.ToArray()

Write-Output "Submitted $($records.Count) certificate inventory records."
```

## 1.5 Certificate Alert Query

```kusto name=CertificateExpiryAlert.kql
ServerCertificateInventory_CL
| where TimeGenerated > ago(2d)
| summarize arg_max(TimeGenerated, *) by Computer, Thumbprint
| where CollectionStatus == "Success"
| where DaysUntilExpiry <= 30
| project
    TimeGenerated,
    Computer,
    StoreName,
    Subject,
    DnsNames,
    NotAfter,
    DaysUntilExpiry,
    IisBindings,
    Thumbprint
| order by DaysUntilExpiry asc
```

A production design should also alert if the inventory collector stops reporting, because absence of certificate records is not evidence that certificates are healthy.

---

# 2. Service-Account Password Expiry and Windows LAPS

## 2.1 Important Architecture Clarification

A **Windows LAPS-managed account is normally a local administrator account on a device**. It is not the same thing as:

- A domain service account.
- A gMSA.
- An Entra application/service principal.
- An IIS application-pool identity.
- A SQL login.

Microsoft Entra ID may store Windows LAPS password backup data for an eligible Entra-joined or hybrid-joined device. Merely onboarding a server to Azure Arc does not automatically make it an Entra-joined Windows LAPS device.

The pilot should therefore separate two datasets:

1. **Service identities actually configured on services, tasks, and app pools.**
2. **LAPS device metadata**, such as backup time and expiry time.

Never retrieve or ingest the actual LAPS password.

## 2.2 Service-Account Categories

| Identity type | Expiry interpretation |
|---|---|
| Domain user service account | May have a calculated password-expiry date |
| gMSA | Password rotation is managed automatically; conventional password expiry alerts do not apply |
| Built-in virtual account | No password-expiry monitoring required |
| Local user account | Requires local account policy inspection; should not normally be used broadly across servers |
| Entra service principal | Uses certificates, secrets, or workload identity—not a Windows password |
| LAPS-managed local admin | Monitor backup/rotation metadata, not the password |
| SQL login | Separate SQL credential lifecycle; not covered by AD password expiry |

## 2.3 Recommended Collection Architecture

Run domain-account collection centrally from a hardened domain-connected server with:

- ActiveDirectory PowerShell module.
- Read access to account metadata.
- Remote visibility of services/tasks/app pools, or locally produced inventories.
- No permission to reset or retrieve passwords.

A scalable design is:

1. Each server reports account references used by services/tasks/app pools.
2. A central collector resolves those references against Active Directory.
3. LAPS metadata is queried separately through approved Microsoft Graph/LAPS APIs.
4. Records are joined in Log Analytics by normalized account/device identifier.

## 2.4 Domain Service-Account Collector

The following pilot sample inventories Windows service identities on the local server and resolves domain user/gMSA status through Active Directory.

It intentionally does not retrieve passwords.

```powershell name=Collect-ServiceAccountStatus.ps1
[CmdletBinding()]
param(
    [Parameter(Mandatory)]
    [string] $IngestionEndpoint,

    [Parameter(Mandatory)]
    [string] $DcrImmutableId,

    [string] $StreamName = 'Custom-ServiceAccountStatus',

    [switch] $IncludeScheduledTasks,

    [switch] $IncludeIisApplicationPools
)

Set-StrictMode -Version Latest
$ErrorActionPreference = 'Stop'

Import-Module "$PSScriptRoot\AzureMonitorIngestion.psm1" -Force
Import-Module ActiveDirectory -ErrorAction Stop

function Test-BuiltInServiceIdentity {
    param([string] $AccountName)

    $normalized = $AccountName.Trim().ToUpperInvariant()

    return $normalized -in @(
        'LOCALSYSTEM',
        'NT AUTHORITY\SYSTEM',
        'NT AUTHORITY\LOCALSERVICE',
        'NT AUTHORITY\NETWORKSERVICE',
        'NT SERVICE\ALL SERVICES'
    ) -or $normalized.StartsWith('NT SERVICE\') `
       -or $normalized.StartsWith('IIS APPPOOL\')
}

function Get-NormalizedSamAccountName {
    param([string] $AccountName)

    if ($AccountName -match '^[^\\]+\\(.+)$') {
        return $Matches[1]
    }

    if ($AccountName -match '^([^@]+)@') {
        return $Matches[1]
    }

    return $AccountName
}

function Convert-AdFileTime {
    param($Value)

    if ($null -eq $Value) {
        return $null
    }

    $numericValue = [Int64]$Value

    if ($numericValue -le 0 -or $numericValue -eq [Int64]::MaxValue) {
        return $null
    }

    return [DateTime]::FromFileTimeUtc($numericValue)
}

function Resolve-DirectoryAccount {
    param(
        [Parameter(Mandatory)]
        [string] $AccountName
    )

    if (Test-BuiltInServiceIdentity -AccountName $AccountName) {
        return [pscustomobject]@{
            AccountType       = 'BuiltInOrVirtual'
            SamAccountName    = $AccountName
            DistinguishedName = $null
            Enabled           = $true
            LockedOut         = $false
            PasswordLastSet   = $null
            PasswordExpires   = $null
            DaysUntilExpiry   = $null
            PasswordNeverExpires = $true
            ManagedPassword   = $true
            ResolutionStatus  = 'NotApplicable'
        }
    }

    $sam = Get-NormalizedSamAccountName -AccountName $AccountName

    if ($sam.EndsWith('$')) {
        try {
            $gmsa = Get-ADServiceAccount `
                -Identity $sam `
                -Properties Enabled, DistinguishedName `
                -ErrorAction Stop

            return [pscustomobject]@{
                AccountType       = 'gMSA'
                SamAccountName    = $gmsa.SamAccountName
                DistinguishedName = $gmsa.DistinguishedName
                Enabled           = [bool]$gmsa.Enabled
                LockedOut         = $false
                PasswordLastSet   = $null
                PasswordExpires   = $null
                DaysUntilExpiry   = $null
                PasswordNeverExpires = $true
                ManagedPassword   = $true
                ResolutionStatus  = 'Resolved'
            }
        }
        catch {
            # A trailing dollar sign can also represent a computer account.
        }
    }

    try {
        $user = Get-ADUser `
            -Identity $sam `
            -Properties @(
                'Enabled',
                'LockedOut',
                'PasswordLastSet',
                'PasswordNeverExpires',
                'msDS-UserPasswordExpiryTimeComputed',
                'DistinguishedName'
            ) `
            -ErrorAction Stop

        $expiry = Convert-AdFileTime `
            -Value $user.'msDS-UserPasswordExpiryTimeComputed'

        $daysUntilExpiry = $null

        if ($expiry) {
            $daysUntilExpiry = [Math]::Floor(
                ($expiry - (Get-Date).ToUniversalTime()).TotalDays
            )
        }

        return [pscustomobject]@{
            AccountType       = 'DomainUser'
            SamAccountName    = $user.SamAccountName
            DistinguishedName = $user.DistinguishedName
            Enabled           = [bool]$user.Enabled
            LockedOut         = [bool]$user.LockedOut
            PasswordLastSet   = if ($user.PasswordLastSet) {
                $user.PasswordLastSet.ToUniversalTime().ToString('o')
            } else {
                $null
            }
            PasswordExpires   = if ($expiry) {
                $expiry.ToString('o')
            } else {
                $null
            }
            DaysUntilExpiry   = $daysUntilExpiry
            PasswordNeverExpires = [bool]$user.PasswordNeverExpires
            ManagedPassword   = $false
            ResolutionStatus  = 'Resolved'
        }
    }
    catch {
        return [pscustomobject]@{
            AccountType       = 'UnknownOrLocal'
            SamAccountName    = $sam
            DistinguishedName = $null
            Enabled           = $null
            LockedOut         = $null
            PasswordLastSet   = $null
            PasswordExpires   = $null
            DaysUntilExpiry   = $null
            PasswordNeverExpires = $null
            ManagedPassword   = $false
            ResolutionStatus  = 'NotResolved'
        }
    }
}

$references = [System.Collections.Generic.List[object]]::new()

foreach ($service in Get-CimInstance -ClassName Win32_Service) {
    if ([string]::IsNullOrWhiteSpace($service.StartName)) {
        continue
    }

    $references.Add([pscustomobject]@{
        UsageType   = 'WindowsService'
        UsageName   = $service.Name
        DisplayName = $service.DisplayName
        AccountName = $service.StartName
    })
}

if ($IncludeScheduledTasks) {
    foreach ($task in Get-ScheduledTask) {
        $account = $task.Principal.UserId

        if ([string]::IsNullOrWhiteSpace($account)) {
            continue
        }

        $references.Add([pscustomobject]@{
            UsageType   = 'ScheduledTask'
            UsageName   = "$($task.TaskPath)$($task.TaskName)"
            DisplayName = $task.TaskName
            AccountName = $account
        })
    }
}

if ($IncludeIisApplicationPools) {
    if (Get-Module -ListAvailable -Name WebAdministration) {
        Import-Module WebAdministration -ErrorAction Stop

        foreach ($appPool in Get-ChildItem IIS:\AppPools) {
            $identityType = [string]$appPool.processModel.identityType
            $account = [string]$appPool.processModel.userName

            if ([string]::IsNullOrWhiteSpace($account)) {
                $account = "IIS APPPOOL\$($appPool.Name) [$identityType]"
            }

            $references.Add([pscustomobject]@{
                UsageType   = 'IISApplicationPool'
                UsageName   = $appPool.Name
                DisplayName = $appPool.Name
                AccountName = $account
            })
        }
    }
}

$collectionTime = (Get-Date).ToUniversalTime().ToString('o')
$records = [System.Collections.Generic.List[object]]::new()
$resolutionCache = @{}

foreach ($reference in $references) {
    if (-not $resolutionCache.ContainsKey($reference.AccountName)) {
        $resolutionCache[$reference.AccountName] =
            Resolve-DirectoryAccount -AccountName $reference.AccountName
    }

    $identity = $resolutionCache[$reference.AccountName]

    $records.Add([pscustomobject]@{
        TimeGenerated        = $collectionTime
        Computer             = $env:COMPUTERNAME
        UsageType            = $reference.UsageType
        UsageName            = $reference.UsageName
        DisplayName          = $reference.DisplayName
        ConfiguredAccount    = $reference.AccountName
        AccountType          = $identity.AccountType
        SamAccountName       = $identity.SamAccountName
        DistinguishedName    = $identity.DistinguishedName
        Enabled              = $identity.Enabled
        LockedOut            = $identity.LockedOut
        PasswordLastSet      = $identity.PasswordLastSet
        PasswordExpires      = $identity.PasswordExpires
        DaysUntilExpiry      = $identity.DaysUntilExpiry
        PasswordNeverExpires = $identity.PasswordNeverExpires
        ManagedPassword      = $identity.ManagedPassword
        ResolutionStatus     = $identity.ResolutionStatus
        CollectorVersion     = '1.0.0'
    })
}

Send-AzureMonitorRecords `
    -Endpoint $IngestionEndpoint `
    -DcrImmutableId $DcrImmutableId `
    -StreamName $StreamName `
    -Records $records.ToArray()

Write-Output "Submitted $($records.Count) service-account status records."
```

### Security limitation

`msDS-UserPasswordExpiryTimeComputed` is useful for domain user accounts, but results depend on:

- Effective domain password policy.
- Fine-grained password policies.
- `PasswordNeverExpires`.
- Account type.
- Directory replication.
- Collector authorization.

For gMSAs, monitor whether the service is configured with a gMSA and whether the host can retrieve its managed password—not a conventional expiry date.

A useful local validation is:

```powershell name=Test-GmsaOnServer.ps1
param(
    [Parameter(Mandatory)]
    [string] $Identity
)

Import-Module ActiveDirectory
Test-ADServiceAccount -Identity $Identity
```

## 2.5 Entra-Backed Windows LAPS Metadata

Run this collector centrally, not separately on every ERP server. Use only metadata-level Graph/LAPS permission and **do not request password values**.

The precise properties returned can depend on module/API version, so the script treats the LAPS result defensively.

```powershell name=Collect-EntraLapsMetadata.ps1
[CmdletBinding()]
param(
    [Parameter(Mandatory)]
    [string[]] $DeviceIds,

    [Parameter(Mandatory)]
    [string] $IngestionEndpoint,

    [Parameter(Mandatory)]
    [string] $DcrImmutableId,

    [string] $StreamName = 'Custom-LapsDeviceStatus'
)

Set-StrictMode -Version Latest
$ErrorActionPreference = 'Stop'

Import-Module LAPS -ErrorAction Stop
Import-Module "$PSScriptRoot\AzureMonitorIngestion.psm1" -Force

$collectionTime = (Get-Date).ToUniversalTime()
$records = [System.Collections.Generic.List[object]]::new()

foreach ($deviceId in $DeviceIds) {
    try {
        # Deliberately omit any option that requests password values.
        $result = Get-LapsAADPassword -DeviceIds $deviceId -ErrorAction Stop

        foreach ($item in @($result)) {
            $expiry = $item.PasswordExpirationTime
            $backup = $item.BackupTimestamp
            $daysUntilExpiry = $null

            if ($expiry) {
                $daysUntilExpiry = [Math]::Floor(
                    (
                        ([DateTime]$expiry).ToUniversalTime() -
                        $collectionTime
                    ).TotalDays
                )
            }

            $records.Add([pscustomobject]@{
                TimeGenerated          = $collectionTime.ToString('o')
                DeviceId               = [string]$deviceId
                DeviceName             = [string]$item.DeviceName
                PasswordExpirationTime = if ($expiry) {
                    ([DateTime]$expiry).ToUniversalTime().ToString('o')
                } else {
                    $null
                }
                BackupTimestamp        = if ($backup) {
                    ([DateTime]$backup).ToUniversalTime().ToString('o')
                } else {
                    $null
                }
                DaysUntilExpiry        = $daysUntilExpiry
                BackupPresent          = [bool]$backup
                CollectionStatus       = 'Success'
                CollectorVersion       = '1.0.0'
            })
        }
    }
    catch {
        $records.Add([pscustomobject]@{
            TimeGenerated          = $collectionTime.ToString('o')
            DeviceId               = [string]$deviceId
            DeviceName             = $null
            PasswordExpirationTime = $null
            BackupTimestamp        = $null
            DaysUntilExpiry        = $null
            BackupPresent          = $false
            CollectionStatus       = "Error: $($_.Exception.Message)"
            CollectorVersion       = '1.0.0'
        })
    }
}

Send-AzureMonitorRecords `
    -Endpoint $IngestionEndpoint `
    -DcrImmutableId $DcrImmutableId `
    -StreamName $StreamName `
    -Records $records.ToArray()

Write-Output "Submitted $($records.Count) LAPS metadata records."
```

### LAPS permission model

For metadata-only access, grant only the relevant basic LAPS/Graph permission. Do not grant the more privileged permission that allows password retrieval.

Review the installed LAPS module and current Microsoft Graph documentation during implementation because LAPS Graph capabilities and property exposure can change.

## 2.6 Service-Account Alert Query

```kusto name=ServiceAccountRiskAlert.kql
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
    or ResolutionStatus == "NotResolved"
| project
    TimeGenerated,
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
| order by DaysUntilExpiry asc nulls last
```

---

# 3. ERP Log-Directory File and Folder Size

## 3.1 Design Considerations

A directory-size scan sounds simple but can become expensive when folders contain millions of files.

The collector should:

- Scan only explicitly approved directories.
- Avoid following reparse points/junctions.
- Record inaccessible files/directories.
- Record duration.
- Avoid hashing or reading file contents.
- Avoid collecting filenames unless operationally necessary.
- Use a low collection frequency.
- Apply a timeout or monitoring window for very large trees.
- Exclude archive directories where appropriate.
- Run under an account with read/list permission only.

For very large log estates, consider obtaining folder statistics from:

- The ERP application itself.
- A log-management product.
- Storage/file-system metrics.
- A vendor-supported maintenance API.

## 3.2 Suggested Fields

| Field | Description |
|---|---|
| `Computer` | Server |
| `Application` | ERP/middleware application |
| `FolderPath` | Monitored directory |
| `TotalBytes` / `TotalGB` | Total file size |
| `FileCount` | Number of files |
| `OldestFileUtc` | Oldest file modification timestamp |
| `NewestFileUtc` | Latest file modification timestamp |
| `FilesOlderThanRetention` | Files older than configured retention |
| `AccessErrorCount` | Files/directories that could not be inspected |
| `ScanDurationSeconds` | Collector performance |
| `CollectionStatus` | Success, partial success, or error |

## 3.3 Folder Metrics Collector

```powershell name=Collect-ErpFolderMetrics.ps1
[CmdletBinding()]
param(
    [Parameter(Mandatory)]
    [hashtable[]] $FolderDefinitions,

    [Parameter(Mandatory)]
    [string] $IngestionEndpoint,

    [Parameter(Mandatory)]
    [string] $DcrImmutableId,

    [string] $StreamName = 'Custom-ErpFolderMetrics',

    [ValidateRange(1, 3650)]
    [int] $RetentionDays = 30
)

Set-StrictMode -Version Latest
$ErrorActionPreference = 'Stop'

Import-Module "$PSScriptRoot\AzureMonitorIngestion.psm1" -Force

function Get-FolderMetrics {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)]
        [string] $Application,

        [Parameter(Mandatory)]
        [string] $Path,

        [Parameter(Mandatory)]
        [int] $RetentionDays
    )

    $scanStarted = (Get-Date).ToUniversalTime()
    $retentionCutoff = $scanStarted.AddDays(-$RetentionDays)

    if (-not (Test-Path -LiteralPath $Path -PathType Container)) {
        return [pscustomobject]@{
            TimeGenerated          = $scanStarted.ToString('o')
            Computer               = $env:COMPUTERNAME
            Application            = $Application
            FolderPath             = $Path
            TotalBytes             = 0
            TotalGB                = 0
            FileCount              = 0
            OldestFileUtc          = $null
            NewestFileUtc          = $null
            FilesOlderThanRetention = 0
            RetentionDays          = $RetentionDays
            AccessErrorCount       = 0
            ScanDurationSeconds    = 0
            CollectionStatus       = 'FolderNotFound'
            CollectorVersion       = '1.0.0'
        }
    }

    [Int64]$totalBytes = 0
    [Int64]$fileCount = 0
    [Int64]$oldFileCount = 0
    [Int64]$accessErrors = 0
    [DateTime]$oldestFile = [DateTime]::MaxValue
    [DateTime]$newestFile = [DateTime]::MinValue

    $directories = [System.Collections.Generic.Stack[string]]::new()
    $directories.Push((Resolve-Path -LiteralPath $Path).Path)

    while ($directories.Count -gt 0) {
        $currentDirectory = $directories.Pop()

        try {
            foreach ($file in [IO.Directory]::EnumerateFiles(
                $currentDirectory,
                '*',
                [IO.SearchOption]::TopDirectoryOnly
            )) {
                try {
                    $info = [IO.FileInfo]::new($file)

                    $totalBytes += $info.Length
                    $fileCount++

                    $lastWriteUtc = $info.LastWriteTimeUtc

                    if ($lastWriteUtc -lt $oldestFile) {
                        $oldestFile = $lastWriteUtc
                    }

                    if ($lastWriteUtc -gt $newestFile) {
                        $newestFile = $lastWriteUtc
                    }

                    if ($lastWriteUtc -lt $retentionCutoff) {
                        $oldFileCount++
                    }
                }
                catch {
                    $accessErrors++
                }
            }

            foreach ($subDirectory in [IO.Directory]::EnumerateDirectories(
                $currentDirectory,
                '*',
                [IO.SearchOption]::TopDirectoryOnly
            )) {
                try {
                    $directoryInfo = [IO.DirectoryInfo]::new($subDirectory)

                    if (
                        ($directoryInfo.Attributes -band
                            [IO.FileAttributes]::ReparsePoint) -ne 0
                    ) {
                        continue
                    }

                    $directories.Push($directoryInfo.FullName)
                }
                catch {
                    $accessErrors++
                }
            }
        }
        catch {
            $accessErrors++
        }
    }

    $scanCompleted = (Get-Date).ToUniversalTime()

    [pscustomobject]@{
        TimeGenerated           = $scanStarted.ToString('o')
        Computer                = $env:COMPUTERNAME
        Application             = $Application
        FolderPath              = $Path
        TotalBytes              = $totalBytes
        TotalGB                 = [Math]::Round($totalBytes / 1GB, 3)
        FileCount               = $fileCount
        OldestFileUtc           = if ($fileCount -gt 0) {
            $oldestFile.ToString('o')
        } else {
            $null
        }
        NewestFileUtc           = if ($fileCount -gt 0) {
            $newestFile.ToString('o')
        } else {
            $null
        }
        FilesOlderThanRetention = $oldFileCount
        RetentionDays           = $RetentionDays
        AccessErrorCount        = $accessErrors
        ScanDurationSeconds     = [Math]::Round(
            ($scanCompleted - $scanStarted).TotalSeconds,
            2
        )
        CollectionStatus        = if ($accessErrors -gt 0) {
            'PartialSuccess'
        } else {
            'Success'
        }
        CollectorVersion        = '1.0.0'
    }
}

$records = [System.Collections.Generic.List[object]]::new()

foreach ($definition in $FolderDefinitions) {
    if (
        -not $definition.ContainsKey('Application') -or
        -not $definition.ContainsKey('Path')
    ) {
        throw 'Each folder definition requires Application and Path.'
    }

    $records.Add(
        (Get-FolderMetrics `
            -Application $definition.Application `
            -Path $definition.Path `
            -RetentionDays $RetentionDays)
    )
}

Send-AzureMonitorRecords `
    -Endpoint $IngestionEndpoint `
    -DcrImmutableId $DcrImmutableId `
    -StreamName $StreamName `
    -Records $records.ToArray()

Write-Output "Submitted $($records.Count) folder metric records."
```

Example invocation:

```powershell name=Invoke-ErpFolderMetrics.ps1
$folders = @(
    @{
        Application = 'ERP'
        Path        = 'D:\ERP\Logs'
    },
    @{
        Application = 'ERP-IIS'
        Path        = 'D:\IISLogs'
    },
    @{
        Application = 'Middleware'
        Path        = 'E:\Middleware\MessageArchive'
    }
)

.\Collect-ErpFolderMetrics.ps1 `
    -FolderDefinitions $folders `
    -RetentionDays 30 `
    -IngestionEndpoint 'https://<dce-or-dcr-ingestion-endpoint>' `
    -DcrImmutableId 'dcr-00000000000000000000000000000000' `
    -StreamName 'Custom-ErpFolderMetrics'
```

## 3.4 Folder Growth Alert

A static size alert identifies a large directory. A growth-rate query identifies a runaway application before the disk fills.

```kusto name=ErpFolderGrowthAlert.kql
let Current =
    ErpFolderMetrics_CL
    | where TimeGenerated > ago(2h)
    | summarize arg_max(TimeGenerated, TotalBytes, TotalGB, FileCount, CollectionStatus)
        by Computer, Application, FolderPath;
let Previous =
    ErpFolderMetrics_CL
    | where TimeGenerated between (ago(26h) .. ago(22h))
    | summarize arg_max(TimeGenerated, TotalBytes)
        by Computer, Application, FolderPath;
Current
| join kind=leftouter Previous
    on Computer, Application, FolderPath
| extend GrowthBytes24h = TotalBytes - TotalBytes1
| extend GrowthGB24h = round(todouble(GrowthBytes24h) / 1024 / 1024 / 1024, 2)
| where GrowthGB24h >= 5 or CollectionStatus != "Success"
| project
    Computer,
    Application,
    FolderPath,
    TotalGB,
    GrowthGB24h,
    FileCount,
    CollectionStatus
```

The threshold should be workload-specific. A 5 GB/day increase may be severe for an ERP application volume but normal for a high-volume integration archive.

---

# C. DCR and Table Design

Each input stream schema must match the PowerShell fields. Use explicit Azure Monitor types:

- `datetime`
- `string`
- `long`
- `real`
- `boolean`
- `dynamic`, only where genuinely necessary

Prefer typed columns over putting every record into a single dynamic JSON column.

A representative stream-to-table mapping is:

| Stream | Destination table |
|---|---|
| `Custom-ServerCertificateInventory` | `ServerCertificateInventory_CL` |
| `Custom-ServiceAccountStatus` | `ServiceAccountStatus_CL` |
| `Custom-LapsDeviceStatus` | `LapsDeviceStatus_CL` |
| `Custom-ErpFolderMetrics` | `ErpFolderMetrics_CL` |

Use a DCR transformation such as `source` when the incoming schema already matches the destination. Add transformations only for:

- Renaming fields.
- Converting timestamps.
- Dropping unnecessary fields.
- Redacting sensitive values.
- Calculating normalized fields.

---

# D. Deployment and Testing Plan

## D.1 Pre-production Tests

For each collector, test:

1. Normal successful execution.
2. Missing target store/account/folder.
3. Access denied.
4. Azure endpoint unavailable.
5. Invalid access token.
6. DCR schema mismatch.
7. Payload over configured batch size.
8. Server reboot during collection.
9. Duplicate execution.
10. Collector execution duration.
11. Data arrival latency.
12. Alert generation and recovery.

## D.2 Fault Scenarios

### Certificates

- Add a short-lived test certificate.
- Remove an IIS binding.
- Bind an expired test certificate in an isolated environment.
- Verify 60/30/14/7-day alert tiers.

### Service accounts

- Use a test account approaching expiry.
- Disable or lock a test account.
- Configure a test service using a gMSA.
- Verify an unresolved/local account is reported.
- Confirm no password appears in script output, local logs, Azure records, or ServiceNow.

### ERP directories

- Add a known volume of test files.
- Create an inaccessible subdirectory.
- Add a reparse point and verify it is not traversed.
- Simulate rapid directory growth.
- Confirm scan duration is acceptable.

## D.3 Production Hardening

Before production rollout:

- Sign scripts with an enterprise code-signing certificate.
- Use PowerShell execution policy consistent with organizational security standards.
- Store scripts in source control.
- Implement peer review and versioning.
- Use configuration files for folder paths and thresholds.
- Restrict modification rights to the script directory.
- Configure scheduled tasks to run whether or not a user is logged in.
- Capture task exit code and execution history.
- Forward collector errors to `CollectorHealth_CL`.
- Document ownership, escalation, and rollback.
- Monitor the collector itself for missing data.

---

# E. Key Decisions Needed Before Finalizing Implementation

The integrators should resolve these points during detailed design:

1. Are the servers only Azure Arc-enabled, or are they also Entra joined/hybrid joined?
2. Are LAPS passwords backed up to on-premises Active Directory or Microsoft Entra ID?
3. Which identities run ERP services, IIS app pools, scheduled tasks, middleware, and SQL Agent jobs?
4. Are ERP/IIS logs local, on a shared filesystem, or stored by a vendor log service?
5. How many files and how much data exist under each proposed directory?
6. Is Azure Automation Hybrid Worker approved, or must Windows Task Scheduler be used?
7. Is public Azure Monitor ingestion permitted, or is Azure Monitor Private Link required?
8. Which certificate stores and application bindings are in scope?
9. What are the required retention, privacy, and data-residency controls?

---

## Current Microsoft platform notes

According to official Microsoft documentation, you can audit Windows LAPS password backup and expiration metadata in Microsoft Entra ID using **PowerShell** (with the `Get-LapsAADPassword` cmdlet) or the **Microsoft Graph API** without reading the password itself. 

### How To Audit LAPS Password Metadata

- **PowerShell Cmdlet:** Use `Get-LapsAADPassword` with parameters that **do not** include the password, such as:
  - `Get-LapsAADPassword -DeviceIds <DeviceID>`
  - This returns metadata like `PasswordExpirationTime` and `BackupTimestamp`, but not the actual password.
- **Microsoft Graph API:** Use the `deviceLocalCredentials` collection for the device to return non-sensitive metadata related to password backup and expiration.

### Required Permissions

- **To Read Only Metadata (no password):**  
  The minimum Microsoft Graph permission required is:
  - `DeviceLocalCredential.ReadBasic.All`

- **If Querying by Device Name Instead of Device ID:**  
  An additional permission is required for the extra directory lookup:
  - `Device.Read.All`

- **If querying managed devices via Microsoft Managed Desktop:**  
  Also requires:
  - `DeviceManagementManagedDevices.Read.All`

> **Note:** If you query for the password itself, the much more privileged `DeviceLocalCredential.Read.All` permission is needed, but this is not required for auditing only metadata.

### Reference Commands

**Example PowerShell command to read only metadata:**
```powershell
Connect-MgGraph -TenantId <TenantId> -ClientId <ClientId>
Get-LapsAADPassword -DeviceIds <DeviceID>
```
**This will output (example):**
```
DeviceName DeviceId PasswordExpirationTime BackupTimestamp ...
---------- -------- --------------------- ---------------
<MyPC>     <GUID>   2024-06-20T12:00:00Z  2024-06-10T11:40:00Z
```
**No password values are shown unless explicit password flags are used AND higher privileged permissions are granted**[[1]](https://learn.microsoft.com/en-us/powershell/module/laps/get-lapsaadpassword?view=windowsserver2025-ps)[[2]](https://www.intuneautomation.com/script/get-windows-laps-audit/).

### Overview Table

| Operation                    | Minimum Required Permission           |
|------------------------------|---------------------------------------|
| Read metadata only           | DeviceLocalCredential.ReadBasic.All   |
| Query by device name         | Device.Read.All                       |
| Query password (not needed)  | DeviceLocalCredential.Read.All        |

### Further Reading
Microsoft’s own documentation confirms these modes, required permissions, and that metadata-only queries do not expose passwords[[1]](https://learn.microsoft.com/en-us/powershell/module/laps/get-lapsaadpassword?view=windowsserver2025-ps)[[2]](https://www.intuneautomation.com/script/get-windows-laps-audit/).

**Summary:** To audit LAPS metadata in Microsoft Entra ID (Azure AD) via PowerShell or Graph **without reading the password**, use the `Get-LapsAADPassword` cmdlet or Graph API, and ensure the account/app has the `DeviceLocalCredential.ReadBasic.All` (and if using names instead of IDs, also `Device.Read.All`) permission. No higher (password-retrieving) permission is needed.

---

1. [Get-LapsAADPassword (LAPS) | Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/laps/get-lapsaadpassword?view=windowsserver2025-ps)
2. [Get Windows LAPS Audit - IntuneAutomation](https://www.intuneautomation.com/script/get-windows-laps-audit/)

Here’s a concise summary of the key requirements and reference patterns from official Microsoft documentation for using the Azure Monitor Logs Ingestion API from a custom Windows telemetry collector, specifically addressing PowerShell usage, authentication, DCR/endpoints, payload, and limits:

---

**PowerShell Pattern:**
- Microsoft provides sample scripts to automate setup and sending logs. See [Sample code to send data to Azure Monitor Logs using Logs ingestion API] for templates.
- Common steps include:
  1. Register Microsoft Entra application for API authentication.
  2. Create a Data Collection Endpoint (DCE), Data Collection Rule (DCR), and custom table in Log Analytics workspace.
  3. Use a PowerShell module (like `AzLogDcrIngestPS`) or direct REST API via PowerShell to send data based on requirements.

See the full script and logic in the setup documentation and the sample code repository for PowerShell templates[[1]](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/set-up-logs-ingestion-api-prerequisites).

---

**Authentication Roles:**
- The client authenticates with Microsoft Entra ID (Azure AD) using a registered application (client) and a secret (client credential flow).
- App registration/service principal must be granted sufficient roles:
  - **Contributor** on Log Analytics workspace (or scoped at resource group as appropriate) – to manage tables.
  - **Contributor** on the resource group of the DCR – to create/manage DCRs.
  - **Monitoring Metrics Publisher** role on DCR resource group – to send data.
  - **Contributor** on the DCE resource group – to manage endpoints[[1]](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/set-up-logs-ingestion-api-prerequisites).

---

**DCR Endpoint Format:**
- The API endpoint is determined by the DCR or DCE settings (data collection rule/endpoint). Two common patterns:
  - Without DCE:  
    ```
    https://<region>.monitor.azure.com/dataCollectionRules/<DCR ImmutableId>/streams/<streamName>?api-version=2021-11-01-preview
    ```
  - With DCE:  
    ```
    https://<dceName>.<region>.collection.azure.com/dataCollectionRules/<DCR ImmutableId>/streams/<streamName>?api-version=2021-11-01-preview
    ```
- DCR and DCE must be in the same region as the workspace[[2]](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/tutorial-logs-ingestion-api).

---

**Payload Requirements:**
- Data sent must be in JSON, formatted to match the structure defined by the DCR, *not necessarily* the destination table structure, as the DCR can transform and map fields.
- Example:
  ```json
  [
      {
          "Time": "2024-06-19T15:20:03Z",
          "Computer": "Server01",
          "AdditionalContext": {
              "InstanceName": "App1",
              "Level": 2
          }
      }
  ]
  ```
- Send batches in an array (up to the limits; see below)[[3]](https://docs.azure.cn/en-us/azure-monitor/logs/tutorial-logs-ingestion-code).

---

**Limits:**
- The key API limits from documentation:
  - Maximum payload size: **1 MB** (uncompressed JSON) per request.
  - Maximum of **2000 records** per request.
  - Each DCR stream is subject to ingestion throttles; see the [official limits page](https://learn.microsoft.com/azure/azure-monitor/logs/logs-ingestion-api-overview#limits-and-restrictions) for detailed up-to-date values.
  - Maximum **rate**: 5000 requests per minute per DCR stream in a region (as of writing)[[4]](https://docs.azure.cn/en-us/azure-monitor/logs/logs-ingestion-api-overview).

---

**References & Additional Resources:**
- [Logs Ingestion API overview](https://learn.microsoft.com/azure/azure-monitor/logs/logs-ingestion-api-overview)
- [Set up resources required to send data (PowerShell)](https://learn.microsoft.com/azure/azure-monitor/logs/set-up-logs-ingestion-api-prerequisites)
- [Sample code to send data using PowerShell](https://learn.microsoft.com/azure/azure-monitor/logs/tutorial-logs-ingestion-code)

For absolute correctness, always refer to Microsoft's current documentation as limits, endpoints, or required fields may evolve over time.

---

**In Summary:**  
A custom Windows telemetry collector using the Logs Ingestion API must use a registered Microsoft Entra application for auth, have appropriate Azure roles, format payloads as dictated by the DCR, target the correct endpoint, and observe payload size/record limits. PowerShell scripts/templates and modules are provided by Microsoft for rapid setup and sending log data[[5]](https://portal.zero.govt.nz/89437584da775e1bfaae480ae6f37fdd/en-us/azure/azure-monitor/logs/logs-ingestion-api-overview)[[1]](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/set-up-logs-ingestion-api-prerequisites)[[3]](https://docs.azure.cn/en-us/azure-monitor/logs/tutorial-logs-ingestion-code).

---

1. [Set up resources required to send data to Azure Monitor Logs using the ...](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/set-up-logs-ingestion-api-prerequisites)
2. [Tutorial: Send data to Azure Monitor Logs with Logs ingestion API ...](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/tutorial-logs-ingestion-api)
3. [Sample code to send data to Azure Monitor using Logs ingestion API](https://docs.azure.cn/en-us/azure-monitor/logs/tutorial-logs-ingestion-code)
4. [Logs Ingestion API in Azure Monitor](https://docs.azure.cn/en-us/azure-monitor/logs/logs-ingestion-api-overview)
5. [Logs Ingestion API in Azure Monitor - Azure Monitor | Microsoft Learn](https://portal.zero.govt.nz/89437584da775e1bfaae480ae6f37fdd/en-us/azure/azure-monitor/logs/logs-ingestion-api-overview)
