# Azure Data Pipeline Deployment Instructions

## Overview
This deployment creates a secure Azure data processing pipeline following Zero Trust Architecture principles. The solution includes Azure Data Factory, SQL Database, Blob Storage, Key Vault, and monitoring components with private endpoints and comprehensive security controls.

## Prerequisites

### Required Tools
- Azure CLI (version 2.30.0 or later)
- Azure PowerShell (optional alternative)
- Sufficient Azure subscription permissions (Contributor or Owner)

### Required Information
- SQL Server administrator username and password
- List of IP ranges that need access to resources
- Resource group name (will be created if it doesn't exist)

## Zero Trust Architecture Implementation

This template implements Zero Trust principles through:

1. **Never Trust, Always Verify**
   - All resources use managed identities
   - Private endpoints for all data services
   - Network access controls and IP restrictions

2. **Least Privilege Access**
   - Role-based access control (RBAC) assignments
   - Minimal required permissions for each service
   - Disabled public network access where possible

3. **Assume Breach**
   - Comprehensive logging and monitoring
   - Data encryption at rest and in transit
   - Audit trails for all activities

## Deployment Steps

### Step 1: Prepare Environment

```bash
# Login to Azure
az login

# Set your subscription (replace with your subscription ID)
az account set --subscription "your-subscription-id"

# Create resource group (replace with your preferred name and location)
az group create --name "rg-datapipeline-prod" --location "East US"
```

### Step 2: Prepare Parameters

Create a `parameters.json` file:

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "environmentPrefix": {
      "value": "dp-prod"
    },
    "sqlAdministratorLogin": {
      "value": "sqladmin"
    },
    "sqlAdministratorPassword": {
      "value": "YourSecurePassword123!"
    },
    "allowedIpRanges": {
      "value": [
        {
          "value": "203.0.113.0/24"
        },
        {
          "value": "198.51.100.0/24"
        }
      ]
    }
  }
}
```

### Step 3: Deploy the Template

```bash
# Deploy using Azure CLI
az deployment group create \
  --resource-group "rg-datapipeline-prod" \
  --template-file "azuredeploy.json" \
  --parameters "@parameters.json" \
  --verbose
```

Alternative PowerShell deployment:
```powershell
New-AzResourceGroupDeployment `
  -ResourceGroupName "rg-datapipeline-prod" `
  -TemplateFile "azuredeploy.json" `
  -TemplateParameterFile "parameters.json" `
  -Verbose
```

### Step 4: Post-Deployment Configuration

#### Configure Key Vault Access Policies

```bash
# Get the Data Factory managed identity from deployment outputs
DATA_FACTORY_PRINCIPAL_ID=$(az deployment group show \
  --resource-group "rg-datapipeline-prod" \
  --name "azuredeploy" \
  --query "properties.outputs.dataFactoryManagedIdentityPrincipalId.value" \
  --output tsv)

# Get Key Vault name from outputs
KEY_VAULT_NAME=$(az deployment group show \
  --resource-group "rg-datapipeline-prod" \
  --name "azuredeploy" \
  --query "properties.outputs.keyVaultName.value" \
  --output tsv)

# Grant Data Factory access to Key Vault
az keyvault set-policy \
  --name $KEY_VAULT_NAME \
  --object-id $DATA_FACTORY_PRINCIPAL_ID \
  --secret-permissions get list
```

#### Store SFTP Credentials in Key Vault

```bash
# Store SFTP server credentials (replace with actual values)
az keyvault secret set \
  --vault-name $KEY_VAULT_NAME \
  --name "sftp-username" \
  --value "your-sftp-username"

az keyvault secret set \
  --vault-name $KEY_VAULT_NAME \
  --name "sftp-password" \
  --value "your-sftp-password"

az keyvault secret set \
  --vault-name $KEY_VAULT_NAME \
  --name "sftp-host" \
  --value "your-sftp-server.com"
```

### Step 5: Configure Data Factory Pipelines

After deployment, you'll need to create the actual data pipelines in Azure Data Factory:

1. **Access Data Factory Studio**
   - Navigate to the Data Factory resource in Azure portal
   - Click "Launch Studio"

2. **Create Linked Services**
   - SFTP server connection (using Key Vault for credentials)
   - Azure Blob Storage connection
   - Azure SQL Database connection

3. **Create Datasets**
   - Source: SFTP CSV files
   - Sink: Blob Storage and SQL Database

4. **Build Pipelines**
   - Copy activity from SFTP to Blob Storage
   - Data Flow for transformation
   - Copy activity from Blob to SQL Database

### Step 6: Configure Monitoring

#### Set up Alerts
```bash
# Create action group for notifications
az monitor action-group create \
  --resource-group "rg-datapipeline-prod" \
  --name "DataPipelineAlerts" \
  --short-name "DPAlerts" \
  --email-receiver name="admin" email="admin@yourcompany.com"

# Create pipeline failure alert
az monitor metrics alert create \
  --resource-group "rg-datapipeline-prod" \
  --name "Pipeline-Failure-Alert" \
  --description "Alert when Data Factory pipeline fails" \
  --severity 2 \
  --condition "count 'static' > 0" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --action-group "DataPipelineAlerts"
```

## Security Validation Checklist

After deployment, verify these security configurations:

### Network Security
- [ ] All resources have private endpoints configured
- [ ] Public network access is disabled for storage, SQL, and Key Vault
- [ ] Network Security Groups are properly configured
- [ ] IP restrictions are in place where needed

### Identity and Access
- [ ] All services use managed identities
- [ ] RBAC assignments follow least privilege principle
- [ ] Key Vault access policies are properly configured
- [ ] No service accounts with passwords are used

### Data Protection
- [ ] Storage account has encryption at rest enabled
- [ ] SQL Database has Transparent Data Encryption enabled
- [ ] All connections use TLS 1.2 or higher
- [ ] Shared access keys are disabled where possible

### Monitoring and Auditing
- [ ] Diagnostic settings are configured for all resources
- [ ] Log Analytics workspace is collecting logs
- [ ] Audit logging is enabled for SQL Database
- [ ] Key Vault audit events are being captured
- [ ] Data Factory pipeline monitoring is active

### Compliance
- [ ] Retention policies are set (90 days default)
- [ ] Soft delete is enabled for Key Vault and Storage
- [ ] Backup policies are configured
- [ ] Data classification is implemented where required

## Troubleshooting Common Issues

### Issue: Private Endpoint DNS Resolution
**Problem**: Services can't connect through private endpoints
**Solution**: 
```bash
# Configure Private DNS zones
az network private-dns zone create \
  --resource-group "rg-datapipeline-prod" \
  --name "privatelink.blob.core.windows.net"

az network private-dns zone create \
  --resource-group "rg-datapipeline-prod" \
  --name "privatelink.database.windows.net"

az network private-dns zone create \
  --resource-group "rg-datapipeline-prod" \
  --name "privatelink.vaultcore.azure.net"

# Link DNS zones to VNet
VNET_ID=$(az network vnet show \
  --resource-group "rg-datapipeline-prod" \
  --name "dp-prod-vnet" \
  --query "id" --output tsv)

az network private-dns link vnet create \
  --resource-group "rg-datapipeline-prod" \
  --zone-name "privatelink.blob.core.windows.net" \
  --name "blob-dns-link" \
  --virtual-network $VNET_ID \
  --registration-enabled false
```

### Issue: Data Factory Can't Access Resources
**Problem**: Data Factory managed identity lacks permissions
**Solution**:
```bash
# Grant additional permissions if needed
az role assignment create \
  --assignee $DATA_FACTORY_PRINCIPAL_ID \
  --role "SQL DB Contributor" \
  --scope "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/rg-datapipeline-prod"
```

### Issue: SFTP Connection Failures
**Problem**: Can't connect to external SFTP server
**Solution**:
1. Verify SFTP server allows connections from Azure Data Factory IP ranges
2. Check if firewall rules are blocking outbound connections
3. Ensure credentials in Key Vault are correct

## Power BI Integration

### Step 1: Create Power BI Workspace
```bash
# Install Power BI CLI (if not already installed)
pip install pbi-cli

# Create workspace (requires Power BI Pro license)
pbi workspace create --name "DataPipeline-Analytics"
```

### Step 2: Configure SQL Database Connection
1. Open Power BI Desktop
2. Get Data > Azure > Azure SQL Database
3. Use the SQL Server name from deployment outputs
4. Configure gateway for on-premises data refresh (if needed)

### Step 3: Create Reports and Dashboards
- Build reports based on processed data tables
- Set up automated refresh schedules
- Configure row-level security if needed

## Automation Scripts

### Daily Health Check Script
```bash
#!/bin/bash
# daily-health-check.sh

RESOURCE_GROUP="rg-datapipeline-prod"
STORAGE_ACCOUNT=$(az deployment group show --resource-group $RESOURCE_GROUP --name "azuredeploy" --query "properties.outputs.storageAccountName.value" -o tsv)
DATA_FACTORY=$(az deployment group show --resource-group $RESOURCE_GROUP --name "azuredeploy" --query "properties.outputs.dataFactoryName.value" -o tsv)

echo "=== Daily Health Check Report ==="
echo "Date: $(date)"
echo

# Check storage account health
echo "Storage Account Status:"
az storage account show --name $STORAGE_ACCOUNT --resource-group $RESOURCE_GROUP --query "statusOfPrimary" -o tsv

# Check Data Factory status
echo "Data Factory Status:"
az datafactory show --name $DATA_FACTORY --resource-group $RESOURCE_GROUP --query "provisioningState" -o tsv

# Check recent pipeline runs
echo "Recent Pipeline Runs (Last 24 hours):"
az datafactory pipeline-run query-by-factory \
  --factory-name $DATA_FACTORY \
  --resource-group $RESOURCE_GROUP \
  --last-updated-after $(date -d '1 day ago' -u +%Y-%m-%dT%H:%M:%S.%3NZ) \
  --query "value[].{PipelineName:pipelineName,Status:status,RunStart:runStart}" \
  --output table

echo "=== End Health Check ==="
```

### Automated Backup Script
```bash
#!/bin/bash
# backup-configurations.sh

RESOURCE_GROUP="rg-datapipeline-prod"
BACKUP_DATE=$(date +%Y%m%d)
BACKUP_DIR="./backups/$BACKUP_DATE"

mkdir -p $BACKUP_DIR

# Export Data Factory configurations
echo "Backing up Data Factory configurations..."
DATA_FACTORY=$(az deployment group show --resource-group $RESOURCE_GROUP --name "azuredeploy" --query "properties.outputs.dataFactoryName.value" -o tsv)

# Export pipelines
az datafactory pipeline list --factory-name $DATA_FACTORY --resource-group $RESOURCE_GROUP > "$BACKUP_DIR/pipelines.json"

# Export datasets
az datafactory dataset list --factory-name $DATA_FACTORY --resource-group $RESOURCE_GROUP > "$BACKUP_DIR/datasets.json"

# Export linked services
az datafactory linked-service list --factory-name $DATA_FACTORY --resource-group $RESOURCE_GROUP > "$BACKUP_DIR/linked-services.json"

# Backup SQL Database schema
SQL_SERVER=$(az deployment group show --resource-group $RESOURCE_GROUP --name "azuredeploy" --query "properties.outputs.sqlServerName.value" -o tsv)
SQL_DATABASE=$(az deployment group show --resource-group $RESOURCE_GROUP --name "azuredeploy" --query "properties.outputs.sqlDatabaseName.value" -o tsv)

echo "Backing up SQL Database schema..."
# Note: This requires sqlcmd to be installed and configured
# sqlcmd -S $SQL_SERVER.database.windows.net -d $SQL_DATABASE -E -Q "SELECT * FROM INFORMATION_SCHEMA.TABLES" > "$BACKUP_DIR/db-schema.txt"

echo "Backup completed: $BACKUP_DIR"
```

## Performance Optimization

### Data Factory Optimization
1. **Use Copy Activity Optimization**
   - Enable parallel copying for large files
   - Use appropriate data integration units (DIUs)
   - Implement incremental data loading

2. **Pipeline Design Best Practices**
   - Use pipeline parameters for reusability
   - Implement proper error handling
   - Use pipeline activities efficiently

### SQL Database Optimization
```sql
-- Create indexes for frequently queried columns
CREATE INDEX IX_ProcessedData_Date ON ProcessedData(ProcessedDate);
CREATE INDEX IX_ProcessedData_Source ON ProcessedData(SourceSystem);

-- Update statistics regularly
UPDATE STATISTICS ProcessedData;

-- Monitor query performance
SELECT 
    query_stats.query_id,
    query_stats.avg_duration,
    query_stats.avg_cpu_time,
    query_text.query_sql_text
FROM sys.query_store_query_stats query_stats
JOIN sys.query_store_query_text query_text 
    ON query_stats.query_id = query_text.query_id
ORDER BY query_stats.avg_duration DESC;
```

### Storage Optimization
```bash
# Configure lifecycle management for blob storage
STORAGE_ACCOUNT=$(az deployment group show --resource-group "rg-datapipeline-prod" --name "azuredeploy" --query "properties.outputs.storageAccountName.value" -o tsv)

# Create lifecycle policy
cat > lifecycle-policy.json << EOF
{
  "rules": [
    {
      "name": "ArchiveOldFiles",
      "enabled": true,
      "type": "Lifecycle",
      "definition": {
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["raw-files/"]
        },
        "actions": {
          "baseBlob": {
            "tierToCool": {"daysAfterModificationGreaterThan": 30},
            "tierToArchive": {"daysAfterModificationGreaterThan": 90},
            "delete": {"daysAfterModificationGreaterThan": 365}
          }
        }
      }
    }
  ]
}
EOF

# Apply lifecycle policy
az storage account management-policy create \
  --account-name $STORAGE_ACCOUNT \
  --resource-group "rg-datapipeline-prod" \
  --policy @lifecycle-policy.json
```

## Cost Optimization

### Monitoring Costs
```bash
# Set up budget alerts
az consumption budget create \
  --resource-group "rg-datapipeline-prod" \
  --budget-name "DataPipeline-Monthly-Budget" \
  --amount 1000 \
  --time-grain Monthly \
  --start-date $(date +%Y-%m-01) \
  --end-date $(date -d '+1 year' +%Y-%m-01) \
  --notifications threshold=80 operator=GreaterThan \
  --contact-emails "admin@yourcompany.com"
```

### Cost Optimization Recommendations
1. **Data Factory**: Use triggers efficiently, avoid unnecessary pipeline runs
2. **SQL Database**: Consider DTU vs vCore pricing, implement auto-pause for dev environments
3. **Storage**: Use appropriate access tiers, implement lifecycle policies
4. **Log Analytics**: Configure appropriate retention periods, use basic logs where possible

## Disaster Recovery

### Backup Strategy
1. **Automated Backups**
   - SQL Database: Automated backups (7-35 days retention)
   - Storage Account: Geo-redundant storage with soft delete
   - Key Vault: Backup keys and secrets to secondary region

2. **Recovery Procedures**
   ```bash
   # Restore SQL Database from backup
   az sql db restore \
     --dest-name "restored-database" \
     --edition Standard \
     --name $SQL_DATABASE \
     --resource-group $RESOURCE_GROUP \
     --server $SQL_SERVER \
     --time "2024-01-01T00:00:00Z"
   ```

### Business Continuity Plan
1. Document all external dependencies (SFTP servers, data sources)
2. Maintain configuration backups
3. Test disaster recovery procedures quarterly
4. Document recovery time objectives (RTO) and recovery point objectives (RPO)

## Support and Maintenance

### Regular Maintenance Tasks
- [ ] Weekly: Review pipeline execution logs
- [ ] Monthly: Update and patch all components
- [ ] Quarterly: Review and update security configurations
- [ ] Annually: Conduct disaster recovery testing

### Monitoring Dashboard
Create a comprehensive monitoring dashboard in Azure Monitor with:
- Pipeline success/failure rates
- Data processing volumes
- System performance metrics
- Security alerts and violations
- Cost trends

### Contact Information
For support and escalation:
- Azure Support: [Azure Support Portal](https://portal.azure.com/#blade/Microsoft_Azure_Support/HelpAndSupportBlade)
- Internal IT Team: your-it-team@company.com
- Data Factory Documentation: [Microsoft Docs](https://docs.microsoft.com/en-us/azure/data-factory/)

---

**Note**: This deployment template provides a solid foundation following Zero Trust principles. Additional customization may be required based on specific business requirements and compliance needs.