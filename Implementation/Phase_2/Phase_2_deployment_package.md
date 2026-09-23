Below is a companion deployment script that packages the Phase 2 Bicep source, validates it, runs `what-if`, deploys it, captures outputs, and generates collector configuration templates using the deployed DCE endpoint and DCR immutable IDs.

It is intended to run from an administrator workstation or CI/CD runner with:

- Azure CLI installed and authenticated.
- Bicep available through Azure CLI.
- Permission to create the resource group, workspace, DCE, DCRs, tables, Action Group, and RBAC assignments.
- Permission to read the Azure Arc machine managed-identity principal IDs.

It does **not** deploy the Windows collector package to servers; use the Phase 1 installer after this script generates the Azure-side configuration values.

```powershell name=azure/scripts/Deploy-Phase2Monitoring.ps1
<#
.SYNOPSIS
    Packages, validates, performs what-if, and deploys the Phase 2 Azure Monitor
    Bicep infrastructure for the ERP hybrid monitoring pilot.

.DESCRIPTION
    This script:
      1. Validates Azure CLI and Bicep availability.
      2. Authenticates/selects the target Azure subscription.
      3. Creates the pilot resource group if it does not exist.
      4. Retrieves system-assigned managed identity principal IDs from Azure Arc servers.
      5. Produces a deployment package ZIP containing the Bicep source and parameter file.
      6. Runs Bicep build and Azure deployment validation.
      7. Runs Azure Resource Manager what-if.
      8. Deploys the Log Analytics Workspace, tables, DCE, DCRs, RBAC assignments,
         and Action Group.
      9. Saves deployment outputs as JSON.
     10. Generates Phase 1 collector configuration templates populated with the
         DCE endpoint and DCR immutable IDs.

.NOTES
    Run from an approved administration workstation or CI/CD runner.
    Do not run this from an ERP production server.

    The script intentionally does not store credentials, workspace shared keys,
    client secrets, or certificate private keys.

    Requires:
      - Azure CLI
      - Azure CLI Bicep integration
      - Azure RBAC permissions appropriate to deploy Azure Monitor resources
      - Access to inspect Azure Arc connected machines
#>

[CmdletBinding(SupportsShouldProcess)]
param(
    [Parameter(Mandatory)]
    [ValidateNotNullOrEmpty()]
    [string] $SubscriptionId,

    [Parameter(Mandatory)]
    [ValidateNotNullOrEmpty()]
    [string] $ResourceGroupName,

    [Parameter(Mandatory)]
    [ValidateNotNullOrEmpty()]
    [string] $Location,

    [Parameter(Mandatory)]
    [ValidateScript({ Test-Path -LiteralPath $_ -PathType Leaf })]
    [string] $TemplateFile,

    [Parameter(Mandatory)]
    [ValidateScript({ Test-Path -LiteralPath $_ -PathType Leaf })]
    [string] $ParameterFile,

    [Parameter(Mandatory)]
    [ValidateNotNullOrEmpty()]
    [string] $ArcResourceGroupName,

    [Parameter(Mandatory)]
    [ValidateNotNullOrEmpty()]
    [string[]] $CertificateCollectorArcMachineNames,

    [Parameter(Mandatory)]
    [ValidateNotNullOrEmpty()]
    [string[]] $FolderCollectorArcMachineNames,

    [Parameter()]
    [ValidateNotNullOrEmpty()]
    [string[]] $HealthCollectorArcMachineNames,

    [Parameter(Mandatory)]
    [ValidateNotNullOrEmpty()]
    [string] $ArtifactDirectory,

    [switch] $CreateResourceGroup,

    [switch] $SkipWhatIf,

    [switch] $SkipPackaging,

    [switch] $SkipDeployment
)

Set-StrictMode -Version Latest
$ErrorActionPreference = 'Stop'

$script:Timestamp = Get-Date -Format 'yyyyMMdd-HHmmss'
$script:DeploymentName = "erpmon-phase2-$script:Timestamp"

function Write-DeploymentLog {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)]
        [ValidateSet('Information', 'Warning', 'Error')]
        [string] $Level,

        [Parameter(Mandatory)]
        [string] $Message
    )

    $timestamp = (Get-Date).ToUniversalTime().ToString('o')
    $prefix = "[$timestamp] [$Level]"

    switch ($Level) {
        'Information' { Write-Host "$prefix $Message" -ForegroundColor Cyan }
        'Warning'     { Write-Warning "$prefix $Message" }
        'Error'       { Write-Error "$prefix $Message" }
    }
}

function Invoke-AzureCli {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)]
        [string[]] $Arguments,

        [switch] $AllowFailure
    )

    $result = & az @Arguments 2>&1
    $exitCode = $LASTEXITCODE

    if ($exitCode -ne 0 -and -not $AllowFailure) {
        throw (
            "Azure CLI command failed with exit code $exitCode.`n" +
            "Command: az $($Arguments -join ' ')`n" +
            "Output:`n$($result -join [Environment]::NewLine)"
        )
    }

    return $result
}

function Test-RequiredTools {
    Write-DeploymentLog -Level Information -Message 'Validating Azure CLI and Bicep availability.'

    $azCommand = Get-Command az -ErrorAction SilentlyContinue

    if ($null -eq $azCommand) {
        throw (
            'Azure CLI is not installed or is unavailable on PATH. ' +
            'Install Azure CLI before running this deployment.'
        )
    }

    $azVersion = Invoke-AzureCli -Arguments @('version', '--output', 'json') |
        ConvertFrom-Json -ErrorAction Stop

    Write-DeploymentLog -Level Information -Message (
        "Azure CLI version detected: $($azVersion.'azure-cli')"
    )

    $bicepVersionOutput = Invoke-AzureCli `
        -Arguments @('bicep', 'version') `
        -AllowFailure

    if ($LASTEXITCODE -ne 0 -or
        ($bicepVersionOutput -join ' ') -match 'not found|not installed') {
        Write-DeploymentLog -Level Information -Message 'Installing Azure CLI Bicep component.'

        Invoke-AzureCli -Arguments @('bicep', 'install') | Out-Null
    }

    $bicepVersion = Invoke-AzureCli -Arguments @('bicep', 'version')
    Write-DeploymentLog -Level Information -Message (
        "Bicep version detected: $($bicepVersion -join ' ')"
    )
}

function Set-AzureSubscription {
    Write-DeploymentLog -Level Information -Message (
        "Selecting Azure subscription: $SubscriptionId"
    )

    $account = Invoke-AzureCli `
        -Arguments @('account', 'show', '--output', 'json') `
        -AllowFailure

    if ($LASTEXITCODE -ne 0) {
        Write-DeploymentLog -Level Information -Message (
            'No active Azure CLI session was found. Opening Azure login.'
        )

        Invoke-AzureCli -Arguments @('login', '--output', 'none') | Out-Null
    }

    Invoke-AzureCli `
        -Arguments @('account', 'set', '--subscription', $SubscriptionId) |
        Out-Null

    $selectedAccount = Invoke-AzureCli `
        -Arguments @('account', 'show', '--output', 'json') |
        ConvertFrom-Json -ErrorAction Stop

    if ($selectedAccount.id -ne $SubscriptionId) {
        throw (
            "Azure CLI selected subscription '$($selectedAccount.id)' does not match " +
            "requested subscription '$SubscriptionId'."
        )
    }

    Write-DeploymentLog -Level Information -Message (
        "Authenticated as '$($selectedAccount.user.name)' in subscription '$($selectedAccount.name)'."
    )
}

function Ensure-ResourceGroup {
    [CmdletBinding()]
    param()

    $existing = Invoke-AzureCli -Arguments @(
        'group',
        'exists',
        '--name',
        $ResourceGroupName
    )

    $resourceGroupExists = [System.Convert]::ToBoolean(($existing -join '').Trim())

    if ($resourceGroupExists) {
        Write-DeploymentLog -Level Information -Message (
            "Resource group exists: $ResourceGroupName"
        )

        return
    }

    if (-not $CreateResourceGroup) {
        throw (
            "Resource group '$ResourceGroupName' does not exist. " +
            'Re-run with -CreateResourceGroup after confirming the approved Azure region.'
        )
    }

    if ($PSCmdlet.ShouldProcess(
        $ResourceGroupName,
        "Create resource group in region '$Location'"
    )) {
        Write-DeploymentLog -Level Information -Message (
            "Creating resource group '$ResourceGroupName' in '$Location'."
        )

        Invoke-AzureCli -Arguments @(
            'group',
            'create',
            '--name',
            $ResourceGroupName,
            '--location',
            $Location,
            '--tags',
            'Application=ERP-Monitoring',
            'Environment=Pilot',
            'ManagedBy=Bicep',
            'DataClassification=Internal'
        ) | Out-Null
    }
}

function Get-ArcMachinePrincipalId {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)]
        [string] $MachineName
    )

    Write-DeploymentLog -Level Information -Message (
        "Retrieving system-assigned managed identity principal ID for Arc machine '$MachineName'."
    )

    $machineJson = Invoke-AzureCli -Arguments @(
        'connectedmachine',
        'show',
        '--resource-group',
        $ArcResourceGroupName,
        '--name',
        $MachineName,
        '--output',
        'json'
    )

    $machine = $machineJson | ConvertFrom-Json -ErrorAction Stop
    $principalId = [string]$machine.identity.principalId

    if ([string]::IsNullOrWhiteSpace($principalId)) {
        throw (
            "Azure Arc machine '$MachineName' does not have a system-assigned managed " +
            'identity principal ID. Enable system-assigned managed identity before deployment.'
        )
    }

    return $principalId
}

function Get-UniqueArcPrincipalIds {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)]
        [string[]] $MachineNames
    )

    $principalIds = foreach ($machineName in $MachineNames) {
        Get-ArcMachinePrincipalId -MachineName $machineName
    }

    return @($principalIds | Sort-Object -Unique)
}

function New-DeploymentArtifact {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)]
        [string[]] $CertificatePrincipalIds,

        [Parameter(Mandatory)]
        [string[]] $FolderPrincipalIds,

        [Parameter(Mandatory)]
        [string[]] $HealthPrincipalIds
    )

    $artifactRoot = Join-Path $ArtifactDirectory "phase2-$script:Timestamp"
    $sourceRoot = Split-Path -Path $TemplateFile -Parent
    $stagingRoot = Join-Path $artifactRoot 'source'
    $zipPath = Join-Path $artifactRoot "erpmon-phase2-$script:Timestamp.zip"

    New-Item -Path $artifactRoot -ItemType Directory -Force | Out-Null

    $metadata = [ordered]@{
        DeploymentName               = $script:DeploymentName
        CreatedUtc                   = (Get-Date).ToUniversalTime().ToString('o')
        SubscriptionId               = $SubscriptionId
        ResourceGroupName            = $ResourceGroupName
        Location                     = $Location
        TemplateFile                 = (Resolve-Path -LiteralPath $TemplateFile).Path
        ParameterFile                = (Resolve-Path -LiteralPath $ParameterFile).Path
        ArcResourceGroupName         = $ArcResourceGroupName
        CertificateCollectorMachines = $CertificateCollectorArcMachineNames
        FolderCollectorMachines      = $FolderCollectorArcMachineNames
        HealthCollectorMachines      = $HealthCollectorArcMachineNames
        CertificatePrincipalIds      = $CertificatePrincipalIds
        FolderPrincipalIds           = $FolderPrincipalIds
        HealthPrincipalIds           = $HealthPrincipalIds
    }

    $metadataPath = Join-Path $artifactRoot 'deployment-metadata.json'
    $metadata | ConvertTo-Json -Depth 8 |
        Set-Content -LiteralPath $metadataPath -Encoding UTF8

    if ($SkipPackaging) {
        Write-DeploymentLog -Level Information -Message (
            'Source packaging was skipped by request.'
        )

        return [pscustomobject]@{
            ArtifactRoot = $artifactRoot
            MetadataPath = $metadataPath
            ZipPath      = $null
        }
    }

    New-Item -Path $stagingRoot -ItemType Directory -Force | Out-Null

    Copy-Item `
        -LiteralPath $sourceRoot `
        -Destination (Join-Path $stagingRoot 'azure') `
        -Recurse `
        -Force

    Copy-Item `
        -LiteralPath $ParameterFile `
        -Destination (Join-Path $stagingRoot 'deployment-parameters.bicepparam') `
        -Force

    Copy-Item `
        -LiteralPath $metadataPath `
        -Destination (Join-Path $stagingRoot 'deployment-metadata.json') `
        -Force

    Compress-Archive `
        -Path (Join-Path $stagingRoot '*') `
        -DestinationPath $zipPath `
        -CompressionLevel Optimal `
        -Force

    $hash = Get-FileHash -LiteralPath $zipPath -Algorithm SHA256
    $hashPath = "$zipPath.sha256"

    "$($hash.Hash)  $(Split-Path -Path $zipPath -Leaf)" |
        Set-Content -LiteralPath $hashPath -Encoding ASCII

    Write-DeploymentLog -Level Information -Message (
        "Deployment package created: $zipPath"
    )

    return [pscustomobject]@{
        ArtifactRoot = $artifactRoot
        MetadataPath = $metadataPath
        ZipPath      = $zipPath
        HashPath     = $hashPath
    }
}

function Get-DeploymentParameterOverrides {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)]
        [string[]] $CertificatePrincipalIds,

        [Parameter(Mandatory)]
        [string[]] $FolderPrincipalIds,

        [Parameter(Mandatory)]
        [string[]] $HealthPrincipalIds
    )

    $certificateJson = $CertificatePrincipalIds | ConvertTo-Json -Compress
    $folderJson = $FolderPrincipalIds | ConvertTo-Json -Compress
    $healthJson = $HealthPrincipalIds | ConvertTo-Json -Compress

    return @(
        "certificateCollectorPrincipalIds=$certificateJson",
        "folderCollectorPrincipalIds=$folderJson",
        "healthCollectorPrincipalIds=$healthJson"
    )
}

function Build-BicepTemplate {
    [CmdletBinding()]
    param()

    Write-DeploymentLog -Level Information -Message (
        "Building Bicep template '$TemplateFile'."
    )

    Invoke-AzureCli -Arguments @(
        'bicep',
        'build',
        '--file',
        $TemplateFile
    ) | Out-Null
}

function Invoke-DeploymentValidation {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)]
        [string[]] $ParameterOverrides
    )

    Write-DeploymentLog -Level Information -Message (
        'Running Azure Resource Manager deployment validation.'
    )

    $arguments = @(
        'deployment',
        'group',
        'validate',
        '--resource-group',
        $ResourceGroupName,
        '--template-file',
        $TemplateFile,
        '--parameters',
        $ParameterFile
    ) + $ParameterOverrides + @(
        '--output',
        'json'
    )

    $validation = Invoke-AzureCli -Arguments $arguments
    return ($validation | ConvertFrom-Json -ErrorAction Stop)
}

function Invoke-DeploymentWhatIf {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)]
        [string[]] $ParameterOverrides,

        [Parameter(Mandatory)]
        [string] $ArtifactRoot
    )

    if ($SkipWhatIf) {
        Write-DeploymentLog -Level Information -Message (
            'Azure Resource Manager what-if was skipped by request.'
        )

        return
    }

    Write-DeploymentLog -Level Information -Message (
        'Running Azure Resource Manager what-if.'
    )

    $arguments = @(
        'deployment',
        'group',
        'what-if',
        '--resource-group',
        $ResourceGroupName,
        '--template-file',
        $TemplateFile,
        '--parameters',
        $ParameterFile
    ) + $ParameterOverrides + @(
        '--result-format',
        'FullResourcePayloads',
        '--output',
        'json'
    )

    $whatIfOutput = Invoke-AzureCli -Arguments $arguments
    $whatIfPath = Join-Path $ArtifactRoot 'what-if.json'

    ($whatIfOutput -join [Environment]::NewLine) |
        Set-Content -LiteralPath $whatIfPath -Encoding UTF8

    Write-DeploymentLog -Level Information -Message (
        "What-if output saved to: $whatIfPath"
    )
}

function Invoke-BicepDeployment {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)]
        [string[]] $ParameterOverrides,

        [Parameter(Mandatory)]
        [string] $ArtifactRoot
    )

    if ($SkipDeployment) {
        Write-DeploymentLog -Level Warning -Message (
            'Deployment was skipped by request. No Azure resources were changed.'
        )

        return $null
    }

    if (-not $PSCmdlet.ShouldProcess(
        $ResourceGroupName,
        "Deploy Phase 2 Azure Monitor infrastructure '$script:DeploymentName'"
    )) {
        return $null
    }

    Write-DeploymentLog -Level Information -Message (
        "Deploying '$script:DeploymentName' to resource group '$ResourceGroupName'."
    )

    $arguments = @(
        'deployment',
        'group',
        'create',
        '--name',
        $script:DeploymentName,
        '--resource-group',
        $ResourceGroupName,
        '--template-file',
        $TemplateFile,
        '--parameters',
        $ParameterFile
    ) + $ParameterOverrides + @(
        '--output',
        'json'
    )

    $deploymentJson = Invoke-AzureCli -Arguments $arguments
    $deployment = $deploymentJson | ConvertFrom-Json -ErrorAction Stop

    $deploymentPath = Join-Path $ArtifactRoot 'deployment-result.json'
    $deployment | ConvertTo-Json -Depth 32 |
        Set-Content -LiteralPath $deploymentPath -Encoding UTF8

    $outputs = $deployment.properties.outputs
    $outputsPath = Join-Path $ArtifactRoot 'deployment-outputs.json'

    $outputs | ConvertTo-Json -Depth 32 |
        Set-Content -LiteralPath $outputsPath -Encoding UTF8

    Write-DeploymentLog -Level Information -Message (
        "Deployment outputs saved to: $outputsPath"
    )

    return [pscustomobject]@{
        Deployment = $deployment
        Outputs    = $outputs
        OutputPath = $outputsPath
    }
}

function New-CollectorConfigurationTemplates {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)]
        [object] $Outputs,

        [Parameter(Mandatory)]
        [string] $ArtifactRoot
    )

    $collectorConfigDirectory = Join-Path $ArtifactRoot 'collector-config'

    New-Item -Path $collectorConfigDirectory -ItemType Directory -Force | Out-Null

    $ingestionEndpoint = [string]$Outputs.logsIngestionEndpoint.value
    $certificateDcrId = [string]$Outputs.certificateDcrImmutableId.value
    $folderDcrId = [string]$Outputs.folderMetricsDcrImmutableId.value
    $healthDcrId = [string]$Outputs.collectorHealthDcrImmutableId.value

    $certificateConfig = [ordered]@{
        SchemaVersion              = '1.0'
        CollectorName              = 'CertificateInventory'
        Enabled                    = $true
        IngestionEndpoint          = $ingestionEndpoint
        DcrImmutableId             = $certificateDcrId
        StreamName                 = 'Custom-ServerCertificateInventoryRaw'
        HealthIngestionEndpoint    = $ingestionEndpoint
        HealthDcrImmutableId       = $healthDcrId
        HealthStreamName           = 'Custom-CollectorHealthRaw'
        CertificateStores          = @('Cert:\LocalMachine\WebHosting')
        EnableIisBindingDiscovery  = $true
        ExpiryWarningDays          = 30
        ExpiryCriticalDays          = 14
    }

    $folderConfig = [ordered]@{
        SchemaVersion              = '1.0'
        CollectorName              = 'ErpFolderMetrics'
        Enabled                    = $true
        IngestionEndpoint          = $ingestionEndpoint
        DcrImmutableId             = $folderDcrId
        StreamName                 = 'Custom-ErpFolderMetricsRaw'
        HealthIngestionEndpoint    = $ingestionEndpoint
        HealthDcrImmutableId       = $healthDcrId
        HealthStreamName           = 'Custom-CollectorHealthRaw'
        DefaultRetentionDays       = 30
        MaximumScanDurationMinutes = 20
        Folders                    = @(
            [ordered]@{
                Application   = 'ERP'
                Path          = 'REPLACE_WITH_APPROVED_ERP_LOG_PATH'
                RetentionDays = 30
                Enabled       = $true
            }
        )
    }

    $certificateConfigPath = Join-Path $collectorConfigDirectory 'CertificateInventory.config.json'
    $folderConfigPath = Join-Path $collectorConfigDirectory 'ErpFolderMetrics.config.json'

    $certificateConfig | ConvertTo-Json -Depth 8 |
        Set-Content -LiteralPath $certificateConfigPath -Encoding UTF8

    $folderConfig | ConvertTo-Json -Depth 8 |
        Set-Content -LiteralPath $folderConfigPath -Encoding UTF8

    Write-DeploymentLog -Level Information -Message (
        "Collector configuration templates created in: $collectorConfigDirectory"
    )

    Write-DeploymentLog -Level Warning -Message (
        'Update REPLACE_WITH_APPROVED_ERP_LOG_PATH before copying the folder collector configuration to a server.'
    )
}

try {
    Write-DeploymentLog -Level Information -Message (
        'Starting Phase 2 Azure Monitor infrastructure deployment.'
    )

    Test-RequiredTools
    Set-AzureSubscription
    Ensure-ResourceGroup

    $certificatePrincipalIds = Get-UniqueArcPrincipalIds `
        -MachineNames $CertificateCollectorArcMachineNames

    $folderPrincipalIds = Get-UniqueArcPrincipalIds `
        -MachineNames $FolderCollectorArcMachineNames

    $healthMachineNames = @(
        $CertificateCollectorArcMachineNames +
        $FolderCollectorArcMachineNames +
        $HealthCollectorArcMachineNames
    ) | Sort-Object -Unique

    $healthPrincipalIds = Get-UniqueArcPrincipalIds `
        -MachineNames $healthMachineNames

    $artifact = New-DeploymentArtifact `
        -CertificatePrincipalIds $certificatePrincipalIds `
        -FolderPrincipalIds $folderPrincipalIds `
        -HealthPrincipalIds $healthPrincipalIds

    Build-BicepTemplate

    $parameterOverrides = Get-DeploymentParameterOverrides `
        -CertificatePrincipalIds $certificatePrincipalIds `
        -FolderPrincipalIds $folderPrincipalIds `
        -HealthPrincipalIds $healthPrincipalIds

    Invoke-DeploymentValidation -ParameterOverrides $parameterOverrides | Out-Null

    Invoke-DeploymentWhatIf `
        -ParameterOverrides $parameterOverrides `
        -ArtifactRoot $artifact.ArtifactRoot

    $deploymentResult = Invoke-BicepDeployment `
        -ParameterOverrides $parameterOverrides `
        -ArtifactRoot $artifact.ArtifactRoot

    if ($null -ne $deploymentResult) {
        New-CollectorConfigurationTemplates `
            -Outputs $deploymentResult.Outputs `
            -ArtifactRoot $artifact.ArtifactRoot

        Write-DeploymentLog -Level Information -Message (
            'Phase 2 deployment completed successfully.'
        )

        Write-Host ''
        Write-Host 'Important next actions:' -ForegroundColor Green
        Write-Host '1. Review deployment-outputs.json in the artifact folder.'
        Write-Host '2. Update the generated ERP folder configuration with the approved log path.'
        Write-Host '3. Copy generated collector configuration files to the Phase 1 package Config folder.'
        Write-Host '4. Run direct-ingestion validation on each Azure Arc collector server.'
        Write-Host '5. Verify records with KQL in the Log Analytics Workspace.'
    }
}
catch {
    Write-DeploymentLog -Level Error -Message $_.Exception.Message
    exit 1
}
```

## Example Invocation

```powershell name=Invoke-Phase2Deployment.ps1
$secureParameters = @{
    SubscriptionId                    = '<subscription-id>'
    ResourceGroupName                 = 'rg-erpmon-pilot-cac-001'
    Location                          = 'canadacentral'
    TemplateFile                      = 'C:\Source\erp-hybrid-monitoring\azure\main.bicep'
    ParameterFile                     = 'C:\Source\erp-hybrid-monitoring\azure\parameters\pilot.bicepparam'
    ArcResourceGroupName              = 'rg-arc-onprem-pilot-cac-001'
    CertificateCollectorArcMachineNames = @(
        'ERP-MW-01'
    )
    FolderCollectorArcMachineNames = @(
        'ERP-APP-01'
    )
    ArtifactDirectory                 = 'C:\DeploymentArtifacts\ERP-Monitoring'
    CreateResourceGroup               = $true
}

.\Deploy-Phase2Monitoring.ps1 @secureParameters
```

## Validation-Only Invocation

Use this first in a new subscription or region:

```powershell name=Validate-Phase2Deployment.ps1
.\Deploy-Phase2Monitoring.ps1 `
    -SubscriptionId '<subscription-id>' `
    -ResourceGroupName 'rg-erpmon-pilot-cac-001' `
    -Location 'canadacentral' `
    -TemplateFile 'C:\Source\erp-hybrid-monitoring\azure\main.bicep' `
    -ParameterFile 'C:\Source\erp-hybrid-monitoring\azure\parameters\pilot.bicepparam' `
    -ArcResourceGroupName 'rg-arc-onprem-pilot-cac-001' `
    -CertificateCollectorArcMachineNames @('ERP-MW-01') `
    -FolderCollectorArcMachineNames @('ERP-APP-01') `
    -ArtifactDirectory 'C:\DeploymentArtifacts\ERP-Monitoring' `
    -CreateResourceGroup `
    -SkipDeployment
```

---

## Post-Deployment Execution Checklist

1. Confirm all Azure resources exist.
2. Inspect `what-if.json` and `deployment-outputs.json`.
3. Update `ErpFolderMetrics.config.json` with the approved ERP log directory.
4. Copy generated configuration files into each server’s deployed collector package:
   ```text
   C:\ProgramData\Contoso\AzureMonitorCollectors\Current\Config\
   ```
5. Run the Phase 1 prerequisite test as the Scheduled Task account.
6. Run `Test-CertificateDirectIngestion.ps1` from the Arc-enabled certificate collector server.
7. Start the certificate collector task manually.
8. Query `ServerCertificateInventory_CL`.
9. Start the ERP folder collector task manually.
10. Query `ErpFolderMetrics_CL`.
11. Verify `CollectorHealth_CL` reports successful task execution.

### Want me to…
<!-- copilot:suggestions -->
1. Continue to Phase 3 with certificate-monitoring KQL, scheduled-query alerts, email/Teams notification design, and Workbook dashboard implementation.
2. Review and harden the Phase 2 Bicep templates for production, including API-version validation and RBAC-scope refinements.
3. Hand this Bicep deployment package off to the coding agent to open a pull request in a repository you provide.
