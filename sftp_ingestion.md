Comprehensive solution architecture and a corresponding Azure Resource Manager (ARM) template to build the foundation for your data acquisition pipeline.

This solution is designed with your specific requirements in mind:
*   **Cost-Optimized:** Utilizes serverless and consumption-based services where possible, minimizing idle costs.
*   **Scalable:** Each component can scale independently to handle growth in data volume or frequency.
*   **Reporting-Ready:** Lands the data in a relational, queryable format in Azure SQL, perfect for downstream reporting tools like Power BI.

---

### 1. Solution Architecture

The architecture uses a modern, metadata-driven approach, separating the infrastructure (ARM Template) from the data pipeline logic (ADF artifacts).

#### Architectural Diagram

```
+----------------+      +--------------------------+      +---------------------+      +-------------------+
| Q2 SFTP Server |      |   Self-Hosted IR (VM)    |      | Azure Data Factory  |      |   Azure Key Vault |
| (On-Prem/VNet) | <--- | (Secure Data Channel)    | <--- | (Orchestration)     | <--- | (Stores Secrets)  |
+----------------+      +--------------------------+      +----------+----------+      +-------------------+
                                                                     |
                                                               (Triggers Daily)
                                                                     |
+--------------------------------------------------------------------+--------------------------------------------------------------------+
|                                                                                                                                       |
| Pipeline Step 1: Copy CSV from SFTP                                                                                                   |
|    - Uses SFTP Linked Service (via Self-Hosted IR)                                                                                    |
|    - Copies file to Azure Blob Storage                                                                                                |
|                                                                                                                                       |
v                                                                                                                                       v
+----------------------------------+                                                                   +--------------------------------+
|      Azure Blob Storage          |                                                                   |      Azure Monitor             |
|                                  |                                                                   |                                |
|  /raw-data/YYYY/MM/DD/file.csv   |                                                                   |  - Pipeline Run Monitoring     |
|  (Initial Landing Zone)          |                                                                   |  - Logging & Diagnostics       |
|                                  |                                                                   |  - Alerting on Failures        |
|  /processed-data/YYYY/MM/DD/..   |                                                                   |                                |
|  (Archive after processing)      |                                                                   |                                |
+----------------+-----------------+                                                                   +--------------------------------+
                 |
                 | (File path passed as parameter)
                 v
|--------------------------------------------------------------------+
| Pipeline Step 2: Ingest Data into SQL                               |
|    - Calls a Stored Procedure in Azure SQL DB                       |
|    - The Stored Procedure reads the file from Blob Storage and      |
|      performs a MERGE operation to update the target table.         |
|                                                                     |
v                                                                     v
+----------------------------------+                             +--------------------------------+
|      Azure SQL Database          |                             |      Power BI / Reporting      |
|                                  |                             |                                |
|  - Staging Table (temp)          | <---------------------------+  - Connects to SQL DB          |
|  - Production Table              |                             |  - Creates Reports/Dashboards  |
|  - Stored Procedure (for MERGE)  |                             |                                |
+----------------------------------+                             +--------------------------------+

```

#### Component Breakdown

1.  **Q2 SFTP Server:** Your existing source system. We assume it's in a private network or on-premises, necessitating a secure connection.

2.  **Self-Hosted Integration Runtime (SHIR):** A small Windows Virtual Machine (e.g., a cost-effective B-series VM) running the SHIR agent. This agent acts as a secure data gateway, allowing Azure Data Factory to connect to your private SFTP server without exposing it to the public internet.

3.  **Azure Key Vault:** The central, secure repository for all secrets. We will store the SFTP credentials and the SQL Database connection string here. Azure Data Factory will access these using its **Managed Identity**, eliminating the need to store credentials in code or ADF configurations.

4.  **Azure Data Factory (ADF):** The core orchestrator of the entire process.
    *   **Linked Services:** Connections to data stores (SFTP, Blob, SQL DB) using credentials from Key Vault.
    *   **Datasets:** Pointers to the specific data (e.g., the CSV files in Blob, the table in SQL).
    *   **Pipeline:** The sequence of activities.
        *   **Schedule Trigger:** Kicks off the pipeline daily.
        *   **Copy Data Activity:** Moves the CSV file from SFTP (via SHIR) to a "raw-data" container in Blob Storage.
        *   **Stored Procedure Activity:** Executes a stored procedure in the Azure SQL Database, passing the file path of the newly landed CSV as a parameter.
        *   **(Optional) Delete/Move Activity:** Cleans up by moving the processed file to an "archive" container.

5.  **Azure Blob Storage:** A cost-effective staging area.
    *   **`raw-data` container:** Where files land directly from the SFTP. Files are organized by date (`YYYY/MM/DD/`) for easy partitioning and historical lookups.
    *   **`processed-data` container:** (Optional but recommended) Where files are moved after successful ingestion into the SQL DB, serving as a cheap archive.

6.  **Azure SQL Database:** The final destination for the data.
    *   We use a **stored procedure** to handle the data loading. This is more efficient and powerful than doing the transformations in ADF's Data Flow for this use case. The procedure can use a `MERGE` statement to efficiently handle inserts, updates, and even deletes, fulfilling your "update the tables" requirement.
    *   Using the **Serverless tier** is highly recommended for cost savings if the daily load is the primary workload and the database is otherwise idle.

7.  **Azure Monitor:** Provides out-of-the-box monitoring for all ADF pipeline runs. You can set up alerts to be notified via email or SMS if a pipeline fails.

8.  **Power BI (or other reporting tool):** Connects directly to the Azure SQL Database to provide rich, interactive reporting and analytics on the up-to-date data.

---

### 2. Deployment Azure Resource Manager (ARM) Template

This ARM template will create the core Azure infrastructure. **It will NOT create the ADF artifacts (pipelines, datasets) or the SQL schema**, as those are typically managed separately.

**What this template creates:**
*   Azure Data Factory v2 with a System-Assigned Managed Identity.
*   Azure Storage Account with `raw-data` and `processed-data` containers.
*   Azure Key Vault.
*   Azure SQL Server.
*   Azure SQL Database (defaults to a cost-effective Serverless SKU).
*   A firewall rule on the SQL Server to allow access from Azure services.
*   An access policy in Key Vault granting the new Data Factory permission to get secrets.

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "prefix": {
            "type": "string",
            "metadata": {
                "description": "A unique prefix for all resource names (e.g., 'q2dataload')."
            }
        },
        "location": {
            "type": "string",
            "defaultValue": "[resourceGroup().location]",
            "metadata": {
                "description": "The Azure region for all resources."
            }
        },
        "sqlAdminLogin": {
            "type": "string",
            "metadata": {
                "description": "The administrator login for the SQL Server."
            }
        },
        "sqlAdminPassword": {
            "type": "securestring",
            "metadata": {
                "description": "The administrator password for the SQL Server."
            }
        }
    },
    "variables": {
        "storageAccountName": "[concat(parameters('prefix'), 'sa', uniqueString(resourceGroup().id))]",
        "keyVaultName": "[concat(parameters('prefix'), '-kv-', uniqueString(resourceGroup().id))]",
        "dataFactoryName": "[concat(parameters('prefix'), '-adf-', uniqueString(resourceGroup().id))]",
        "sqlServerName": "[concat(parameters('prefix'), '-sql-', uniqueString(resourceGroup().id))]",
        "sqlDatabaseName": "[concat(parameters('prefix'), '-sqldb')]"
    },
    "resources": [
        {
            "type": "Microsoft.Storage/storageAccounts",
            "apiVersion": "2021-09-01",
            "name": "[variables('storageAccountName')]",
            "location": "[parameters('location')]",
            "sku": {
                "name": "Standard_LRS"
            },
            "kind": "StorageV2",
            "properties": {
                "allowBlobPublicAccess": false
            }
        },
        {
            "type": "Microsoft.Storage/storageAccounts/blobServices/containers",
            "apiVersion": "2021-09-01",
            "name": "[concat(variables('storageAccountName'), '/default/raw-data')]",
            "dependsOn": [
                "[resourceId('Microsoft.Storage/storageAccounts', variables('storageAccountName'))]"
            ]
        },
        {
            "type": "Microsoft.Storage/storageAccounts/blobServices/containers",
            "apiVersion": "2021-09-01",
            "name": "[concat(variables('storageAccountName'), '/default/processed-data')]",
            "dependsOn": [
                "[resourceId('Microsoft.Storage/storageAccounts', variables('storageAccountName'))]"
            ]
        },
        {
            "type": "Microsoft.DataFactory/factories",
            "apiVersion": "2018-06-01",
            "name": "[variables('dataFactoryName')]",
            "location": "[parameters('location')]",
            "identity": {
                "type": "SystemAssigned"
            }
        },
        {
            "type": "Microsoft.KeyVault/vaults",
            "apiVersion": "2021-10-01",
            "name": "[variables('keyVaultName')]",
            "location": "[parameters('location')]",
            "properties": {
                "sku": {
                    "family": "A",
                    "name": "standard"
                },
                "tenantId": "[subscription().tenantId]",
                "accessPolicies": [
                    {
                        "tenantId": "[subscription().tenantId]",
                        "objectId": "[reference(resourceId('Microsoft.DataFactory/factories', variables('dataFactoryName')), '2018-06-01', 'Full').identity.principalId]",
                        "permissions": {
                            "secrets": [
                                "get",
                                "list"
                            ]
                        }
                    }
                ]
            },
            "dependsOn": [
                "[resourceId('Microsoft.DataFactory/factories', variables('dataFactoryName'))]"
            ]
        },
        {
            "type": "Microsoft.Sql/servers",
            "apiVersion": "2021-11-01",
            "name": "[variables('sqlServerName')]",
            "location": "[parameters('location')]",
            "properties": {
                "administratorLogin": "[parameters('sqlAdminLogin')]",
                "administratorLoginPassword": "[parameters('sqlAdminPassword')]"
            }
        },
        {
            "type": "Microsoft.Sql/servers/databases",
            "apiVersion": "2021-11-01",
            "name": "[concat(variables('sqlServerName'), '/', variables('sqlDatabaseName'))]",
            "location": "[parameters('location')]",
            "sku": {
                "name": "GP_S_Gen5",
                "tier": "GeneralPurpose",
                "family": "Gen5",
                "capacity": 1
            },
            "properties": {
                "autoPauseDelay": 60,
                "minCapacity": 0.5
            },
            "dependsOn": [
                "[resourceId('Microsoft.Sql/servers', variables('sqlServerName'))]"
            ]
        },
        {
            "type": "Microsoft.Sql/servers/firewallRules",
            "apiVersion": "2021-11-01",
            "name": "[concat(variables('sqlServerName'), '/AllowAllWindowsAzureIps')]",
            "location": "[parameters('location')]",
            "properties": {
                "startIpAddress": "0.0.0.0",
                "endIpAddress": "0.0.0.0"
            },
            "dependsOn": [
                "[resourceId('Microsoft.Sql/servers', variables('sqlServerName'))]"
            ]
        }
    ],
    "outputs": {
        "dataFactoryName": {
            "type": "string",
            "value": "[variables('dataFactoryName')]"
        },
        "storageAccountName": {
            "type": "string",
            "value": "[variables('storageAccountName')]"
        },
        "keyVaultName": {
            "type": "string",
            "value": "[variables('keyVaultName')]"
        },
        "sqlServerName": {
            "type": "string",
            "value": "[variables('sqlServerName')]"
        },
        "sqlDatabaseName": {
            "type": "string",
            "value": "[variables('sqlDatabaseName')]"
        }
    }
}
```

---

### 3. Post-Deployment Steps (Crucial)

After deploying the ARM template, you must perform these steps to complete the solution.

#### Step 1: Configure the Self-Hosted Integration Runtime (SHIR)
1.  Create a Windows Virtual Machine in Azure (a `B1s` or `B2s` is usually sufficient and cost-effective). It should have network access to your Q2 SFTP server.
2.  In the Azure portal, navigate to your newly created Data Factory.
3.  Go to `Manage` -> `Integration runtimes` -> `+ New`.
4.  Select `Azure, Self-Hosted` and then `Self-Hosted`.
5.  Give it a name (e.g., `q2-sftp-shir`) and copy the authentication key provided.
6.  On the VM, download and install the SHIR software from the Microsoft Download Center.
7.  During setup, paste the authentication key to register the VM with your Data Factory.

#### Step 2: Populate Azure Key Vault
1.  Navigate to your new Key Vault.
2.  Go to `Secrets` -> `+ Generate/Import`.
3.  Create two secrets:
    *   **`sftp-username`**: The username for the Q2 SFTP server.
    *   **`sftp-password`**: The password for the Q2 SFTP server.
    *   **`sql-connection-string`**: The ADO.NET connection string for your new Azure SQL DB. You can find this in the database's "Connection strings" blade in the portal. **Remember to replace the password placeholder with your actual password.**

#### Step 3: Create SQL Database Schema
Connect to your new Azure SQL Database using a tool like Azure Data Studio or SSMS and run a script to create the necessary objects.

**Sample T-SQL Script:**

```sql
-- Create the target table where your final data will live
CREATE TABLE dbo.YourFinalTable (
    ProductID INT PRIMARY KEY,
    ProductName VARCHAR(255),
    Price DECIMAL(18, 2),
    LastUpdated DATETIME2
);

-- Create a staging table that matches the CSV structure
CREATE TABLE dbo.YourStagingTable (
    ProductID INT,
    ProductName VARCHAR(255),
    Price DECIMAL(18, 2)
);
GO

-- Create the stored procedure that ADF will call
CREATE PROCEDURE dbo.usp_UpdateDataFromCsv
    @BlobFilePath VARCHAR(500) -- Parameter to receive the file path from ADF
AS
BEGIN
    SET NOCOUNT ON;

    -- 1. Clear the staging table
    TRUNCATE TABLE dbo.YourStagingTable;

    -- 2. Bulk load the new CSV data from Blob Storage into the staging table
    -- NOTE: You may need to create a DATABASE SCOPED CREDENTIAL for this to work with Managed Identity
    -- For simplicity, this example uses public access. For production, secure this with Managed Identity or SAS.
    DECLARE @BulkInsertStatement NVARCHAR(MAX);
    SET @BulkInsertStatement = N'
        BULK INSERT dbo.YourStagingTable
        FROM ''' + @BlobFilePath + '''
        WITH (
            DATA_SOURCE = ''YourBlobStorageDataSource'', -- You must create this data source in SQL
            FIRSTROW = 2, -- If your CSV has a header row
            FIELDTERMINATOR = '','',
            ROWTERMINATOR = ''\n'',
            TABLOCK
        );';
    EXEC sp_executesql @BulkInsertStatement;


    -- 3. Use MERGE to update the final table from the staging table
    MERGE dbo.YourFinalTable AS Target
    USING dbo.YourStagingTable AS Source
    ON (Target.ProductID = Source.ProductID)
    WHEN MATCHED THEN
        UPDATE SET
            Target.ProductName = Source.ProductName,
            Target.Price = Source.Price,
            Target.LastUpdated = GETDATE()
    WHEN NOT MATCHED BY TARGET THEN
        INSERT (ProductID, ProductName, Price, LastUpdated)
        VALUES (Source.ProductID, Source.ProductName, Source.Price, GETDATE());
    -- Optional: WHEN NOT MATCHED BY SOURCE THEN DELETE;

    SET NOCOUNT OFF;
END;
GO

-- Create a Data Source pointing to your Blob Storage (required for BULK INSERT)
-- Replace 'yourstorageaccount' with the actual name
CREATE EXTERNAL DATA SOURCE YourBlobStorageDataSource
WITH (
    TYPE = BLOB_STORAGE,
    LOCATION = 'https://[YOUR_STORAGE_ACCOUNT_NAME].blob.core.windows.net/raw-data'
    -- For production, add CREDENTIAL for Managed Identity
);
GO
```

#### Step 4: Create and Deploy Data Factory Artifacts
In the ADF Studio UI, create the following:

1.  **Linked Services:**
    *   **AzureKeyVault:** Point to your Key Vault. Authentication should be `System Assigned Managed Identity`.
    *   **SftpServer:** Type `SFTP`. Connect via your `q2-sftp-shir`. For authentication, pull the username/password from Key Vault.
    *   **AzureBlobStorage:** Point to your storage account. Use `System Assigned Managed Identity` for authentication.
    *   **AzureSqlDatabase:** Point to your SQL DB. For the connection string, choose the `Azure Key Vault` option and select the `sql-connection-string` secret.

2.  **Datasets:**
    *   **SftpCsvFile:** A `DelimitedText` dataset using the `SftpServer` linked service. Parameterize the file path.
    *   **BlobCsvFile:** A `DelimitedText` dataset using the `AzureBlobStorage` linked service. Parameterize the container and file path.

3.  **Pipeline:**
    *   Create a new pipeline (e.g., `pl_ingest_q2_daily`).
    *   Add a **Copy Data** activity:
        *   **Source:** `SftpCsvFile`.
        *   **Sink:** `BlobCsvFile`. Set the sink file path using dynamic content to include the date, e.g., `@concat(formatDateTime(utcNow(), 'yyyy/MM/dd'), '/yourfile.csv')`.
    *   Add a **Stored Procedure** activity and connect it to the success output of the Copy activity.
        *   **Settings:** Select the `AzureSqlDatabase` linked service and the `usp_UpdateDataFromCsv` procedure.
        *   **Parameters:** Import the `@BlobFilePath` parameter. Set its value using the output of the Copy activity, e.g., `@activity('Copy data1').output.sink.fileName`.
    *   Add a **Schedule Trigger** to run this pipeline daily at a specified time.

By following these steps, you will have a robust, automated, and scalable solution for your daily data ingestion needs.