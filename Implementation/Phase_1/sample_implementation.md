# Complete Phase 1 PowerShell Package

Below is a copy-ready package for local deployment on Azure Arc-enabled Windows servers. It implements:

- Azure Arc managed-identity authentication through HIMDS.
- Certificate inventory from `Cert:\LocalMachine\WebHosting`.
- ERP log-folder metrics.
- Azure Monitor Logs Ingestion API submission.
- Retry and local spool/replay.
- Collector health records.
- Execution locking.
- Installation, scheduled-task registration, rollback, and uninstallation.

> **Correction to the prior Phase 1 design:** Azure Arc HIMDS managed-identity token acquisition uses a local challenge-response flow. The script first requests a token with `Metadata: true`, reads the local secret-file path provided through `WWW-Authenticate`, then resubmits using `Authorization: Basic <secret-file-contents>`. The implementation below follows that pattern.  
>
> This package assumes that Phase 2 will provide the actual DCR ingestion endpoint, DCR immutable ID, stream names, and workspace tables.

---

## 1. Package Source Structure

```text
AzureMonitorCollectors/
├── Install-CollectorPackage.ps1
├── Register-CollectorScheduledTasks.ps1
├── Rollback-CollectorPackage.ps1
├── Uninstall-CollectorPackage.ps1
├── Test-CollectorPrerequisites.ps1
│
├── Modules/
│   └── AzureMonitorCollector.Common.psm1
│
├── Collectors/
│   ├── Collect-CertificateInventory.ps1
│   ├── Collect-ErpFolderMetrics.ps1
│   └── Submit-CollectorSpool.ps1
│
├── Config/
│   ├── CollectorSettings.json
│   ├── CertificateInventory.config.json
│   └── ErpFolderMetrics.config.json
│
└── Version/
    └── release.json
```

---

# 2. Shared Runtime Module

This module provides configuration loading, structured logging, lock handling, Arc managed-identity authentication, Logs Ingestion API submission, spool/retry, and collector health records.

```powershell name=Modules/AzureMonitorCollector.Common.psm1
Set-StrictMode -Version Latest

function Get-UtcIso8601 {
    return (Get-Date).ToUniversalTime().ToString('o')
}

function Read-CollectorJsonConfiguration {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)]
        [string] $Path
    )

    if (-not (Test-Path -LiteralPath $Path -PathType Leaf)) {
        throw "Configuration file does not exist: $Path"
    }

    try {
        return Get-Content -LiteralPath $Path -Raw -Encoding UTF8 |
            ConvertFrom-Json -ErrorAction Stop
    }
    catch {
        throw "Invalid JSON configuration at '$Path'. $($_.Exception.Message)"
    }
}

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

    $record = [ordered]@{
        TimeGenerated = Get-UtcIso8601
        Level         = $Level
        ExecutionId   = $ExecutionId
        Computer      = $env:COMPUTERNAME
        Message       = $Message
    }

    $fileName = '{0:yyyy-MM-dd}.log' -f (Get-Date)
    $path = Join-Path $LogDirectory $fileName

    # Do not write secrets, access tokens, private keys, full ingestion payloads,
    # passwords, or raw HTTP headers to the local log.
    ($record | ConvertTo-Json -Compress) |
        Add-Content -LiteralPath $path -Encoding UTF8
}

function Remove-CollectorExpiredFiles {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)]
        [string] $Directory,

        [Parameter(Mandatory)]
        [ValidateRange(1, 3650)]
        [int] $RetentionDays
    )

    if (-not (Test-Path -LiteralPath $Directory)) {
        return
    }

    $cutoff = (Get-Date).ToUniversalTime().AddDays(-$RetentionDays)

    Get-ChildItem -LiteralPath $Directory -File -Force -ErrorAction SilentlyContinue |
        Where-Object { $_.LastWriteTimeUtc -lt $cutoff } |
        Remove-Item -Force -ErrorAction SilentlyContinue
}

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
        return [System.IO.File]::Open(
            $LockPath,
            [System.IO.FileMode]::OpenOrCreate,
            [System.IO.FileAccess]::ReadWrite,
            [System.IO.FileShare]::None
        )
    }
    catch {
        throw "Another instance is already running. Lock: $LockPath"
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

    if ($null -ne $LockHandle) {
        $LockHandle.Dispose()
    }

    Remove-Item -LiteralPath $LockPath -Force -ErrorAction SilentlyContinue
}

function Get-ArcManagedIdentityAccessToken {
    [CmdletBinding()]
    param(
        [string] $Resource = 'https://monitor.azure.com/',

        [string] $ApiVersion = '2020-06-01'
    )

    if ([string]::IsNullOrWhiteSpace($env:IDENTITY_ENDPOINT)) {
        throw (
            'IDENTITY_ENDPOINT is not available. Confirm Azure Arc onboarding and ' +
            'that the system-assigned managed identity is enabled.'
        )
    }

    $endpoint = [uri]$env:IDENTITY_ENDPOINT

    if (-not $endpoint.IsLoopback) {
        throw "IDENTITY_ENDPOINT is not a loopback address: $($endpoint.AbsoluteUri)"
    }

    $resourceEncoded = [uri]::EscapeDataString($Resource)
    $uri = '{0}?resource={1}&api-version={2}' -f `
        $endpoint.AbsoluteUri,
        $resourceEncoded,
        $ApiVersion

    $metadataHeaders = @{
        Metadata = 'True'
    }

    $secretFilePath = $null

    try {
        # Azure Arc HIMDS normally responds with a 401 challenge that provides a
        # local secret file through its WWW-Authenticate header.
        Invoke-WebRequest `
            -Method Get `
            -Uri $uri `
            -Headers $metadataHeaders `
            -UseBasicParsing `
            -TimeoutSec 30 `
            -ErrorAction Stop | Out-Null

        throw (
            'Unexpected HIMDS response: a local challenge was expected but was not returned.'
        )
    }
    catch {
        if ($null -eq $_.Exception.Response) {
            throw "Unable to contact Azure Arc HIMDS. $($_.Exception.Message)"
        }

        $wwwAuthenticate = $_.Exception.Response.Headers['WWW-Authenticate']

        if ([string]::IsNullOrWhiteSpace($wwwAuthenticate)) {
            throw (
                'Azure Arc HIMDS did not return a WWW-Authenticate challenge. ' +
                "HTTP response: $($_.Exception.Response.StatusCode)"
            )
        }

        if ($wwwAuthenticate -notmatch 'Basic\s+realm=(.+)$') {
            throw "Unexpected Azure Arc HIMDS authentication challenge."
        }

        $secretFilePath = $Matches[1].Trim('"')
    }

    if (-not (Test-Path -LiteralPath $secretFilePath -PathType Leaf)) {
        throw "Azure Arc HIMDS secret file is unavailable: $secretFilePath"
    }

    $secret = Get-Content -LiteralPath $secretFilePath -Raw -Encoding UTF8

    try {
        $tokenResponse = Invoke-RestMethod `
            -Method Get `
            -Uri $uri `
            -Headers @{
                Metadata      = 'True'
                Authorization = "Basic $secret"
            } `
            -TimeoutSec 30 `
            -ErrorAction Stop
    }
    catch {
        throw "Azure Arc managed identity token request failed. $($_.Exception.Message)"
    }

    if ([string]::IsNullOrWhiteSpace($tokenResponse.access_token)) {
        throw 'Azure Arc HIMDS did not return an access token.'
    }

    return [string]$tokenResponse.access_token
}

function Get-LogsIngestionUri {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)]
        [string] $IngestionEndpoint,

        [Parameter(Mandatory)]
        [string] $DcrImmutableId,

        [Parameter(Mandatory)]
        [string] $StreamName
    )

    $endpoint = $IngestionEndpoint.TrimEnd('/')
    $dcrId = [uri]::EscapeDataString($DcrImmutableId)
    $stream = [uri]::EscapeDataString($StreamName)

    return (
        '{0}/dataCollectionRules/{1}/streams/{2}?api-version=2023-01-01' -f
        $endpoint,
        $dcrId,
        $stream
    )
}

function Save-CollectorSpoolBatch {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)]
        [string] $SpoolDirectory,

        [Parameter(Mandatory)]
        [string] $CollectorName,

        [Parameter(Mandatory)]
        [string] $IngestionEndpoint,

        [Parameter(Mandatory)]
        [string] $DcrImmutableId,

        [Parameter(Mandatory)]
        [string] $StreamName,

        [Parameter(Mandatory)]
        [string] $PayloadJson
    )

    if (-not (Test-Path -LiteralPath $SpoolDirectory)) {
        New-Item -Path $SpoolDirectory -ItemType Directory -Force | Out-Null
    }

    $spoolObject = [ordered]@{
        SchemaVersion    = '1.0'
        CollectorName    = $CollectorName
        CreatedUtc       = Get-UtcIso8601
        IngestionEndpoint = $IngestionEndpoint
        DcrImmutableId   = $DcrImmutableId
        StreamName       = $StreamName
        Payload          = $PayloadJson
    }

    $fileName = '{0}-{1}-{2}.json' -f `
        $CollectorName,
        (Get-Date -Format 'yyyyMMddHHmmssfff'),
        ([guid]::NewGuid().ToString('N'))

    $path = Join-Path $SpoolDirectory $fileName

    $spoolObject |
        ConvertTo-Json -Depth 10 -Compress |
        Set-Content -LiteralPath $path -Encoding UTF8

    return $path
}

function Send-AzureMonitorRecords {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)]
        [string] $CollectorName,

        [Parameter(Mandatory)]
        [string] $IngestionEndpoint,

        [Parameter(Mandatory)]
        [string] $DcrImmutableId,

        [Parameter(Mandatory)]
        [string] $StreamName,

        [Parameter(Mandatory)]
        [object[]] $Records,

        [Parameter(Mandatory)]
        [pscustomobject] $SharedSettings,

        [string] $SpoolDirectory,

        [string] $LogDirectory,

        [string] $ExecutionId
    )

    if ($Records.Count -eq 0) {
        return 0
    }

    $maxRecords = [int]$SharedSettings.Ingestion.MaximumRecordsPerBatch
    $maxPayloadBytes = [int]$SharedSettings.Ingestion.MaximumPayloadBytes
    $maximumAttempts = [int]$SharedSettings.Ingestion.MaximumAttempts
    $baseDelaySeconds = [int]$SharedSettings.Ingestion.RetryBaseDelaySeconds

    $uri = Get-LogsIngestionUri `
        -IngestionEndpoint $IngestionEndpoint `
        -DcrImmutableId $DcrImmutableId `
        -StreamName $StreamName

    $batches = [System.Collections.Generic.List[object]]::new()
    $batch = [System.Collections.Generic.List[object]]::new()
    $estimatedBytes = 2

    foreach ($record in $Records) {
        $recordJson = $record | ConvertTo-Json -Depth 12 -Compress
        $recordBytes = [Text.Encoding]::UTF8.GetByteCount($recordJson) + 1

        if ($recordBytes -gt $maxPayloadBytes) {
            throw 'A single record is larger than the configured ingestion payload limit.'
        }

        $newBatchRequired =
            $batch.Count -ge $maxRecords -or
            (($estimatedBytes + $recordBytes) -gt $maxPayloadBytes)

        if ($newBatchRequired -and $batch.Count -gt 0) {
            $batches.Add($batch.ToArray())
            $batch = [System.Collections.Generic.List[object]]::new()
            $estimatedBytes = 2
        }

        $batch.Add($record)
        $estimatedBytes += $recordBytes
    }

    if ($batch.Count -gt 0) {
        $batches.Add($batch.ToArray())
    }

    $submitted = 0

    foreach ($currentBatch in $batches) {
        $payload = ConvertTo-Json -InputObject @($currentBatch) -Depth 12 -Compress
        $attempt = 0
        $delivered = $false

        while (-not $delivered -and $attempt -lt $maximumAttempts) {
            $attempt++

            try {
                $token = Get-ArcManagedIdentityAccessToken `
                    -Resource $SharedSettings.ArcManagedIdentity.TokenResource `
                    -ApiVersion $SharedSettings.ArcManagedIdentity.TokenApiVersion

                Invoke-RestMethod `
                    -Method Post `
                    -Uri $uri `
                    -Headers @{ Authorization = "Bearer $token" } `
                    -ContentType 'application/json; charset=utf-8' `
                    -Body ([Text.Encoding]::UTF8.GetBytes($payload)) `
                    -TimeoutSec 60 `
                    -ErrorAction Stop | Out-Null

                $submitted += $currentBatch.Count
                $delivered = $true

                if ($LogDirectory) {
                    Write-CollectorLog `
                        -LogDirectory $LogDirectory `
                        -Level Information `
                        -ExecutionId $ExecutionId `
                        -Message (
                            "Submitted $($currentBatch.Count) records to stream '$StreamName'."
                        )
                }
            }
            catch {
                $isLastAttempt = $attempt -ge $maximumAttempts

                if ($LogDirectory) {
                    Write-CollectorLog `
                        -LogDirectory $LogDirectory `
                        -Level Warning `
                        -ExecutionId $ExecutionId `
                        -Message (
                            "Ingestion attempt $attempt of $maximumAttempts failed for stream " +
                            "'$StreamName'. Error: $($_.Exception.Message)"
                        )
                }

                if ($isLastAttempt) {
                    break
                }

                $delay = [Math]::Min(60, $baseDelaySeconds * [Math]::Pow(2, $attempt - 1))
                Start-Sleep -Seconds $delay
            }
        }

        if (-not $delivered) {
            if ([bool]$SharedSettings.Ingestion.EnableLocalSpool -and $SpoolDirectory) {
                $spoolPath = Save-CollectorSpoolBatch `
                    -SpoolDirectory $SpoolDirectory `
                    -CollectorName $CollectorName `
                    -IngestionEndpoint $IngestionEndpoint `
                    -DcrImmutableId $DcrImmutableId `
                    -StreamName $StreamName `
                    -PayloadJson $payload

                if ($LogDirectory) {
                    Write-CollectorLog `
                        -LogDirectory $LogDirectory `
                        -Level Error `
                        -ExecutionId $ExecutionId `
                        -Message (
                            "Ingestion failed after retries. Batch was spooled to '$spoolPath'."
                        )
                }
            }
            else {
                throw "Ingestion failed after $maximumAttempts attempts and spooling is disabled."
            }
        }
    }

    return $submitted
}

function Submit-CollectorSpoolBatches {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)]
        [string] $SpoolDirectory,

        [Parameter(Mandatory)]
        [pscustomobject] $SharedSettings,

        [string] $LogDirectory,

        [string] $ExecutionId
    )

    if (-not (Test-Path -LiteralPath $SpoolDirectory)) {
        return 0
    }

    $submittedFiles = 0

    foreach ($spoolFile in (Get-ChildItem -LiteralPath $SpoolDirectory -File -Filter '*.json' |
            Sort-Object LastWriteTimeUtc)) {
        try {
            $spool = Get-Content -LiteralPath $spoolFile.FullName -Raw -Encoding UTF8 |
                ConvertFrom-Json -ErrorAction Stop

            $uri = Get-LogsIngestionUri `
                -IngestionEndpoint $spool.IngestionEndpoint `
                -DcrImmutableId $spool.DcrImmutableId `
                -StreamName $spool.StreamName

            $token = Get-ArcManagedIdentityAccessToken `
                -Resource $SharedSettings.ArcManagedIdentity.TokenResource `
                -ApiVersion $SharedSettings.ArcManagedIdentity.TokenApiVersion

            Invoke-RestMethod `
                -Method Post `
                -Uri $uri `
                -Headers @{ Authorization = "Bearer $token" } `
                -ContentType 'application/json; charset=utf-8' `
                -Body ([Text.Encoding]::UTF8.GetBytes([string]$spool.Payload)) `
                -TimeoutSec 60 `
                -ErrorAction Stop | Out-Null

            Remove-Item -LiteralPath $spoolFile.FullName -Force
            $submittedFiles++

            if ($LogDirectory) {
                Write-CollectorLog `
                    -LogDirectory $LogDirectory `
                    -Level Information `
                    -ExecutionId $ExecutionId `
                    -Message "Successfully replayed spool file '$($spoolFile.Name)'."
            }
        }
        catch {
            if ($LogDirectory) {
                Write-CollectorLog `
                    -LogDirectory $LogDirectory `
                    -Level Warning `
                    -ExecutionId $ExecutionId `
                    -Message (
                        "Could not replay spool file '$($spoolFile.Name)'. " +
                        "Error: $($_.Exception.Message)"
                    )
            }
        }
    }

    return $submittedFiles
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
        TimeGenerated    = $completedUtc.ToString('o')
        CollectorName    = $CollectorName
        Computer         = $env:COMPUTERNAME
        TargetScope      = $TargetScope
        ExecutionId      = $ExecutionId
        RecordsCollected = $RecordsCollected
        RecordsSubmitted = $RecordsSubmitted
        DurationSeconds  = [Math]::Round(
            ($completedUtc - $StartedUtc).TotalSeconds,
            3
        )
        Status           = $Status
        ErrorCode        = $ErrorCode
        ErrorMessage     = $ErrorMessage
        CollectorVersion = $CollectorVersion
    }
}

Export-ModuleMember -Function @(
    'Get-UtcIso8601',
    'Read-CollectorJsonConfiguration',
    'Write-CollectorLog',
    'Remove-CollectorExpiredFiles',
    'Enter-CollectorLock',
    'Exit-CollectorLock',
    'Get-ArcManagedIdentityAccessToken',
    'Send-AzureMonitorRecords',
    'Submit-CollectorSpoolBatches',
    'New-CollectorHealthRecord'
)
```

---

# 3. Certificate Inventory Collector

The collector reads only `Cert:\LocalMachine\WebHosting`. It optionally associates certificates with IIS HTTPS bindings. It does not export certificate contents or private keys.

```powershell name=Collectors/Collect-CertificateInventory.ps1
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
    -Path (Join-Path $configRoot 'CertificateInventory.config.json')

if (-not [bool]$collectorSettings.Enabled) {
    Write-Output 'Certificate collector is disabled by configuration.'
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

function Get-CertificateDnsNames {
    param(
        [Parameter(Mandatory)]
        [System.Security.Cryptography.X509Certificates.X509Certificate2] $Certificate
    )

    if ($Certificate.PSObject.Properties.Name -notcontains 'DnsNameList') {
        return $null
    }

    $names = foreach ($dnsName in $Certificate.DnsNameList) {
        if ($dnsName.Unicode) {
            [string]$dnsName.Unicode
        }
    }

    return ($names | Sort-Object -Unique) -join ';'
}

function Get-IisBindingMap {
    $map = @{}

    if (-not [bool]$collectorSettings.EnableIisBindingDiscovery) {
        return $map
    }

    if (-not (Get-Module -ListAvailable -Name WebAdministration)) {
        Write-CollectorLog `
            -LogDirectory $logDirectory `
            -Level Warning `
            -ExecutionId $executionId `
            -Message (
                'IIS binding discovery is enabled but the WebAdministration module is unavailable.'
            )

        return $map
    }

    Import-Module WebAdministration -ErrorAction Stop

    foreach ($site in Get-Website) {
        foreach ($binding in Get-WebBinding -Name $site.Name -Protocol https) {
            $rawHash = $binding.certificateHash

            if ($rawHash -is [byte[]]) {
                $thumbprint = ([BitConverter]::ToString($rawHash)).Replace('-', '')
            }
            else {
                $thumbprint = ([string]$rawHash).Replace(' ', '').ToUpperInvariant()
            }

            if ([string]::IsNullOrWhiteSpace($thumbprint)) {
                continue
            }

            if (-not $map.ContainsKey($thumbprint)) {
                $map[$thumbprint] = [System.Collections.Generic.List[string]]::new()
            }

            $map[$thumbprint].Add((
                '{0}|{1}|{2}' -f
                $site.Name,
                $binding.bindingInformation,
                $binding.certificateStoreName
            ))
        }
    }

    return $map
}

try {
    $lockHandle = Enter-CollectorLock -LockPath $lockPath

    Write-CollectorLog `
        -LogDirectory $logDirectory `
        -Level Information `
        -ExecutionId $executionId `
        -Message 'Certificate inventory collection started.'

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

    $iisBindings = Get-IisBindingMap
    $collectionTime = (Get-Date).ToUniversalTime()
    $records = [System.Collections.Generic.List[object]]::new()

    foreach ($certificateStore in $collectorSettings.CertificateStores) {
        if (-not (Test-Path -LiteralPath $certificateStore)) {
            $status = 'PartialSuccess'

            $records.Add([pscustomobject]@{
                TimeGenerated     = $collectionTime.ToString('o')
                Computer          = $env:COMPUTERNAME
                Environment       = $sharedSettings.Environment
                Application       = $sharedSettings.Application
                StoreLocation     = 'LocalMachine'
                StoreName         = Split-Path -Path $certificateStore -Leaf
                Subject           = $null
                DnsNames          = $null
                Issuer            = $null
                Thumbprint        = $null
                SerialNumber      = $null
                NotBefore         = $null
                NotAfter          = $null
                DaysUntilExpiry   = $null
                HasPrivateKey     = $null
                EnhancedKeyUsage  = $null
                IisBindings       = $null
                CollectionStatus  = 'StoreNotFound'
                CollectorVersion  = '1.0.0'
            })

            continue
        }

        foreach ($certificate in Get-ChildItem -LiteralPath $certificateStore -ErrorAction Stop) {
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
                TimeGenerated     = $collectionTime.ToString('o')
                Computer          = $env:COMPUTERNAME
                Environment       = $sharedSettings.Environment
                Application       = $sharedSettings.Application
                StoreLocation     = 'LocalMachine'
                StoreName         = Split-Path -Path $certificateStore -Leaf
                Subject           = $certificate.Subject
                DnsNames          = Get-CertificateDnsNames -Certificate $certificate
                Issuer            = $certificate.Issuer
                Thumbprint        = $thumbprint
                SerialNumber      = $certificate.SerialNumber
                NotBefore         = $certificate.NotBefore.ToUniversalTime().ToString('o')
                NotAfter          = $certificate.NotAfter.ToUniversalTime().ToString('o')
                DaysUntilExpiry   = [Math]::Floor(
                    ($certificate.NotAfter.ToUniversalTime() - $collectionTime).TotalDays
                )
                HasPrivateKey     = [bool]$certificate.HasPrivateKey
                EnhancedKeyUsage  = $eku
                IisBindings       = $bindingList
                CollectionStatus  = 'Success'
                CollectorVersion  = '1.0.0'
            })
        }
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
            "Certificate inventory completed. Collected=$recordsCollected; " +
            "submitted=$recordsSubmitted."
        )
}
catch {
    $status = 'Failed'
    $errorCode = 'CertificateCollectorFailure'
    $errorMessage = $_.Exception.Message

    Write-CollectorLog `
        -LogDirectory $logDirectory `
        -Level Error `
        -ExecutionId $executionId `
        -Message "Certificate collector failed. Error: $errorMessage"
}
finally {
    $healthRecord = New-CollectorHealthRecord `
        -CollectorName $collectorName `
        -ExecutionId $executionId `
        -StartedUtc $startedUtc `
        -Status $status `
        -RecordsCollected $recordsCollected `
        -RecordsSubmitted $recordsSubmitted `
        -TargetScope (($collectorSettings.CertificateStores) -join ';') `
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
                "Unable to submit collector-health record. Error: $($_.Exception.Message)"
            )
    }

    if ($lockHandle) {
        Exit-CollectorLock -LockHandle $lockHandle -LockPath $lockPath
    }
}

if ($status -eq 'Failed') {
    exit 1
}

exit 0
```

---

# 4. ERP Folder Metrics Collector

This collector performs aggregate folder sizing and retention-age checks. It does not submit individual filenames or file contents.

```powershell name=Collectors/Collect-ErpFolderMetrics.ps1
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
    -Path (Join-Path $configRoot 'ErpFolderMetrics.config.json')

if (-not [bool]$collectorSettings.Enabled) {
    Write-Output 'ERP folder metrics collector is disabled by configuration.'
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

function Get-ErpFolderMetric {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)]
        [pscustomobject] $FolderDefinition,

        [Parameter(Mandatory)]
        [pscustomobject] $SharedSettings,

        [Parameter(Mandatory)]
        [pscustomobject] $CollectorSettings,

        [Parameter(Mandatory)]
        [string] $ExecutionId,

        [Parameter(Mandatory)]
        [string] $LogDirectory
    )

    $scanStartedUtc = (Get-Date).ToUniversalTime()
    $path = [string]$FolderDefinition.Path
    $retentionDays = if ($null -ne $FolderDefinition.RetentionDays) {
        [int]$FolderDefinition.RetentionDays
    }
    else {
        [int]$CollectorSettings.DefaultRetentionDays
    }

    if (-not (Test-Path -LiteralPath $path -PathType Container)) {
        return [pscustomobject]@{
            TimeGenerated           = $scanStartedUtc.ToString('o')
            Computer                = $env:COMPUTERNAME
            Environment             = $SharedSettings.Environment
            Application             = $FolderDefinition.Application
            FolderPath              = $path
            TotalBytes              = [Int64]0
            TotalGB                 = [double]0
            FileCount               = [Int64]0
            OldestFileUtc           = $null
            NewestFileUtc           = $null
            FilesOlderThanRetention = [Int64]0
            RetentionDays           = [Int64]$retentionDays
            AccessErrorCount        = [Int64]0
            ScanDurationSeconds     = [double]0
            CollectionStatus        = 'FolderNotFound'
            CollectorVersion        = '1.0.0'
        }
    }

    $cutoffUtc = $scanStartedUtc.AddDays(-$retentionDays)
    $maximumScanDuration = [TimeSpan]::FromMinutes(
        [int]$CollectorSettings.MaximumScanDurationMinutes
    )

    [Int64]$totalBytes = 0
    [Int64]$fileCount = 0
    [Int64]$oldFileCount = 0
    [Int64]$accessErrors = 0
    [DateTime]$oldestFileUtc = [DateTime]::MaxValue
    [DateTime]$newestFileUtc = [DateTime]::MinValue
    $timedOut = $false

    $directoryStack = [System.Collections.Generic.Stack[string]]::new()
    $directoryStack.Push((Resolve-Path -LiteralPath $path).Path)

    while ($directoryStack.Count -gt 0) {
        if (((Get-Date).ToUniversalTime() - $scanStartedUtc) -gt $maximumScanDuration) {
            $timedOut = $true
            break
        }

        $currentDirectory = $directoryStack.Pop()

        try {
            foreach ($filePath in [System.IO.Directory]::EnumerateFiles(
                $currentDirectory,
                '*',
                [System.IO.SearchOption]::TopDirectoryOnly
            )) {
                try {
                    $file = [System.IO.FileInfo]::new($filePath)

                    $totalBytes += $file.Length
                    $fileCount++

                    if ($file.LastWriteTimeUtc -lt $oldestFileUtc) {
                        $oldestFileUtc = $file.LastWriteTimeUtc
                    }

                    if ($file.LastWriteTimeUtc -gt $newestFileUtc) {
                        $newestFileUtc = $file.LastWriteTimeUtc
                    }

                    if ($file.LastWriteTimeUtc -lt $cutoffUtc) {
                        $oldFileCount++
                    }
                }
                catch {
                    $accessErrors++
                }
            }

            foreach ($subDirectoryPath in [System.IO.Directory]::EnumerateDirectories(
                $currentDirectory,
                '*',
                [System.IO.SearchOption]::TopDirectoryOnly
            )) {
                try {
                    $subDirectory = [System.IO.DirectoryInfo]::new($subDirectoryPath)

                    # Avoid traversing junctions, symbolic links, mount points, and other
                    # reparse points that can create loops or scan unrelated storage.
                    if (
                        ($subDirectory.Attributes -band
                            [System.IO.FileAttributes]::ReparsePoint) -ne 0
                    ) {
                        continue
                    }

                    $directoryStack.Push($subDirectory.FullName)
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

    $scanCompletedUtc = (Get-Date).ToUniversalTime()

    $collectionStatus = if ($timedOut) {
        'ScanTimedOut'
    }
    elseif ($accessErrors -gt 0) {
        'PartialSuccess'
    }
    else {
        'Success'
    }

    if ($timedOut) {
        Write-CollectorLog `
            -LogDirectory $LogDirectory `
            -Level Warning `
            -ExecutionId $ExecutionId `
            -Message (
                "Folder scan timed out after $($CollectorSettings.MaximumScanDurationMinutes) " +
                "minutes: $path"
            )
    }

    return [pscustomobject]@{
        TimeGenerated           = $scanStartedUtc.ToString('o')
        Computer                = $env:COMPUTERNAME
        Environment             = $SharedSettings.Environment
        Application             = $FolderDefinition.Application
        FolderPath              = $path
        TotalBytes              = $totalBytes
        TotalGB                 = [Math]::Round($totalBytes / 1GB, 3)
        FileCount               = $fileCount
        OldestFileUtc           = if ($fileCount -gt 0) {
            $oldestFileUtc.ToString('o')
        } else {
            $null
        }
        NewestFileUtc           = if ($fileCount -gt 0) {
            $newestFileUtc.ToString('o')
        } else {
            $null
        }
        FilesOlderThanRetention = $oldFileCount
        RetentionDays           = [Int64]$retentionDays
        AccessErrorCount        = $accessErrors
        ScanDurationSeconds     = [Math]::Round(
            ($scanCompletedUtc - $scanStartedUtc).TotalSeconds,
            3
        )
        CollectionStatus        = $collectionStatus
        CollectorVersion        = '1.0.0'
    }
}

try {
    $lockHandle = Enter-CollectorLock -LockPath $lockPath

    Write-CollectorLog `
        -LogDirectory $logDirectory `
        -Level Information `
        -ExecutionId $executionId `
        -Message 'ERP folder metrics collection started.'

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

    $records = [System.Collections.Generic.List[object]]::new()

    foreach ($folderDefinition in $collectorSettings.Folders) {
        if (-not [bool]$folderDefinition.Enabled) {
            continue
        }

        $folderRecord = Get-ErpFolderMetric `
            -FolderDefinition $folderDefinition `
            -SharedSettings $sharedSettings `
            -CollectorSettings $collectorSettings `
            -ExecutionId $executionId `
            -LogDirectory $logDirectory

        if ($folderRecord.CollectionStatus -ne 'Success') {
            $status = 'PartialSuccess'
        }

        $records.Add($folderRecord)
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
            "ERP folder metrics completed. Collected=$recordsCollected; " +
            "submitted=$recordsSubmitted."
        )
}
catch {
    $status = 'Failed'
    $errorCode = 'ErpFolderMetricsFailure'
    $errorMessage = $_.Exception.Message

    Write-CollectorLog `
        -LogDirectory $logDirectory `
        -Level Error `
        -ExecutionId $executionId `
        -Message "ERP folder metrics collector failed. Error: $errorMessage"
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
            ($collectorSettings.Folders |
                Where-Object { $_.Enabled } |
                ForEach-Object { $_.Path }) -join ';'
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
                "Unable to submit collector-health record. Error: $($_.Exception.Message)"
            )
    }

    if ($lockHandle) {
        Exit-CollectorLock -LockHandle $lockHandle -LockPath $lockPath
    }
}

if ($status -eq 'Failed') {
    exit 1
}

exit 0
```

---

# 5. Spool Replay Task

This task replays telemetry batches that were spooled because Azure Monitor ingestion was unavailable. It does not perform new certificate or folder collection.

```powershell name=Collectors/Submit-CollectorSpool.ps1
[CmdletBinding()]
param(
    [Parameter(Mandatory)]
    [ValidateSet('CertificateInventory', 'ErpFolderMetrics')]
    [string] $CollectorName
)

Set-StrictMode -Version Latest
$ErrorActionPreference = 'Stop'

$collectorRoot = Split-Path -Path $PSScriptRoot -Parent
$modulePath = Join-Path $collectorRoot 'Modules\AzureMonitorCollector.Common.psm1'
$configRoot = Join-Path $collectorRoot 'Config'

Import-Module $modulePath -Force

$sharedSettings = Read-CollectorJsonConfiguration `
    -Path (Join-Path $configRoot 'CollectorSettings.json')

$executionId = [guid]::NewGuid().ToString()
$logDirectory = Join-Path $sharedSettings.CollectorRoot "Logs\$CollectorName"
$spoolDirectory = Join-Path $sharedSettings.CollectorRoot "Spool\$CollectorName"
$lockPath = Join-Path $sharedSettings.CollectorRoot "Locks\$CollectorName-spool.lock"
$lockHandle = $null

try {
    $lockHandle = Enter-CollectorLock -LockPath $lockPath

    $replayed = Submit-CollectorSpoolBatches `
        -SpoolDirectory $spoolDirectory `
        -SharedSettings $sharedSettings `
        -LogDirectory $logDirectory `
        -ExecutionId $executionId

    Write-CollectorLog `
        -LogDirectory $logDirectory `
        -Level Information `
        -ExecutionId $executionId `
        -Message "Spool replay completed. Files replayed=$replayed."

    exit 0
}
catch {
    Write-CollectorLog `
        -LogDirectory $logDirectory `
        -Level Error `
        -ExecutionId $executionId `
        -Message "Spool replay failed. Error: $($_.Exception.Message)"

    exit 1
}
finally {
    if ($lockHandle) {
        Exit-CollectorLock -LockHandle $lockHandle -LockPath $lockPath
    }
}
```

---

# 6. Configuration Files

The health DCR settings are kept in each collector configuration for simplicity. In a later production revision, these can be centralized in `CollectorSettings.json`.

```json name=Config/CollectorSettings.json
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

```json name=Config/CertificateInventory.config.json
{
  "SchemaVersion": "1.0",
  "CollectorName": "CertificateInventory",
  "Enabled": true,
  "IngestionEndpoint": "https://<certificate-dcr-ingestion-endpoint>",
  "DcrImmutableId": "dcr-<certificate-dcr-immutable-id>",
  "StreamName": "Custom-ServerCertificateInventoryRaw",
  "HealthIngestionEndpoint": "https://<health-dcr-ingestion-endpoint>",
  "HealthDcrImmutableId": "dcr-<health-dcr-immutable-id>",
  "HealthStreamName": "Custom-CollectorHealthRaw",
  "CertificateStores": [
    "Cert:\\LocalMachine\\WebHosting"
  ],
  "EnableIisBindingDiscovery": true,
  "ExpiryWarningDays": 30,
  "ExpiryCriticalDays": 14
}
```

```json name=Config/ErpFolderMetrics.config.json
{
  "SchemaVersion": "1.0",
  "CollectorName": "ErpFolderMetrics",
  "Enabled": true,
  "IngestionEndpoint": "https://<folder-dcr-ingestion-endpoint>",
  "DcrImmutableId": "dcr-<folder-dcr-immutable-id>",
  "StreamName": "Custom-ErpFolderMetricsRaw",
  "HealthIngestionEndpoint": "https://<health-dcr-ingestion-endpoint>",
  "HealthDcrImmutableId": "dcr-<health-dcr-immutable-id>",
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

```json name=Version/release.json
{
  "PackageName": "Contoso.AzureMonitorCollectors",
  "Version": "1.0.0",
  "ReleaseDateUtc": "2026-09-22T00:00:00Z",
  "MinimumPowerShellVersion": "5.1",
  "RequiresAzureArcManagedIdentity": true,
  "Collectors": [
    "CertificateInventory",
    "ErpFolderMetrics"
  ]
}
```

---

# 7. Package Installer

The installer promotes the source package to `Current`, preserves the prior release in `Previous`, creates operational folders, and sets baseline ACLs.

Run the installer as an approved deployment administrator—not as the collector service account.

```powershell name=Install-CollectorPackage.ps1
[CmdletBinding()]
param(
    [Parameter(Mandatory)]
    [string] $SourcePackagePath,

    [string] $CollectorRoot = 'C:\ProgramData\Contoso\AzureMonitorCollectors',

    [Parameter(Mandatory)]
    [string] $CollectorRunAsAccount,

    [switch] $SkipSignatureValidation
)

Set-StrictMode -Version Latest
$ErrorActionPreference = 'Stop'

function Copy-DirectoryContents {
    param(
        [Parameter(Mandatory)]
        [string] $Source,

        [Parameter(Mandatory)]
        [string] $Destination
    )

    New-Item -Path $Destination -ItemType Directory -Force | Out-Null

    Get-ChildItem -LiteralPath $Source -Force |
        Copy-Item -Destination $Destination -Recurse -Force
}

if (-not (Test-Path -LiteralPath $SourcePackagePath -PathType Container)) {
    throw "Source package path was not found: $SourcePackagePath"
}

$requiredItems = @(
    'Modules',
    'Collectors',
    'Config',
    'Version\release.json'
)

foreach ($item in $requiredItems) {
    $path = Join-Path $SourcePackagePath $item

    if (-not (Test-Path -LiteralPath $path)) {
        throw "Source package is missing required item: $item"
    }
}

$stagingPath = Join-Path $CollectorRoot 'Staging'
$currentPath = Join-Path $CollectorRoot 'Current'
$previousPath = Join-Path $CollectorRoot 'Previous'

foreach ($path in @(
    $CollectorRoot,
    (Join-Path $CollectorRoot 'Logs'),
    (Join-Path $CollectorRoot 'Spool'),
    (Join-Path $CollectorRoot 'Locks')
)) {
    New-Item -Path $path -ItemType Directory -Force | Out-Null
}

Remove-Item -LiteralPath $stagingPath -Recurse -Force -ErrorAction SilentlyContinue
Copy-DirectoryContents -Source $SourcePackagePath -Destination $stagingPath

if (-not $SkipSignatureValidation) {
    $signedFiles = Get-ChildItem `
        -LiteralPath $stagingPath `
        -Recurse `
        -File |
        Where-Object { $_.Extension -in @('.ps1', '.psm1') }

    $invalidSignatures = foreach ($file in $signedFiles) {
        $signature = Get-AuthenticodeSignature -FilePath $file.FullName

        if ($signature.Status -ne 'Valid') {
            [pscustomobject]@{
                Path   = $file.FullName
                Status = $signature.Status
                Detail = $signature.StatusMessage
            }
        }
    }

    if ($invalidSignatures) {
        $invalidSignatures | Format-Table -AutoSize | Out-String | Write-Error
        throw 'Package installation stopped because one or more scripts are unsigned or invalid.'
    }
}

Remove-Item -LiteralPath $previousPath -Recurse -Force -ErrorAction SilentlyContinue

if (Test-Path -LiteralPath $currentPath) {
    Move-Item -LiteralPath $currentPath -Destination $previousPath -Force
}

Move-Item -LiteralPath $stagingPath -Destination $currentPath -Force

$aclTargets = @(
    (Join-Path $CollectorRoot 'Current'),
    (Join-Path $CollectorRoot 'Logs'),
    (Join-Path $CollectorRoot 'Spool'),
    (Join-Path $CollectorRoot 'Locks'),
    (Join-Path $CollectorRoot 'Previous')
)

foreach ($target in $aclTargets) {
    $acl = Get-Acl -LiteralPath $target
    $acl.SetAccessRuleProtection($true, $false)

    $administrators = New-Object System.Security.AccessControl.FileSystemAccessRule(
        'BUILTIN\Administrators',
        'FullControl',
        'ContainerInherit,ObjectInherit',
        'None',
        'Allow'
    )

    $system = New-Object System.Security.AccessControl.FileSystemAccessRule(
        'NT AUTHORITY\SYSTEM',
        'FullControl',
        'ContainerInherit,ObjectInherit',
        'None',
        'Allow'
    )

    $acl.SetAccessRule($administrators)
    $acl.SetAccessRule($system)

    Set-Acl -LiteralPath $target -AclObject $acl
}

$currentAcl = Get-Acl -LiteralPath $currentPath
$currentAcl.AddAccessRule(
    (New-Object System.Security.AccessControl.FileSystemAccessRule(
        $CollectorRunAsAccount,
        'ReadAndExecute',
        'ContainerInherit,ObjectInherit',
        'None',
        'Allow'
    ))
)
Set-Acl -LiteralPath $currentPath -AclObject $currentAcl

foreach ($path in @(
    (Join-Path $CollectorRoot 'Logs'),
    (Join-Path $CollectorRoot 'Spool'),
    (Join-Path $CollectorRoot 'Locks')
)) {
    $acl = Get-Acl -LiteralPath $path
    $acl.AddAccessRule(
        (New-Object System.Security.AccessControl.FileSystemAccessRule(
            $CollectorRunAsAccount,
            'Modify',
            'ContainerInherit,ObjectInherit',
            'None',
            'Allow'
        ))
    )
    Set-Acl -LiteralPath $path -AclObject $acl
}

Write-Output "Collector package installed successfully at: $currentPath"
Write-Output "Previous package, if present: $previousPath"
```

---

# 8. Scheduled Task Registration

The task account must already exist and must have:

- **Log on as a batch job**
- Membership in **Hybrid Agent Extension Applications**
- Read/execute access to the collector package
- Write access to Logs, Spool, and Locks
- Required access to `Cert:\LocalMachine\WebHosting` and ERP log folders

```powershell name=Register-CollectorScheduledTasks.ps1
[CmdletBinding()]
param(
    [Parameter(Mandatory)]
    [string] $CollectorRunAsAccount,

    [Parameter(Mandatory)]
    [SecureString] $CollectorRunAsPassword,

    [string] $CollectorRoot = 'C:\ProgramData\Contoso\AzureMonitorCollectors',

    [ValidateRange(5, 1440)]
    [int] $FolderMetricsIntervalMinutes = 60
)

Set-StrictMode -Version Latest
$ErrorActionPreference = 'Stop'

$currentPath = Join-Path $CollectorRoot 'Current'
$collectorPath = Join-Path $currentPath 'Collectors'
$powerShell = 'C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe'

if (-not (Test-Path -LiteralPath $collectorPath)) {
    throw "Collector package was not found at: $collectorPath"
}

function Register-CollectorTask {
    param(
        [Parameter(Mandatory)]
        [string] $TaskName,

        [Parameter(Mandatory)]
        [string] $ScriptPath,

        [Parameter(Mandatory)]
        [Microsoft.Management.Infrastructure.CimInstance[]] $Triggers,

        [Parameter(Mandatory)]
        [string] $Description
    )

    $action = New-ScheduledTaskAction `
        -Execute $powerShell `
        -Argument (
            '-NoLogo -NoProfile -NonInteractive -ExecutionPolicy AllSigned ' +
            "-File `"$ScriptPath`""
        ) `
        -WorkingDirectory $currentPath

    $settings = New-ScheduledTaskSettingsSet `
        -ExecutionTimeLimit (New-TimeSpan -Minutes 30) `
        -MultipleInstances IgnoreNew `
        -RestartCount 3 `
        -RestartInterval (New-TimeSpan -Minutes 15) `
        -StartWhenAvailable

    Register-ScheduledTask `
        -TaskName $TaskName `
        -Action $action `
        -Trigger $Triggers `
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
        -Description $Description `
        -Force | Out-Null
}

$certificateScript = Join-Path $collectorPath 'Collect-CertificateInventory.ps1'
$folderScript = Join-Path $collectorPath 'Collect-ErpFolderMetrics.ps1'
$spoolScript = Join-Path $collectorPath 'Submit-CollectorSpool.ps1'

$certificateTrigger = New-ScheduledTaskTrigger -Daily -At 2:15AM

$folderTrigger = New-ScheduledTaskTrigger -Once -At (Get-Date).Date.AddMinutes(30)
$folderTrigger.Repetition.Interval = (New-TimeSpan -Minutes $FolderMetricsIntervalMinutes)
$folderTrigger.Repetition.Duration = (New-TimeSpan -Days 1)

$certificateSpoolAction = New-ScheduledTaskAction `
    -Execute $powerShell `
    -Argument (
        '-NoLogo -NoProfile -NonInteractive -ExecutionPolicy AllSigned ' +
        "-File `"$spoolScript`" -CollectorName CertificateInventory"
    ) `
    -WorkingDirectory $currentPath

$folderSpoolAction = New-ScheduledTaskAction `
    -Execute $powerShell `
    -Argument (
        '-NoLogo -NoProfile -NonInteractive -ExecutionPolicy AllSigned ' +
        "-File `"$spoolScript`" -CollectorName ErpFolderMetrics"
    ) `
    -WorkingDirectory $currentPath

$spoolTrigger = New-ScheduledTaskTrigger -Once -At (Get-Date).Date.AddMinutes(10)
$spoolTrigger.Repetition.Interval = (New-TimeSpan -Minutes 15)
$spoolTrigger.Repetition.Duration = (New-TimeSpan -Days 1)

$settings = New-ScheduledTaskSettingsSet `
    -ExecutionTimeLimit (New-TimeSpan -Minutes 30) `
    -MultipleInstances IgnoreNew `
    -RestartCount 2 `
    -RestartInterval (New-TimeSpan -Minutes 15) `
    -StartWhenAvailable

Register-CollectorTask `
    -TaskName 'Contoso-Monitor-CertificateInventory' `
    -ScriptPath $certificateScript `
    -Triggers @($certificateTrigger) `
    -Description 'Collects WebHosting certificate inventory for Azure Monitor.'

Register-CollectorTask `
    -TaskName 'Contoso-Monitor-ErpFolderMetrics' `
    -ScriptPath $folderScript `
    -Triggers @($folderTrigger) `
    -Description 'Collects ERP log-directory metrics for Azure Monitor.'

Register-ScheduledTask `
    -TaskName 'Contoso-Monitor-CertificateInventory-SpoolReplay' `
    -Action $certificateSpoolAction `
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
    -Description 'Replays spooled certificate telemetry to Azure Monitor.' `
    -Force | Out-Null

Register-ScheduledTask `
    -TaskName 'Contoso-Monitor-ErpFolderMetrics-SpoolReplay' `
    -Action $folderSpoolAction `
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
    -Description 'Replays spooled ERP folder telemetry to Azure Monitor.' `
    -Force | Out-Null

Write-Output 'Collector scheduled tasks were registered successfully.'
```

> For production, replace repeated password conversion with an enterprise deployment mechanism that handles the account credential securely. Do not store the password in the task-registration script, source control, local configuration, or a command-line history.

---

# 9. Rollback Script

The rollback returns the previous signed release to `Current`.

```powershell name=Rollback-CollectorPackage.ps1
[CmdletBinding()]
param(
    [string] $CollectorRoot = 'C:\ProgramData\Contoso\AzureMonitorCollectors'
)

Set-StrictMode -Version Latest
$ErrorActionPreference = 'Stop'

$currentPath = Join-Path $CollectorRoot 'Current'
$previousPath = Join-Path $CollectorRoot 'Previous'
$rollbackPath = Join-Path $CollectorRoot 'Rollback-Staging'

if (-not (Test-Path -LiteralPath $previousPath -PathType Container)) {
    throw 'No previous collector package is available for rollback.'
}

Get-ScheduledTask -TaskName 'Contoso-Monitor-*' -ErrorAction SilentlyContinue |
    Stop-ScheduledTask -ErrorAction SilentlyContinue

Start-Sleep -Seconds 5

Remove-Item -LiteralPath $rollbackPath -Recurse -Force -ErrorAction SilentlyContinue

if (Test-Path -LiteralPath $currentPath) {
    Move-Item -LiteralPath $currentPath -Destination $rollbackPath -Force
}

Move-Item -LiteralPath $previousPath -Destination $currentPath -Force

if (Test-Path -LiteralPath $rollbackPath) {
    Move-Item -LiteralPath $rollbackPath -Destination $previousPath -Force
}

Write-Output 'Collector package rollback completed successfully.'
Write-Output "Active release: $currentPath"
Write-Output "Previous release: $previousPath"
```

---

# 10. Uninstall Script

```powershell name=Uninstall-CollectorPackage.ps1
[CmdletBinding(SupportsShouldProcess)]
param(
    [string] $CollectorRoot = 'C:\ProgramData\Contoso\AzureMonitorCollectors',

    [switch] $RemoveOperationalLogsAndSpool
)

Set-StrictMode -Version Latest
$ErrorActionPreference = 'Stop'

$taskNames = @(
    'Contoso-Monitor-CertificateInventory',
    'Contoso-Monitor-ErpFolderMetrics',
    'Contoso-Monitor-CertificateInventory-SpoolReplay',
    'Contoso-Monitor-ErpFolderMetrics-SpoolReplay'
)

foreach ($taskName in $taskNames) {
    if (Get-ScheduledTask -TaskName $taskName -ErrorAction SilentlyContinue) {
        if ($PSCmdlet.ShouldProcess($taskName, 'Unregister scheduled task')) {
            Unregister-ScheduledTask -TaskName $taskName -Confirm:$false
        }
    }
}

foreach ($directory in @(
    (Join-Path $CollectorRoot 'Current'),
    (Join-Path $CollectorRoot 'Previous'),
    (Join-Path $CollectorRoot 'Staging'),
    (Join-Path $CollectorRoot 'Locks')
)) {
    if (Test-Path -LiteralPath $directory) {
        if ($PSCmdlet.ShouldProcess($directory, 'Remove collector package content')) {
            Remove-Item -LiteralPath $directory -Recurse -Force
        }
    }
}

if ($RemoveOperationalLogsAndSpool) {
    foreach ($directory in @(
        (Join-Path $CollectorRoot 'Logs'),
        (Join-Path $CollectorRoot 'Spool')
    )) {
        if (Test-Path -LiteralPath $directory) {
            if ($PSCmdlet.ShouldProcess($directory, 'Remove operational logs and spool files')) {
                Remove-Item -LiteralPath $directory -Recurse -Force
            }
        }
    }
}

Write-Output 'Collector package uninstallation completed.'
```

---

# 11. Prerequisite Validation Script

Run this using the **same local account** that will execute the scheduled tasks.

```powershell name=Test-CollectorPrerequisites.ps1
[CmdletBinding()]
param(
    [string] $CollectorRoot = 'C:\ProgramData\Contoso\AzureMonitorCollectors',

    [Parameter(Mandatory)]
    [string] $ErpLogPath,

    [switch] $RequireIisBindingDiscovery
)

Set-StrictMode -Version Latest
$ErrorActionPreference = 'Continue'

$failures = [System.Collections.Generic.List[string]]::new()
$warnings = [System.Collections.Generic.List[string]]::new()

if ($PSVersionTable.PSVersion -lt [Version]'5.1') {
    $failures.Add("PowerShell 5.1 or newer is required. Found: $($PSVersionTable.PSVersion)")
}

$himds = Get-Service -Name 'himds' -ErrorAction SilentlyContinue

if ($null -eq $himds) {
    $failures.Add('Azure Arc HIMDS service is not installed or cannot be found.')
}
elseif ($himds.Status -ne 'Running') {
    $failures.Add("Azure Arc HIMDS service is not running. Current state: $($himds.Status)")
}

if ([string]::IsNullOrWhiteSpace($env:IDENTITY_ENDPOINT)) {
    $warnings.Add(
        'IDENTITY_ENDPOINT is not visible in this shell. Confirm it is visible to the ' +
        'scheduled-task execution context and that Azure Arc managed identity is enabled.'
    )
}

if (-not (Test-Path -LiteralPath 'Cert:\LocalMachine\WebHosting')) {
    $failures.Add('Certificate store Cert:\LocalMachine\WebHosting is unavailable.')
}

try {
    Get-ChildItem -LiteralPath 'Cert:\LocalMachine\WebHosting' -ErrorAction Stop |
        Select-Object -First 1 | Out-Null
}
catch {
    $failures.Add(
        "The current account cannot enumerate Cert:\LocalMachine\WebHosting. $($_.Exception.Message)"
    )
}

if (-not (Test-Path -LiteralPath $ErpLogPath -PathType Container)) {
    $failures.Add("ERP log directory does not exist: $ErpLogPath")
}
else {
    try {
        Get-ChildItem -LiteralPath $ErpLogPath -Force -ErrorAction Stop |
            Select-Object -First 1 | Out-Null
    }
    catch {
        $failures.Add(
            "The current account cannot read ERP log directory '$ErpLogPath'. " +
            "$($_.Exception.Message)"
        )
    }
}

if ($RequireIisBindingDiscovery -and
    -not (Get-Module -ListAvailable -Name WebAdministration)) {
    $warnings.Add(
        'WebAdministration module is unavailable. IIS binding discovery will be skipped.'
    )
}

try {
    New-Item -Path $CollectorRoot -ItemType Directory -Force -ErrorAction Stop | Out-Null
}
catch {
    $failures.Add("Cannot create/access collector root '$CollectorRoot'. $($_.Exception.Message)")
}

Write-Output '--- Collector Prerequisite Validation ---'

foreach ($warning in $warnings) {
    Write-Warning $warning
}

foreach ($failure in $failures) {
    Write-Error $failure
}

if ($failures.Count -gt 0) {
    Write-Output "Validation failed. Failures=$($failures.Count); Warnings=$($warnings.Count)"
    exit 1
}

Write-Output "Validation passed. Warnings=$($warnings.Count)"
exit 0
```

---

# 12. Deployment Sequence

## Per Collector Server

1. Ensure Azure Arc onboarding is complete.
2. Enable the Azure Arc system-assigned managed identity.
3. Create the dedicated local collector account.
4. Add the account to `Hybrid Agent Extension Applications`, subject to approval.
5. Grant **Log on as a batch job**.
6. Grant minimum read permissions:
   - `Cert:\LocalMachine\WebHosting`
   - Approved ERP log folders
   - IIS configuration only when binding discovery is enabled
7. Configure outbound HTTPS and proxy requirements.
8. Deploy the signed package with `Install-CollectorPackage.ps1`.
9. Replace all Phase 2 placeholders in JSON:
   - `IngestionEndpoint`
   - `DcrImmutableId`
   - `StreamName`
   - Health DCR fields
10. Run `Test-CollectorPrerequisites.ps1` as the collector account.
11. Register tasks using `Register-CollectorScheduledTasks.ps1`.
12. Trigger each task manually:
   ```powershell name=Invoke-CollectorTasks.ps1
   Start-ScheduledTask -TaskName 'Contoso-Monitor-CertificateInventory'
   Start-ScheduledTask -TaskName 'Contoso-Monitor-ErpFolderMetrics'
   ```
13. Review:
   - Task Scheduler Operational log
   - Local collector JSON logs
   - Spool folder
   - In Phase 2, Log Analytics custom tables

---

# 13. Required Phase 2 Changes

Before collectors can successfully send data, Phase 2 must provide:

| Config field | Source |
|---|---|
| `IngestionEndpoint` | DCR ingestion endpoint or DCE endpoint |
| `DcrImmutableId` | Created DCR immutable ID |
| `StreamName` | DCR stream declaration |
| `HealthIngestionEndpoint` | Collector health DCR/DCE endpoint |
| `HealthDcrImmutableId` | Collector health DCR immutable ID |
| `HealthStreamName` | Collector health stream declaration |
| Azure RBAC | Arc machine identity receives minimum DCR ingestion authorization |

---

# 14. Important Production Hardening Items

Before production rollout, add:

- Pester unit tests for certificate, folder traversal, locking, retry, and payload functions.
- PSScriptAnalyzer in CI.
- Script signing in CI/CD.
- Package checksum validation.
- Central deployment mechanism.
- Spool-size monitoring.
- Local disk-capacity check for collector logs/spool.
- Collector-version tracking in every record.
- Alerting for no collector-health record within expected time.
- Explicit proxy support tests under the scheduled-task identity.
- Bicep deployment for all Azure resources.
- Change control for configuration files and DCR schema evolution.

### Want me to…
<!-- copilot:suggestions -->
1. Continue to Phase 2 with copy-ready Bicep for custom tables, DCR stream schemas, Azure Arc identity RBAC, and direct-ingestion validation.
2. Add Pester tests and a CI pipeline design for this package.
3. Hand this package off to the coding agent to open a pull request in a repository you provide.
