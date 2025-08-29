# Microsoft Fabric Workspace Automation

!!! info "Automated Fabric Environment Provisioning"
    This post covers the complete automation of Microsoft Fabric workspace creation, deployment pipeline setup, OneLake configuration, and Power BI report deployment for client onboarding.

## Overview

Steps 7-8 of our automated client onboarding system focus on provisioning Microsoft Fabric environments and connecting Power BI reports to client data. This creates isolated Dev/Test/Prod workspaces with automated deployment pipelines and secure data connections.

## Step 7: Fabric Workspaces Provisioning

The Fabric automation creates three workspaces per client (Dev, Test, Prod) and links them through a deployment pipeline for proper release management.

### Workspace Creation via Power BI REST API

```powershell
function New-FabricWorkspaces {
    param(
        [Parameter(Mandatory = $true)]
        [string]$ClientName,

        [Parameter(Mandatory = $true)]
        [string]$AccessToken
    )

    Write-Host "Creating Fabric workspaces for client: $ClientName" -ForegroundColor Green

    $headers = @{
        "Authorization" = "Bearer $AccessToken"
        "Content-Type" = "application/json"
    }

    $workspaces = @()
    $environments = @("Dev", "Test", "Prod")

    foreach ($env in $environments) {
        $workspaceName = "$ClientName-$env"

        Write-Host "  Creating workspace: $workspaceName"

        $payload = @{
            name = $workspaceName
            description = "$ClientName data platform - $env environment"
        } | ConvertTo-Json

        try {
            $response = Invoke-RestMethod -Uri "https://api.powerbi.com/v1.0/myorg/groups?workspaceV2=true" -Method POST -Headers $headers -Body $payload

            $workspaces += @{
                Environment = $env
                Name = $workspaceName
                Id = $response.id
                Url = "https://app.powerbi.com/groups/$($response.id)"
            }

            Write-Host "    Created: $workspaceName (ID: $($response.id))"
            Start-Sleep -Seconds 2  # Rate limiting
        }
        catch {
            Write-Error "    Failed to create workspace $workspaceName`: $($_.Exception.Message)"
            throw
        }
    }

    return $workspaces
}
```

### Deployment Pipeline Creation

Creating and configuring the deployment pipeline to link Dev → Test → Prod:

```powershell
function New-FabricDeploymentPipeline {
    param(
        [Parameter(Mandatory = $true)]
        [string]$ClientName,

        [Parameter(Mandatory = $true)]
        [array]$Workspaces,

        [Parameter(Mandatory = $true)]
        [string]$AccessToken
    )

    Write-Host "Creating deployment pipeline for client: $ClientName" -ForegroundColor Cyan

    $headers = @{
        "Authorization" = "Bearer $AccessToken"
        "Content-Type" = "application/json"
    }

    # Create deployment pipeline
    $pipelineName = "$ClientName-Release-Pipeline"
    $pipelinePayload = @{
        displayName = $pipelineName
        description = "Automated deployment pipeline for $ClientName data platform"
    } | ConvertTo-Json

    try {
        $pipeline = Invoke-RestMethod -Uri "https://api.powerbi.com/v1.0/myorg/pipelines" -Method POST -Headers $headers -Body $pipelinePayload

        Write-Host "  Created deployment pipeline: $pipelineName (ID: $($pipeline.id))"

        # Assign workspaces to pipeline stages
        $stageMapping = @{
            "Dev" = 0
            "Test" = 1
            "Prod" = 2
        }

        foreach ($workspace in $Workspaces) {
            $stageOrder = $stageMapping[$workspace.Environment]

            $assignPayload = @{
                workspaceId = $workspace.Id
            } | ConvertTo-Json

            $assignUrl = "https://api.powerbi.com/v1.0/myorg/pipelines/$($pipeline.id)/stages/$stageOrder/assignWorkspace"

            Invoke-RestMethod -Uri $assignUrl -Method POST -Headers $headers -Body $assignPayload

            Write-Host "    Assigned $($workspace.Name) to $($workspace.Environment) stage"
            Start-Sleep -Seconds 1
        }

        return @{
            PipelineId = $pipeline.id
            PipelineName = $pipelineName
            Url = "https://app.powerbi.com/groups/me/pipelines/$($pipeline.id)"
        }
    }
    catch {
        Write-Error "Failed to create deployment pipeline: $($_.Exception.Message)"
        throw
    }
}
```

### OneLake Lakehouse Creation

Creating Lakehouse items in each workspace for data storage:

```powershell
function New-OneLakeLakehouses {
    param(
        [Parameter(Mandatory = $true)]
        [string]$ClientName,

        [Parameter(Mandatory = $true)]
        [array]$Workspaces,

        [Parameter(Mandatory = $true)]
        [string]$AccessToken
    )

    Write-Host "Creating OneLake Lakehouses for client: $ClientName" -ForegroundColor Blue

    $headers = @{
        "Authorization" = "Bearer $AccessToken"
        "Content-Type" = "application/json"
    }

    $lakehouses = @()

    foreach ($workspace in $Workspaces) {
        $lakehouseName = "$ClientName-Lakehouse-$($workspace.Environment)"

        Write-Host "  Creating Lakehouse: $lakehouseName in $($workspace.Name)"

        $payload = @{
            displayName = $lakehouseName
            description = "Data lakehouse for $ClientName - $($workspace.Environment) environment"
        } | ConvertTo-Json

        try {
            $lakehouseUrl = "https://api.fabric.microsoft.com/v1/workspaces/$($workspace.Id)/lakehouses"
            $response = Invoke-RestMethod -Uri $lakehouseUrl -Method POST -Headers $headers -Body $payload

            $lakehouses += @{
                Environment = $workspace.Environment
                WorkspaceId = $workspace.Id
                LakehouseId = $response.id
                LakehouseName = $lakehouseName
                OneLakePath = "abfss://$($workspace.Id)@onelake.dfs.fabric.microsoft.com/$($response.id).Lakehouse"
            }

            Write-Host "    Created: $lakehouseName (ID: $($response.id))"
            Start-Sleep -Seconds 2
        }
        catch {
            Write-Error "    Failed to create Lakehouse $lakehouseName`: $($_.Exception.Message)"
            throw
        }
    }

    return $lakehouses
}
```

## Step 8: Power BI Data Binding

Connecting Power BI reports to client-specific OneLake data and configuring automated deployment.

### PBIX Template Deployment

Deploying template Power BI reports and connecting them to client data:

```powershell
function Deploy-PowerBIReports {
    param(
        [Parameter(Mandatory = $true)]
        [string]$ClientName,

        [Parameter(Mandatory = $true)]
        [array]$Workspaces,

        [Parameter(Mandatory = $true)]
        [array]$Lakehouses,

        [Parameter(Mandatory = $true)]
        [string]$AccessToken,

        [Parameter(Mandatory = $true)]
        [string]$TemplatePbixPath
    )

    Write-Host "Deploying Power BI reports for client: $ClientName" -ForegroundColor Magenta

    $headers = @{
        "Authorization" = "Bearer $AccessToken"
    }

    $deployedReports = @()

    # Deploy to Dev workspace first
    $devWorkspace = $Workspaces | Where-Object { $_.Environment -eq "Dev" }
    $devLakehouse = $Lakehouses | Where-Object { $_.Environment -eq "Dev" }

    if ($devWorkspace -and $devLakehouse) {
        Write-Host "  Deploying report to Dev workspace: $($devWorkspace.Name)"

        # Import PBIX file
        $importUrl = "https://api.powerbi.com/v1.0/myorg/groups/$($devWorkspace.Id)/imports"

        # Create multipart form data for file upload
        $boundary = [System.Guid]::NewGuid().ToString()
        $bodyLines = @(
            "--$boundary",
            'Content-Disposition: form-data; name="file"; filename="' + (Split-Path $TemplatePbixPath -Leaf) + '"',
            'Content-Type: application/octet-stream',
            '',
            [System.IO.File]::ReadAllText($TemplatePbixPath),
            "--$boundary--"
        )
        $body = $bodyLines -join "`r`n"

        $importHeaders = $headers.Clone()
        $importHeaders["Content-Type"] = "multipart/form-data; boundary=$boundary"

        try {
            $importResponse = Invoke-RestMethod -Uri $importUrl -Method POST -Headers $importHeaders -Body $body

            # Wait for import to complete
            do {
                Start-Sleep -Seconds 5
                $importStatus = Invoke-RestMethod -Uri "https://api.powerbi.com/v1.0/myorg/groups/$($devWorkspace.Id)/imports/$($importResponse.id)" -Headers $headers
            } while ($importStatus.importState -eq "Publishing")

            if ($importStatus.importState -eq "Succeeded") {
                Write-Host "    Report imported successfully"

                # Update data source connection
                $datasetId = $importStatus.datasets[0].id
                Update-PowerBIDataSource -WorkspaceId $devWorkspace.Id -DatasetId $datasetId -OneLakePath $devLakehouse.OneLakePath -AccessToken $AccessToken

                $deployedReports += @{
                    Environment = "Dev"
                    WorkspaceId = $devWorkspace.Id
                    ReportId = $importStatus.reports[0].id
                    DatasetId = $datasetId
                    ReportName = "$ClientName Dashboard"
                }
            }
            else {
                Write-Error "    Report import failed: $($importStatus.error.message)"
            }
        }
        catch {
            Write-Error "    Failed to deploy report: $($_.Exception.Message)"
        }
    }

    return $deployedReports
}
```

### Data Source Connection Update

Updating Power BI dataset connections to point to client-specific OneLake data:

```powershell
function Update-PowerBIDataSource {
    param(
        [Parameter(Mandatory = $true)]
        [string]$WorkspaceId,

        [Parameter(Mandatory = $true)]
        [string]$DatasetId,

        [Parameter(Mandatory = $true)]
        [string]$OneLakePath,

        [Parameter(Mandatory = $true)]
        [string]$AccessToken
    )

    Write-Host "  Updating data source connection for dataset: $DatasetId"

    $headers = @{
        "Authorization" = "Bearer $AccessToken"
        "Content-Type" = "application/json"
    }

    try {
        # Get current data sources
        $dataSourcesUrl = "https://api.powerbi.com/v1.0/myorg/groups/$WorkspaceId/datasets/$DatasetId/datasources"
        $dataSources = Invoke-RestMethod -Uri $dataSourcesUrl -Headers $headers

        foreach ($dataSource in $dataSources.value) {
            if ($dataSource.datasourceType -eq "AzureDataLakeStorage") {
                # Update connection details
                $updatePayload = @{
                    connectionDetails = @{
                        server = "onelake.dfs.fabric.microsoft.com"
                        database = $OneLakePath
                    }
                } | ConvertTo-Json -Depth 3

                $updateUrl = "https://api.powerbi.com/v1.0/myorg/groups/$WorkspaceId/datasets/$DatasetId/Default.UpdateDatasources"

                Invoke-RestMethod -Uri $updateUrl -Method POST -Headers $headers -Body $updatePayload

                Write-Host "    Data source updated to OneLake path: $OneLakePath"
                break
            }
        }

        # Refresh dataset to test connection
        $refreshUrl = "https://api.powerbi.com/v1.0/myorg/groups/$WorkspaceId/datasets/$DatasetId/refreshes"
        Invoke-RestMethod -Uri $refreshUrl -Method POST -Headers $headers

        Write-Host "    Dataset refresh initiated"
    }
    catch {
        Write-Warning "    Failed to update data source: $($_.Exception.Message)"
    }
}
```

### Deployment Pipeline Promotion

Automating the promotion of reports from Dev → Test → Prod:

```powershell
function Invoke-PipelineDeployment {
    param(
        [Parameter(Mandatory = $true)]
        [string]$PipelineId,

        [Parameter(Mandatory = $true)]
        [string]$SourceStage,  # 0=Dev, 1=Test, 2=Prod

        [Parameter(Mandatory = $true)]
        [string]$AccessToken
    )

    Write-Host "Promoting content from stage $SourceStage to next stage" -ForegroundColor Yellow

    $headers = @{
        "Authorization" = "Bearer $AccessToken"
        "Content-Type" = "application/json"
    }

    $deployPayload = @{
        sourceStageOrder = [int]$SourceStage
        options = @{
            allowCreateArtifact = $true
            allowOverwriteArtifact = $true
        }
    } | ConvertTo-Json -Depth 3

    try {
        $deployUrl = "https://api.powerbi.com/v1.0/myorg/pipelines/$PipelineId/deploy"
        $deployResponse = Invoke-RestMethod -Uri $deployUrl -Method POST -Headers $headers -Body $deployPayload

        Write-Host "  Deployment initiated (Operation ID: $($deployResponse.id))"

        # Monitor deployment status
        do {
            Start-Sleep -Seconds 10
            $statusUrl = "https://api.powerbi.com/v1.0/myorg/pipelines/$PipelineId/operations/$($deployResponse.id)"
            $status = Invoke-RestMethod -Uri $statusUrl -Headers $headers

            Write-Host "    Deployment status: $($status.status)"
        } while ($status.status -eq "InProgress")

        if ($status.status -eq "Succeeded") {
            Write-Host "  Deployment completed successfully"
        }
        else {
            Write-Error "  Deployment failed: $($status.error.message)"
        }

        return $status
    }
    catch {
        Write-Error "Failed to initiate deployment: $($_.Exception.Message)"
        throw
    }
}
```

## Complete Fabric Environment Setup

The main orchestration function that creates the entire Fabric environment:

```powershell
function Initialize-FabricEnvironment {
    param(
        [Parameter(Mandatory = $true)]
        [string]$ClientName,

        [Parameter(Mandatory = $true)]
        [string]$AccessToken,

        [Parameter(Mandatory = $true)]
        [string]$TemplatePbixPath
    )

    try {
        Write-Host "Starting complete Fabric environment setup for: $ClientName" -ForegroundColor Green

        # Step 1: Create workspaces
        $workspaces = New-FabricWorkspaces -ClientName $ClientName -AccessToken $AccessToken

        # Step 2: Create deployment pipeline
        $pipeline = New-FabricDeploymentPipeline -ClientName $ClientName -Workspaces $workspaces -AccessToken $AccessToken

        # Step 3: Create OneLake Lakehouses
        $lakehouses = New-OneLakeLakehouses -ClientName $ClientName -Workspaces $workspaces -AccessToken $AccessToken

        # Step 4: Deploy Power BI reports
        $reports = Deploy-PowerBIReports -ClientName $ClientName -Workspaces $workspaces -Lakehouses $lakehouses -AccessToken $AccessToken -TemplatePbixPath $TemplatePbixPath

        # Step 5: Promote to Test environment
        Start-Sleep -Seconds 30  # Allow Dev deployment to settle
        Invoke-PipelineDeployment -PipelineId $pipeline.PipelineId -SourceStage "0" -AccessToken $AccessToken

        # Step 6: Promote to Production environment
        Start-Sleep -Seconds 60  # Allow Test deployment to settle
        Invoke-PipelineDeployment -PipelineId $pipeline.PipelineId -SourceStage "1" -AccessToken $AccessToken

        Write-Host "Fabric environment setup completed successfully!" -ForegroundColor Green

        return @{
            Workspaces = $workspaces
            Pipeline = $pipeline
            Lakehouses = $lakehouses
            Reports = $reports
        }
    }
    catch {
        Write-Error "Fabric environment setup failed: $($_.Exception.Message)"
        throw
    }
}
```

## Workspace Access Control

Setting up security and access permissions for client workspaces:

```powershell
function Set-WorkspaceAccess {
    param(
        [Parameter(Mandatory = $true)]
        [array]$Workspaces,

        [Parameter(Mandatory = $true)]
        [string]$ClientUserGroup,  # Azure AD group for client users

        [Parameter(Mandatory = $true)]
        [string]$InternalDevGroup,  # Azure AD group for internal developers

        [Parameter(Mandatory = $true)]
        [string]$AccessToken
    )

    Write-Host "Setting up workspace access controls" -ForegroundColor Cyan

    $headers = @{
        "Authorization" = "Bearer $AccessToken"
        "Content-Type" = "application/json"
    }

    foreach ($workspace in $Workspaces) {
        Write-Host "  Configuring access for workspace: $($workspace.Name)"

        # Grant internal developers admin access to Dev/Test, Member access to Prod
        $devRole = if ($workspace.Environment -eq "Prod") { "Member" } else { "Admin" }

        $devAccessPayload = @{
            groupUserAccessRight = $devRole
            identifier = $InternalDevGroup
            principalType = "Group"
        } | ConvertTo-Json

        $addUserUrl = "https://api.powerbi.com/v1.0/myorg/groups/$($workspace.Id)/users"

        try {
            Invoke-RestMethod -Uri $addUserUrl -Method POST -Headers $headers -Body $devAccessPayload
            Write-Host "    Granted $devRole access to internal dev group"
        }
        catch {
            Write-Warning "    Failed to set dev group access: $($_.Exception.Message)"
        }

        # Grant client users Viewer access to Prod workspace only
        if ($workspace.Environment -eq "Prod") {
            $clientAccessPayload = @{
                groupUserAccessRight = "Viewer"
                identifier = $ClientUserGroup
                principalType = "Group"
            } | ConvertTo-Json

            try {
                Invoke-RestMethod -Uri $addUserUrl -Method POST -Headers $headers -Body $clientAccessPayload
                Write-Host "    Granted Viewer access to client user group"
            }
            catch {
                Write-Warning "    Failed to set client group access: $($_.Exception.Message)"
            }
        }

        Start-Sleep -Seconds 1  # Rate limiting
    }

    Write-Host "Workspace access controls configured successfully"
}
```

## Key Benefits

### Automated Environment Provisioning
- **Complete isolation** with separate workspaces per client
- **Consistent deployment pipeline** across all environments
- **Automated OneLake setup** with proper data lake structure
- **Ready-to-use Power BI reports** connected to client data

### Enterprise-Grade Release Management
- **Dev → Test → Prod pipeline** with automated promotion
- **Deployment history** and rollback capabilities
- **Environment-specific configurations** and data sources
- **Automated testing** and validation at each stage

### Security and Compliance
- **Workspace-level isolation** prevents cross-client data access
- **Role-based access control** with Azure AD integration
- **Automated permission management** reduces manual errors
- **Audit trail** for all deployments and access changes

### Scalability and Maintenance
- **Template-driven approach** enables rapid client onboarding
- **Consistent architecture** across all client environments
- **Automated deployment** reduces manual intervention
- **Centralized management** through deployment pipelines

## Integration with Data Pipeline

The Fabric environment integrates seamlessly with the Azure Data Factory pipelines:

### OneLake Connection Configuration

```json
{
  "name": "OneLakeSink",
  "properties": {
    "type": "AzureBlobFS",
    "linkedServiceName": {
      "referenceName": "OneLakeLinkedService",
      "type": "LinkedServiceReference"
    },
    "parameters": {
      "workspaceId": {
        "type": "string"
      },
      "lakehouseId": {
        "type": "string"
      },
      "folderPath": {
        "type": "string"
      }
    },
    "typeProperties": {
      "location": {
        "type": "AzureBlobFSLocation",
        "fileSystem": "@dataset().workspaceId",
        "folderPath": "@concat(dataset().lakehouseId, '.Lakehouse/Files/', dataset().folderPath)"
      },
      "format": {
        "type": "DeltalakeFormat"
      }
    }
  }
}
```

### Automated Data Refresh

Power BI datasets automatically refresh when new data arrives in OneLake, ensuring reports always show the latest client data.

## Next Steps

With the Microsoft Fabric environment automated, the final phase involves:

1. **[Complete Integration](PowerPages.md)** - End-to-end automation orchestration
2. **Security hardening** and compliance validation
3. **Monitoring and alerting** setup for production environments

This Fabric automation creates a complete, production-ready business intelligence environment that scales effortlessly across multiple clients while maintaining strict security and compliance standards.