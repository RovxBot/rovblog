# Azure Infrastructure Automation

!!! info "Automated Azure Resource Provisioning"
    This post covers the complete automation of Azure infrastructure provisioning for client onboarding, including Resource Groups, Azure Data Factory, Key Vault, and Service Principal configuration.

## Overview

The foundation of our automated client onboarding system is the ability to provision complete Azure environments with a single PowerShell command. This infrastructure automation creates isolated, secure environments for each client while maintaining consistency and following enterprise best practices.

## Step 1: Base Azure Infrastructure

Each client receives a dedicated set of Azure resources that are completely isolated from other clients:

### Core Components

- **Resource Group**: Isolated container for all client resources
- **Azure Data Factory**: Data integration and ETL service instance
- **Key Vault**: Secure credential and secret management
- **Service Principal**: Automated identity for Microsoft Fabric access

### Automated Provisioning Script

The PowerShell automation script handles the complete infrastructure setup:

```powershell
function New-ClientInfrastructure {
    param(
        [Parameter(Mandatory = $true)]
        [string]$ClientName,

        [Parameter(Mandatory = $true)]
        [string]$SubscriptionId,

        [Parameter(Mandatory = $true)]
        [string]$Location = "Australia East"
    )

    Write-Host "Starting infrastructure provisioning for client: $ClientName" -ForegroundColor Green

    # Set Azure context
    Set-AzContext -SubscriptionId $SubscriptionId

    # Create Resource Group
    $resourceGroupName = "rg-$ClientName-data-platform"
    Write-Host "Creating Resource Group: $resourceGroupName"

    $resourceGroup = New-AzResourceGroup -Name $resourceGroupName -Location $Location -Tag @{
        Client = $ClientName
        Environment = "Production"
        Purpose = "DataPlatform"
        CreatedBy = "AutomationScript"
        CreatedDate = (Get-Date).ToString("yyyy-MM-dd")
    }

    # Create Azure Data Factory
    $adfName = "adf-$ClientName-prod"
    Write-Host "Creating Azure Data Factory: $adfName"

    $dataFactory = New-AzDataFactoryV2 -ResourceGroupName $resourceGroupName -Name $adfName -Location $Location

    # Create Key Vault with unique name
    $keyVaultName = "kv-$ClientName-$(Get-Random -Minimum 1000 -Maximum 9999)"
    Write-Host "Creating Key Vault: $keyVaultName"

    $keyVault = New-AzKeyVault -ResourceGroupName $resourceGroupName -VaultName $keyVaultName -Location $Location -EnabledForTemplateDeployment

    return @{
        ResourceGroup = $resourceGroup
        DataFactory = $dataFactory
        KeyVault = $keyVault
    }
}
```

## Service Principal Creation for Fabric Access

Creating a service principal that Azure Data Factory can use to access Microsoft Fabric:

```powershell
function New-FabricServicePrincipal {
    param(
        [Parameter(Mandatory = $true)]
        [string]$ClientName,

        [Parameter(Mandatory = $true)]
        [string]$KeyVaultName
    )

    Write-Host "Creating Service Principal for Fabric access"

    # Create Azure AD Application
    $appName = "sp-$ClientName-fabric-access"
    $app = New-AzADApplication -DisplayName $appName

    # Create Service Principal
    $sp = New-AzADServicePrincipal -ApplicationId $app.AppId

    # Generate client secret
    $clientSecret = New-AzADAppCredential -ApplicationId $app.AppId

    # Store credentials in Key Vault
    Set-AzKeyVaultSecret -VaultName $KeyVaultName -Name "Fabric-SP-TenantId" -SecretValue (ConvertTo-SecureString -String (Get-AzContext).Tenant.Id -AsPlainText -Force)
    Set-AzKeyVaultSecret -VaultName $KeyVaultName -Name "Fabric-SP-ClientId" -SecretValue (ConvertTo-SecureString -String $app.AppId -AsPlainText -Force)
    Set-AzKeyVaultSecret -VaultName $KeyVaultName -Name "Fabric-SP-Secret" -SecretValue (ConvertTo-SecureString -String $clientSecret.SecretText -AsPlainText -Force)

    Write-Host "Service Principal created and credentials stored in Key Vault"

    return @{
        TenantId = (Get-AzContext).Tenant.Id
        ClientId = $app.AppId
        ClientSecret = $clientSecret.SecretText
        ServicePrincipal = $sp
    }
}
```

## Key Vault Integration

Configuring Azure Data Factory to access Key Vault secrets:

```powershell
function Set-ADFKeyVaultAccess {
    param(
        [Parameter(Mandatory = $true)]
        [string]$ResourceGroupName,

        [Parameter(Mandatory = $true)]
        [string]$DataFactoryName,

        [Parameter(Mandatory = $true)]
        [string]$KeyVaultName
    )

    Write-Host "Configuring ADF access to Key Vault"

    # Get ADF Managed Identity
    $adf = Get-AzDataFactoryV2 -ResourceGroupName $ResourceGroupName -Name $DataFactoryName
    $adfIdentity = $adf.Identity.PrincipalId

    # Grant ADF access to Key Vault
    Set-AzKeyVaultAccessPolicy -VaultName $KeyVaultName -ObjectId $adfIdentity -PermissionsToSecrets Get,List

    Write-Host "ADF granted access to Key Vault secrets"
}
```

## Client API Credentials Management

Storing client-specific API credentials securely:

```powershell
function Set-ClientAPICredentials {
    param(
        [Parameter(Mandatory = $true)]
        [string]$ClientName,

        [Parameter(Mandatory = $true)]
        [string]$KeyVaultName,

        [Parameter(Mandatory = $true)]
        [string]$ApiKey,

        [Parameter(Mandatory = $true)]
        [string]$ApiBaseUrl
    )

    Write-Host "Storing client API credentials in Key Vault"

    # Store client-specific secrets
    $clientPrefix = "$ClientName-API"

    Set-AzKeyVaultSecret -VaultName $KeyVaultName -Name "$clientPrefix-Key" -SecretValue (ConvertTo-SecureString -String $ApiKey -AsPlainText -Force)
    Set-AzKeyVaultSecret -VaultName $KeyVaultName -Name "$clientPrefix-BaseUrl" -SecretValue (ConvertTo-SecureString -String $ApiBaseUrl -AsPlainText -Force)

    Write-Host "Client API credentials stored securely"
}
```

## Complete Infrastructure Setup

The main orchestration function that ties everything together:

```powershell
function Initialize-ClientInfrastructure {
    param(
        [Parameter(Mandatory = $true)]
        [string]$ClientName,

        [Parameter(Mandatory = $true)]
        [string]$SubscriptionId,

        [Parameter(Mandatory = $true)]
        [string]$ApiKey,

        [Parameter(Mandatory = $true)]
        [string]$ApiBaseUrl,

        [string]$Location = "Australia East"
    )

    try {
        Write-Host "Starting complete infrastructure setup for: $ClientName" -ForegroundColor Cyan

        # Step 1: Create base Azure resources
        $infrastructure = New-ClientInfrastructure -ClientName $ClientName -SubscriptionId $SubscriptionId -Location $Location

        # Step 2: Create Service Principal for Fabric
        $servicePrincipal = New-FabricServicePrincipal -ClientName $ClientName -KeyVaultName $infrastructure.KeyVault.VaultName

        # Step 3: Configure ADF Key Vault access
        Set-ADFKeyVaultAccess -ResourceGroupName $infrastructure.ResourceGroup.ResourceGroupName -DataFactoryName $infrastructure.DataFactory.DataFactoryName -KeyVaultName $infrastructure.KeyVault.VaultName

        # Step 4: Store client API credentials
        Set-ClientAPICredentials -ClientName $ClientName -KeyVaultName $infrastructure.KeyVault.VaultName -ApiKey $ApiKey -ApiBaseUrl $ApiBaseUrl

        Write-Host "Infrastructure setup completed successfully!" -ForegroundColor Green

        return @{
            ResourceGroup = $infrastructure.ResourceGroup.ResourceGroupName
            DataFactory = $infrastructure.DataFactory.DataFactoryName
            KeyVault = $infrastructure.KeyVault.VaultName
            ServicePrincipal = $servicePrincipal
        }
    }
    catch {
        Write-Error "Infrastructure setup failed: $($_.Exception.Message)"
        throw
    }
}
```

## Usage Example

Here's how to use the infrastructure automation:

```powershell
# Install required modules
Install-Module -Name Az -Force -AllowClobber

# Connect to Azure
Connect-AzAccount

# Run complete infrastructure setup
$result = Initialize-ClientInfrastructure -ClientName "Contoso" -SubscriptionId "your-subscription-id" -ApiKey "client-api-key" -ApiBaseUrl "https://api.contoso.com"

# Output results
Write-Host "Infrastructure created:"
Write-Host "Resource Group: $($result.ResourceGroup)"
Write-Host "Data Factory: $($result.DataFactory)"
Write-Host "Key Vault: $($result.KeyVault)"
```

## Key Benefits

### Isolation and Security
- **Complete resource isolation** per client
- **Secure credential management** with Key Vault
- **Managed identity integration** for ADF
- **Automated access control** configuration

### Consistency and Compliance
- **Standardised naming conventions** across all resources
- **Consistent tagging strategy** for resource management
- **Automated compliance** with security best practices
- **Audit trail** through resource tags and logging

### Scalability and Automation
- **Repeatable infrastructure** for any number of clients
- **Error handling and validation** built into scripts
- **Modular functions** for easy maintenance and updates
- **Integration ready** for the next automation steps

## Next Steps

With the Azure infrastructure automated, the next phase involves:

1. **[Repository Template System](IngestionPipelines.md)** - Automated GitHub repository creation
2. **[Microsoft Fabric Automation](PowerBI.md)** - Workspace and pipeline provisioning
3. **[Complete Integration](PowerPages.md)** - End-to-end automation orchestration

This infrastructure automation forms the foundation that enables the entire client onboarding process to be completed in minutes rather than hours or days.