# Deployment Guide

## Overview
This guide provides step-by-step instructions for deploying the Oracle 26ai Database Integration with Microsoft Copilot and Azure Services.

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [PowerShell Deployment Script](#powershell-deployment-script)
3. [ARM Template](#arm-template)
4. [Manual Deployment Steps](#manual-deployment-steps)
5. [Configuration Files](#configuration-files)
6. [Testing and Validation](#testing-and-validation)

---

## Prerequisites

### Required Software
- Azure CLI 2.50+
- PowerShell 7+
- Oracle Database 26ai
- Node.js 18+ or .NET 6+
- Git

### Required Permissions
- Azure Subscription Contributor
- Oracle DBA privileges
- GitHub repository access

---

## PowerShell Deployment Script

### Quick Deployment

Save this as `Deploy-OracleAzureIntegration.ps1`:

```powershell
param(
    [Parameter(Mandatory=$true)]
    [string]$ResourceGroupName,
    
    [Parameter(Mandatory=$true)]
    [string]$Location,
    
    [Parameter(Mandatory=$true)]
    [string]$OracleConnectionString,
    
    [Parameter(Mandatory=$true)]
    [string]$SubscriptionId
)

$ErrorActionPreference = "Stop"

Write-Host "Oracle-Azure Integration Deployment" -ForegroundColor Cyan

# 1. Login to Azure
Connect-AzAccount -SubscriptionId $SubscriptionId

# 2. Create Resource Group
New-AzResourceGroup -Name $ResourceGroupName -Location $Location -Force

# 3. Create Key Vault
$keyVaultName = "kv-oracle-" + (Get-Random -Maximum 9999)
$keyVault = New-AzKeyVault `
    -Name $keyVaultName `
    -ResourceGroupName $ResourceGroupName `
    -Location $Location `
    -EnableSoftDelete `
    -EnablePurgeProtection

# Store Oracle connection string
$secureString = ConvertTo-SecureString $OracleConnectionString -AsPlainText -Force
Set-AzKeyVaultSecret `
    -VaultName $keyVaultName `
    -Name "oracle-connection-string" `
    -SecretValue $secureString

# 4. Create Storage Account
$storageAccountName = "storacleint" + (Get-Random -Maximum 9999)
New-AzStorageAccount `
    -ResourceGroupName $ResourceGroupName `
    -Name $storageAccountName `
    -Location $Location `
    -SkuName "Standard_LRS" `
    -Kind "StorageV2"

# 5. Create Log Analytics Workspace
$workspaceName = "law-oracle-integration"
$workspace = New-AzOperationalInsightsWorkspace `
    -ResourceGroupName $ResourceGroupName `
    -Name $workspaceName `
    -Location $Location `
    -Sku "PerGB2018"

# 6. Create Application Insights
$appInsightsName = "ai-oracle-integration"
$appInsights = New-AzApplicationInsights `
    -ResourceGroupName $ResourceGroupName `
    -Name $appInsightsName `
    -Location $Location `
    -WorkspaceResourceId $workspace.ResourceId

# 7. Create Function App
$functionAppName = "func-oracle-api-" + (Get-Random -Maximum 9999)
$functionApp = New-AzFunctionApp `
    -ResourceGroupName $ResourceGroupName `
    -Name $functionAppName `
    -Location $Location `
    -StorageAccountName $storageAccountName `
    -Runtime "node" `
    -RuntimeVersion "18" `
    -FunctionsVersion "4"

# 8. Enable Managed Identity
Update-AzFunctionApp `
    -ResourceGroupName $ResourceGroupName `
    -Name $functionAppName `
    -IdentityType "SystemAssigned"

# 9. Grant Key Vault Access
$functionAppIdentity = (Get-AzFunctionApp `
    -ResourceGroupName $ResourceGroupName `
    -Name $functionAppName).IdentityPrincipalId

Set-AzKeyVaultAccessPolicy `
    -VaultName $keyVaultName `
    -ObjectId $functionAppIdentity `
    -PermissionsToSecrets Get,List

# 10. Configure App Settings
$appSettings = @{
    "APPINSIGHTS_INSTRUMENTATIONKEY" = $appInsights.InstrumentationKey
    "KEY_VAULT_URL" = $keyVault.VaultUri
    "OracleConnectionString" = "@Microsoft.KeyVault(SecretUri=$($keyVault.VaultUri)secrets/oracle-connection-string/)"
}

Update-AzFunctionAppSetting `
    -ResourceGroupName $ResourceGroupName `
    -Name $functionAppName `
    -AppSetting $appSettings

Write-Host "✓ Deployment completed successfully!" -ForegroundColor Green
Write-Host "Function App: $functionAppName" -ForegroundColor White
Write-Host "Key Vault: $keyVaultName" -ForegroundColor White
```

### Run the Deployment

```powershell
.\Deploy-OracleAzureIntegration.ps1 `
    -ResourceGroupName "rg-oracle-integration" `
    -Location "eastus" `
    -OracleConnectionString "your-connection-string" `
    -SubscriptionId "your-subscription-id"
```

---

## ARM Template

### azuredeploy.json

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "location": {
      "type": "string",
      "defaultValue": "[resourceGroup().location]"
    },
    "oracleConnectionString": {
      "type": "securestring"
    }
  },
  "variables": {
    "keyVaultName": "[concat('kv-oracle-', uniqueString(resourceGroup().id))]",
    "storageAccountName": "[concat('storacleint', uniqueString(resourceGroup().id))]",
    "functionAppName": "[concat('func-oracle-api-', uniqueString(resourceGroup().id))]",
    "appInsightsName": "ai-oracle-integration"
  },
  "resources": [
    {
      "type": "Microsoft.KeyVault/vaults",
      "apiVersion": "2022-07-01",
      "name": "[variables('keyVaultName')]",
      "location": "[parameters('location')]",
      "properties": {
        "sku": {
          "family": "A",
          "name": "standard"
        },
        "tenantId": "[subscription().tenantId]",
        "enableSoftDelete": true,
        "enablePurgeProtection": true
      }
    },
    {
      "type": "Microsoft.Storage/storageAccounts",
      "apiVersion": "2022-09-01",
      "name": "[variables('storageAccountName')]",
      "location": "[parameters('location')]",
      "sku": {
        "name": "Standard_LRS"
      },
      "kind": "StorageV2"
    },
    {
      "type": "Microsoft.Web/sites",
      "apiVersion": "2022-03-01",
      "name": "[variables('functionAppName')]",
      "location": "[parameters('location')]",
      "kind": "functionapp",
      "identity": {
        "type": "SystemAssigned"
      },
      "properties": {
        "siteConfig": {
          "appSettings": [
            {
              "name": "FUNCTIONS_EXTENSION_VERSION",
              "value": "~4"
            },
            {
              "name": "FUNCTIONS_WORKER_RUNTIME",
              "value": "node"
            }
          ]
        }
      }
    }
  ]
}
```

### Deploy ARM Template

```bash
az deployment group create \
  --resource-group rg-oracle-integration \
  --template-file azuredeploy.json \
  --parameters oracleConnectionString="your-connection-string"
```

---

## Manual Deployment Steps

### Step 1: Setup Oracle Database

```sql
-- Connect as SYSDBA
CONNECT sys@ORCLPDB AS SYSDBA

-- Create integration user
CREATE USER azure_integration IDENTIFIED BY "SecurePassword123!";

GRANT CREATE SESSION TO azure_integration;
GRANT CREATE TABLE TO azure_integration;
GRANT UNLIMITED TABLESPACE TO azure_integration;

-- Enable cloud integration
@?/rdbms/admin/dbms_cloud_install.sql

-- Create credential for Azure
BEGIN
  DBMS_CLOUD.CREATE_CREDENTIAL(
    credential_name => 'AZURE_STORAGE_CRED',
    username => 'azure_storage_account_name',
    password => 'your_access_key'
  );
END;
/
```

### Step 2: Setup Azure Resources

```bash
# Create resource group
az group create \
  --name rg-oracle-integration \
  --location eastus

# Create virtual network
az network vnet create \
  --resource-group rg-oracle-integration \
  --name vnet-oracle \
  --address-prefix 10.0.0.0/16 \
  --subnet-name subnet-database \
  --subnet-prefix 10.0.1.0/24

# Create NSG
az network nsg create \
  --resource-group rg-oracle-integration \
  --name nsg-database-subnet

# Add Oracle rule
az network nsg rule create \
  --resource-group rg-oracle-integration \
  --nsg-name nsg-database-subnet \
  --name AllowOracle \
  --priority 100 \
  --source-address-prefixes "10.0.0.0/16" \
  --destination-port-ranges 1521 \
  --access Allow \
  --protocol Tcp
```

### Step 3: Deploy Function App Code

```bash
# Clone the repository
git clone https://github.com/hvrcharon1/oracle-26ai-azure-copilot-integration.git
cd oracle-26ai-azure-copilot-integration

# Install dependencies
npm install

# Deploy to Azure
func azure functionapp publish func-oracle-api
```

---

## Configuration Files

### host.json

```json
{
  "version": "2.0",
  "logging": {
    "applicationInsights": {
      "samplingSettings": {
        "isEnabled": true,
        "maxTelemetryItemsPerSecond": 20
      }
    }
  },
  "extensions": {
    "http": {
      "routePrefix": "api",
      "maxOutstandingRequests": 200,
      "maxConcurrentRequests": 100
    }
  },
  "functionTimeout": "00:05:00"
}
```

### package.json

```json
{
  "name": "oracle-azure-functions",
  "version": "1.0.0",
  "dependencies": {
    "@azure/identity": "^3.3.2",
    "@azure/keyvault-secrets": "^4.7.0",
    "@azure/openai": "^1.0.0-beta.8",
    "applicationinsights": "^2.7.3",
    "oracledb": "^6.0.3"
  }
}
```

### local.settings.json

```json
{
  "IsEncrypted": false,
  "Values": {
    "AzureWebJobsStorage": "UseDevelopmentStorage=true",
    "FUNCTIONS_WORKER_RUNTIME": "node",
    "OracleConnectionString": "your-connection-string",
    "KEY_VAULT_URL": "https://your-keyvault.vault.azure.net/",
    "APPINSIGHTS_INSTRUMENTATIONKEY": "your-key"
  }
}
```

---

## Testing and Validation

### Health Check Test

```bash
# Test the health endpoint
curl https://func-oracle-api.azurewebsites.net/api/health

# Expected response:
# {
#   "timestamp": "2025-11-07T10:30:00Z",
#   "status": "healthy",
#   "checks": {
#     "oracle": { "status": "healthy" }
#   }
# }
```

### Integration Test Script

```bash
#!/bin/bash
# integration-tests.sh

API_BASE_URL="https://func-oracle-api.azurewebsites.net/api"

echo "Test 1: Health Check"
curl -s "$API_BASE_URL/health" | jq '.status'

echo "Test 2: Simple Query"
curl -s -X POST "$API_BASE_URL/query" \
  -H "Content-Type: application/json" \
  -d '{"query":"SELECT 1 FROM DUAL"}' | jq '.success'

echo "All tests completed!"
```

---

## Next Steps

1. ✅ Deploy infrastructure
2. ✅ Configure Oracle database
3. ✅ Deploy Function App code
4. ✅ Test connectivity
5. 📝 Configure monitoring alerts
6. 📝 Setup CI/CD pipeline
7. 📝 Configure backup strategy

---

## Support

For deployment issues:
- Review logs in Azure Portal
- Check Application Insights for errors
- Refer to [Troubleshooting Guide](troubleshooting-guide.md)
- Contact: support@yourcompany.com
