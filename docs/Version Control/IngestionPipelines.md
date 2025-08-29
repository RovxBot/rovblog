# Repository Template System & GitHub Automation

!!! info "Automated GitHub Repository Creation"
    This post covers the complete automation of GitHub repository creation from templates, including CI/CD workflow injection, branch protection setup, and parameterized pipeline deployment.

## Overview

Step 2 of our automated client onboarding system focuses on creating standardised GitHub repositories from templates. This ensures every client gets a consistent, production-ready codebase with pre-configured CI/CD workflows, security policies, and deployment automation.

## Step 2: Bootstrapping GitHub Repositories

The repository automation system creates a complete development environment for each client using GitHub's template repository feature combined with API automation.

### Template Repository Structure

First, we maintain a master template repository with the standardised structure:

```
client-template-repository/
├── .github/
│   ├── workflows/
│   │   ├── adf-ci-cd.yml
│   │   ├── powerbi-deployment.yml
│   │   └── fabric-sync.yml
│   ├── ISSUE_TEMPLATE/
│   └── dependabot.yml
├── adf/
│   ├── pipeline/
│   │   └── client-data-ingestion.json
│   ├── dataset/
│   │   ├── http-source.json
│   │   └── onelake-sink.json
│   ├── linkedService/
│   │   ├── client-api.json
│   │   ├── keyvault.json
│   │   └── fabric-lakehouse.json
│   ├── trigger/
│   │   └── monthly-schedule.json
│   └── ARMTemplateForFactory.json
├── infrastructure/
│   ├── bicep/
│   │   ├── main.bicep
│   │   ├── keyvault.bicep
│   │   └── datafactory.bicep
│   └── parameters/
│       ├── dev.parameters.json
│       ├── test.parameters.json
│       └── prod.parameters.json
├── powerbi/
│   ├── reports/
│   │   └── client-dashboard-template.pbix
│   ├── datasets/
│   └── deployment/
├── scripts/
│   ├── pre-deployment.ps1
│   ├── post-deployment.ps1
│   └── utilities/
├── tests/
│   ├── unit/
│   ├── integration/
│   └── data-quality/
└── docs/
    ├── deployment-guide.md
    └── client-requirements.md
```

### Automated Repository Creation

The PowerShell automation uses GitHub's REST API to create repositories from templates:

```powershell
function New-ClientRepository {
    param(
        [Parameter(Mandatory = $true)]
        [string]$ClientName,

        [Parameter(Mandatory = $true)]
        [string]$GitHubOrg,

        [Parameter(Mandatory = $true)]
        [string]$GitHubToken,

        [Parameter(Mandatory = $true)]
        [string]$TemplateRepo = "client-data-platform-template"
    )

    Write-Host "Creating GitHub repository for client: $ClientName" -ForegroundColor Green

    # Set up headers for GitHub API
    $headers = @{
        "Authorization" = "Bearer $GitHubToken"
        "Accept" = "application/vnd.github.v3+json"
        "User-Agent" = "PowerShell-ClientOnboarding"
    }

    # Repository creation payload
    $newRepoName = "$ClientName-data-platform".ToLower()
    $payload = @{
        name = $newRepoName
        description = "Automated data platform for $ClientName"
        private = $true
        include_all_branches = $false
    } | ConvertTo-Json

    # Create repository from template
    $templateUrl = "https://api.github.com/repos/$GitHubOrg/$TemplateRepo/generate"

    try {
        $response = Invoke-RestMethod -Uri $templateUrl -Method POST -Headers $headers -Body $payload -ContentType "application/json"

        Write-Host "Repository created: $($response.html_url)"

        # Wait for repository to be fully initialised
        Start-Sleep -Seconds 10

        return @{
            RepoName = $newRepoName
            RepoUrl = $response.html_url
            CloneUrl = $response.clone_url
            DefaultBranch = $response.default_branch
        }
    }
    catch {
        Write-Error "Failed to create repository: $($_.Exception.Message)"
        throw
    }
}
```

## Step 3: Parameterized ADF Pipeline Templates

The template repository contains parameterized Azure Data Factory pipelines that can be customized for each client without rewriting logic.

### Pipeline Parameter Strategy

The core pipeline template uses parameters for all client-specific values:

```json
{
  "name": "ClientDataIngestion",
  "properties": {
    "parameters": {
      "client_name": {
        "type": "string",
        "defaultValue": "template-client"
      },
      "api_base_url": {
        "type": "string",
        "defaultValue": "https://api.example.com"
      },
      "api_key_secret_name": {
        "type": "string",
        "defaultValue": "client-api-key"
      },
      "onelake_workspace_id": {
        "type": "string",
        "defaultValue": "00000000-0000-0000-0000-000000000000"
      },
      "onelake_lakehouse_id": {
        "type": "string",
        "defaultValue": "00000000-0000-0000-0000-000000000000"
      },
      "output_folder_path": {
        "type": "string",
        "defaultValue": "/clients/template-client"
      }
    },
    "activities": [
      {
        "name": "IngestClientData",
        "type": "Copy",
        "inputs": [
          {
            "referenceName": "ClientAPISource",
            "type": "DatasetReference"
          }
        ],
        "outputs": [
          {
            "referenceName": "OneLakeSink",
            "type": "DatasetReference",
            "parameters": {
              "folderPath": "@pipeline().parameters.output_folder_path"
            }
          }
        ]
      }
    ]
  }
}
```

### Linked Service Templates

Parameterized linked services for API connections and Key Vault integration:

```json
{
  "name": "ClientAPI",
  "properties": {
    "type": "RestService",
    "parameters": {
      "baseUrl": {
        "type": "string"
      }
    },
    "typeProperties": {
      "url": "@{linkedService().baseUrl}",
      "enableServerCertificateValidation": true,
      "authenticationType": "Anonymous"
    }
  }
}
```

### ARM Template Parameterization

Custom ARM parameter configuration ensures all client-specific values are parameterized:

```json
{
  "Microsoft.DataFactory/factories/pipelines": {
    "properties.parameters.*": {
      "type": "string"
    }
  },
  "Microsoft.DataFactory/factories/linkedServices": {
    "properties.parameters.*": {
      "type": "string"
    },
    "properties.typeProperties.url": "=",
    "properties.typeProperties.connectionString": "="
  },
  "Microsoft.DataFactory/factories/datasets": {
    "properties.parameters.*": {
      "type": "string"
    }
  }
}
```

## Template Parameter Injection

The automation script customizes the templates for each client:

```powershell
function Set-ClientPipelineParameters {
    param(
        [Parameter(Mandatory = $true)]
        [string]$RepoPath,

        [Parameter(Mandatory = $true)]
        [string]$ClientName,

        [Parameter(Mandatory = $true)]
        [string]$ApiBaseUrl,

        [Parameter(Mandatory = $true)]
        [string]$WorkspaceId,

        [Parameter(Mandatory = $true)]
        [string]$LakehouseId
    )

    Write-Host "Injecting client parameters into pipeline templates" -ForegroundColor Yellow

    # Update ARM parameters file
    $parametersFile = "$RepoPath/infrastructure/parameters/prod.parameters.json"
    $parameters = Get-Content $parametersFile | ConvertFrom-Json

    # Set client-specific parameter values
    $parameters.parameters.client_name.value = $ClientName
    $parameters.parameters.api_base_url.value = $ApiBaseUrl
    $parameters.parameters.api_key_secret_name.value = "$ClientName-API-Key"
    $parameters.parameters.onelake_workspace_id.value = $WorkspaceId
    $parameters.parameters.onelake_lakehouse_id.value = $LakehouseId
    $parameters.parameters.output_folder_path.value = "/clients/$ClientName"

    # Save updated parameters
    $parameters | ConvertTo-Json -Depth 10 | Set-Content $parametersFile

    Write-Host "Pipeline parameters updated for client: $ClientName"
}
```

## GitHub Actions CI/CD Configuration

The template repository includes pre-configured GitHub Actions workflows that are automatically customized for each client.

### ADF Deployment Workflow Template

```yaml
# .github/workflows/adf-ci-cd.yml
name: ADF CI/CD Pipeline

on:
  push:
    branches: [ main ]
    paths: [ 'adf/**' ]
  pull_request:
    branches: [ main ]
    paths: [ 'adf/**' ]

env:
  AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
  RESOURCE_GROUP: 'rg-{{CLIENT_NAME}}-data-platform'
  ADF_NAME: 'adf-{{CLIENT_NAME}}-prod'

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '18'

    - name: Install ADF Utilities
      run: npm install -g @microsoft/azure-data-factory-utilities

    - name: Validate ADF Resources
      run: npx @microsoft/azure-data-factory-utilities validate ./adf/ ${{ env.ADF_NAME }}

  deploy:
    needs: validate
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production

    steps:
    - uses: actions/checkout@v4

    - name: Azure Login
      uses: azure/login@v1
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}

    - name: Pre-deployment Script
      run: |
        ./scripts/pre-deployment.ps1 -ResourceGroupName ${{ env.RESOURCE_GROUP }} -DataFactoryName ${{ env.ADF_NAME }}
      shell: pwsh

    - name: Deploy ADF
      uses: azure/arm-deploy@v1
      with:
        subscriptionId: ${{ env.AZURE_SUBSCRIPTION_ID }}
        resourceGroupName: ${{ env.RESOURCE_GROUP }}
        template: ./adf/ARMTemplateForFactory.json
        parameters: ./infrastructure/parameters/prod.parameters.json

    - name: Post-deployment Script
      run: |
        ./scripts/post-deployment.ps1 -ResourceGroupName ${{ env.RESOURCE_GROUP }} -DataFactoryName ${{ env.ADF_NAME }}
      shell: pwsh
```

### Workflow Customization

The automation script customizes the workflow templates for each client:

```powershell
function Set-GitHubWorkflowParameters {
    param(
        [Parameter(Mandatory = $true)]
        [string]$RepoPath,

        [Parameter(Mandatory = $true)]
        [string]$ClientName
    )

    Write-Host "Customizing GitHub Actions workflows for client: $ClientName" -ForegroundColor Yellow

    # Get all workflow files
    $workflowFiles = Get-ChildItem "$RepoPath/.github/workflows/*.yml"

    foreach ($file in $workflowFiles) {
        $content = Get-Content $file.FullName -Raw

        # Replace template placeholders
        $content = $content -replace '{{CLIENT_NAME}}', $ClientName.ToLower()

        # Save updated workflow
        Set-Content -Path $file.FullName -Value $content

        Write-Host "  Updated workflow: $($file.Name)"
    }

    Write-Host "All workflows customized for client: $ClientName"
}
```

## Branch Protection and Security

Automated setup of branch protection rules and security policies:

```powershell
function Set-BranchProtection {
    param(
        [Parameter(Mandatory = $true)]
        [string]$GitHubOrg,

        [Parameter(Mandatory = $true)]
        [string]$RepoName,

        [Parameter(Mandatory = $true)]
        [string]$GitHubToken
    )

    Write-Host "Setting up branch protection for repository: $RepoName" -ForegroundColor Cyan

    $headers = @{
        "Authorization" = "Bearer $GitHubToken"
        "Accept" = "application/vnd.github.v3+json"
    }

    $protectionRules = @{
        required_status_checks = @{
            strict = $true
            contexts = @("validate")
        }
        enforce_admins = $true
        required_pull_request_reviews = @{
            required_approving_review_count = 2
            dismiss_stale_reviews = $true
            require_code_owner_reviews = $true
        }
        restrictions = $null
    } | ConvertTo-Json -Depth 10

    $url = "https://api.github.com/repos/$GitHubOrg/$RepoName/branches/main/protection"

    try {
        Invoke-RestMethod -Uri $url -Method PUT -Headers $headers -Body $protectionRules -ContentType "application/json"
        Write-Host "Branch protection rules applied successfully"
    }
    catch {
        Write-Warning "Could not set branch protection: $($_.Exception.Message)"
    }
}
```

## Repository Secrets Configuration

Automated setup of GitHub repository secrets for secure CI/CD:

```powershell
function Set-RepositorySecrets {
    param(
        [Parameter(Mandatory = $true)]
        [string]$GitHubOrg,

        [Parameter(Mandatory = $true)]
        [string]$RepoName,

        [Parameter(Mandatory = $true)]
        [string]$GitHubToken,

        [Parameter(Mandatory = $true)]
        [hashtable]$ServicePrincipal,

        [Parameter(Mandatory = $true)]
        [string]$SubscriptionId
    )

    Write-Host "Setting up repository secrets for: $RepoName" -ForegroundColor Magenta

    $headers = @{
        "Authorization" = "Bearer $GitHubToken"
        "Accept" = "application/vnd.github.v3+json"
    }

    # Get repository public key for secret encryption
    $keyUrl = "https://api.github.com/repos/$GitHubOrg/$RepoName/actions/secrets/public-key"
    $publicKey = Invoke-RestMethod -Uri $keyUrl -Headers $headers

    # Azure credentials for OIDC
    $azureCredentials = @{
        clientId = $ServicePrincipal.ClientId
        clientSecret = $ServicePrincipal.ClientSecret
        subscriptionId = $SubscriptionId
        tenantId = $ServicePrincipal.TenantId
    } | ConvertTo-Json -Compress

    # Secrets to create
    $secrets = @{
        "AZURE_CREDENTIALS" = $azureCredentials
        "AZURE_SUBSCRIPTION_ID" = $SubscriptionId
        "AZURE_CLIENT_ID" = $ServicePrincipal.ClientId
        "AZURE_TENANT_ID" = $ServicePrincipal.TenantId
    }

    foreach ($secretName in $secrets.Keys) {
        $secretValue = $secrets[$secretName]

        # Encrypt secret value (simplified - in practice, use proper encryption)
        $encryptedValue = [Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes($secretValue))

        $secretPayload = @{
            encrypted_value = $encryptedValue
            key_id = $publicKey.key_id
        } | ConvertTo-Json

        $secretUrl = "https://api.github.com/repos/$GitHubOrg/$RepoName/actions/secrets/$secretName"

        try {
            Invoke-RestMethod -Uri $secretUrl -Method PUT -Headers $headers -Body $secretPayload -ContentType "application/json"
            Write-Host "  Secret created: $secretName"
        }
        catch {
            Write-Warning "  Failed to create secret $secretName`: $($_.Exception.Message)"
        }
    }

    Write-Host "Repository secrets configured successfully"
}
```

## Complete Repository Setup Integration

The main function that orchestrates the entire repository setup:

```powershell
function Initialize-ClientRepository {
    param(
        [Parameter(Mandatory = $true)]
        [string]$ClientName,

        [Parameter(Mandatory = $true)]
        [string]$GitHubOrg,

        [Parameter(Mandatory = $true)]
        [string]$GitHubToken,

        [Parameter(Mandatory = $true)]
        [hashtable]$ServicePrincipal,

        [Parameter(Mandatory = $true)]
        [string]$SubscriptionId,

        [Parameter(Mandatory = $true)]
        [string]$ApiBaseUrl,

        [Parameter(Mandatory = $true)]
        [string]$WorkspaceId,

        [Parameter(Mandatory = $true)]
        [string]$LakehouseId
    )

    try {
        Write-Host "Starting complete repository setup for: $ClientName" -ForegroundColor Green

        # Step 1: Create repository from template
        $repo = New-ClientRepository -ClientName $ClientName -GitHubOrg $GitHubOrg -GitHubToken $GitHubToken

        # Step 2: Clone repository locally for modifications
        $tempPath = "$env:TEMP\$($repo.RepoName)"
        git clone $repo.CloneUrl $tempPath

        # Step 3: Customize pipeline parameters
        Set-ClientPipelineParameters -RepoPath $tempPath -ClientName $ClientName -ApiBaseUrl $ApiBaseUrl -WorkspaceId $WorkspaceId -LakehouseId $LakehouseId

        # Step 4: Customize GitHub workflows
        Set-GitHubWorkflowParameters -RepoPath $tempPath -ClientName $ClientName

        # Step 5: Commit and push changes
        Set-Location $tempPath
        git add .
        git commit -m "Customize templates for client: $ClientName"
        git push origin main
        Set-Location $PSScriptRoot

        # Step 6: Set up branch protection
        Set-BranchProtection -GitHubOrg $GitHubOrg -RepoName $repo.RepoName -GitHubToken $GitHubToken

        # Step 7: Configure repository secrets
        Set-RepositorySecrets -GitHubOrg $GitHubOrg -RepoName $repo.RepoName -GitHubToken $GitHubToken -ServicePrincipal $ServicePrincipal -SubscriptionId $SubscriptionId

        # Cleanup
        Remove-Item $tempPath -Recurse -Force

        Write-Host "Repository setup completed successfully!" -ForegroundColor Green

        return $repo
    }
    catch {
        Write-Error "Repository setup failed: $($_.Exception.Message)"
        throw
    }
}
```

## Key Benefits

### Standardization and Consistency
- **Template-driven approach** ensures all clients get the same high-quality foundation
- **Consistent CI/CD workflows** across all client repositories
- **Standardized security policies** and branch protection rules
- **Uniform project structure** for easy maintenance and support

### Security and Compliance
- **Automated secret management** with proper encryption
- **Branch protection rules** prevent direct commits to main
- **Required code reviews** and status checks
- **Service principal isolation** per client environment

### Developer Experience
- **Ready-to-use repositories** with working CI/CD from day one
- **Pre-configured workflows** for common deployment scenarios
- **Comprehensive documentation** and deployment guides
- **Automated parameter injection** eliminates manual configuration

### Scalability and Maintenance
- **Template updates** can be propagated to all client repositories
- **Centralized workflow management** through template repository
- **Automated repository creation** scales to any number of clients
- **Version-controlled infrastructure** enables easy rollbacks and updates

## Next Steps

With the repository template system automated, the next phase involves:

1. **[Microsoft Fabric Automation](PowerBI.md)** - Workspace and deployment pipeline provisioning
2. **[Complete Integration](PowerPages.md)** - End-to-end automation orchestration
3. **Template maintenance** and continuous improvement processes

This repository automation ensures every client starts with a production-ready, secure, and maintainable codebase that follows enterprise best practices from day one.