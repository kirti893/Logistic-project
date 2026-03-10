# Logistics Domain - Salesforce to Azure Lakehouse
## Complete Implementation Package

This package contains everything you need to implement an end-to-end Logistics Data Lakehouse using Azure services, extracting data from Salesforce and following the Medallion Architecture (Bronze → Silver → Gold).

---

## 📁 Package Contents

```
logistics_project/
├── README.md                                    # This file
├── logistics_lakehouse_implementation.md        # Detailed technical documentation
├── setup_azure_infrastructure.sh                # Automated Azure setup script
├── deployment_config.json                       # Generated during setup
│
├── adf_pipelines/
│   └── PL_Salesforce_to_ADLS_Dynamic.json      # ADF pipeline JSON
│
├── sql_scripts/
│   └── 01_create_config_and_audit_tables.sql   # SQL configuration tables
│
├── databricks_notebooks/
│   ├── bronze/
│   │   └── (to be created from documentation)
│   ├── silver/
│   │   └── (to be created from documentation)
│   └── gold/
│       └── (to be created from documentation)
│
└── config_files/
    └── (generated configurations)
```

---

## 🚀 Quick Start Guide

### Prerequisites

Before starting, ensure you have:

✅ **Azure Subscription** with Owner or Contributor access  
✅ **Azure CLI** installed ([Install Guide](https://docs.microsoft.com/cli/azure/install-azure-cli))  
✅ **Salesforce Account** with API access  
✅ **bash** shell (Linux, macOS, or WSL on Windows)  
✅ **jq** installed (`sudo apt install jq` or `brew install jq`)  
✅ **sqlcmd** (optional, for automated SQL deployment)

### Step-by-Step Implementation

---

## Phase 1: Azure Infrastructure Setup (Day 1)

### 1.1 Login to Azure

```bash
az login
az account set --subscription "Your-Subscription-Name"
```

### 1.2 Run Automated Setup Script

```bash
# Navigate to project directory
cd logistics_project

# Make script executable (if not already)
chmod +x setup_azure_infrastructure.sh

# Run the setup script
./setup_azure_infrastructure.sh
```

**What this script does:**
- Creates Resource Group
- Creates ADLS Gen2 Storage Account with containers
- Creates Azure Key Vault and stores secrets
- Creates Azure SQL Database for configuration
- Creates Azure Data Factory
- Creates Azure Databricks Workspace
- Creates Service Principal for Databricks
- Grants necessary permissions
- Generates configuration file

**Duration:** ~15-20 minutes

### 1.3 Verify Infrastructure

After the script completes, verify in Azure Portal:

1. Navigate to your resource group: `rg-logistics-lakehouse`
2. Confirm all resources are created:
   - Storage Account (adlslogistics001)
   - Key Vault (kv-logistics-001)
   - SQL Server (sql-logistics-config)
   - Data Factory (adf-logistics-ingestion)
   - Databricks Workspace (dbw-logistics-lakehouse)

---

## Phase 2: Configuration Setup (Day 1-2)

### 2.1 Deploy SQL Configuration Tables

**Option A: Automated (if sqlcmd is installed)**
```bash
# Already done if you chose 'Yes' during setup script
# Otherwise, run manually:
sqlcmd -S sql-logistics-config.database.windows.net \
       -d logistics_config \
       -U sqladmin \
       -P 'YourSecureP@ssw0rd123!' \
       -i sql_scripts/01_create_config_and_audit_tables.sql
```

**Option B: Azure Portal**
1. Go to Azure Portal → SQL Database → Query Editor
2. Login with admin credentials
3. Copy and paste the SQL script
4. Execute

**What this creates:**
- `salesforce_ingestion_config` table (control table)
- `ingestion_audit_log` table (audit logging)
- Stored procedures and views

### 2.2 Verify Configuration Data

```sql
-- Run in SQL Query Editor
SELECT * FROM dbo.salesforce_ingestion_config WHERE is_active = 1;
```

You should see 11 configured Salesforce objects (Account, Load, Carrier, etc.)

---

## Phase 3: Azure Data Factory Setup (Day 2)

### 3.1 Create Linked Services

Navigate to: **Data Factory Studio → Manage → Linked Services**

#### A. Key Vault Linked Service
1. Click **New** → **Azure Key Vault**
2. Name: `LS_AzureKeyVault`
3. Azure subscription: Select your subscription
4. Key vault name: `kv-logistics-001`
5. **Test connection** → **Create**

#### B. Salesforce Linked Service
1. Click **New** → **Salesforce**
2. Name: `LS_Salesforce`
3. Environment URL: `https://login.salesforce.com`
4. Username: **Azure Key Vault** → Select `salesforce-username`
5. Password: **Azure Key Vault** → Select `salesforce-password`
6. Security Token: **Azure Key Vault** → Select `salesforce-security-token`
7. **Test connection** → **Create**

#### C. ADLS Gen2 Linked Service
1. Click **New** → **Azure Data Lake Storage Gen2**
2. Name: `LS_ADLS_Gen2`
3. Azure subscription: Select your subscription
4. Storage account: `adlslogistics001`
5. Authentication: **Managed Identity**
6. **Test connection** → **Create**

#### D. Azure SQL Database Linked Service
1. Click **New** → **Azure SQL Database**
2. Name: `LS_AzureSqlDB_Config`
3. Azure subscription: Select your subscription
4. Server name: `sql-logistics-config`
5. Database: `logistics_config`
6. Authentication: **SQL authentication**
7. Username: **Azure Key Vault** → Select `sql-admin-password`
8. **Test connection** → **Create**

### 3.2 Create Datasets

#### A. Salesforce Dataset (Parameterized)
1. Navigate to **Author** → **Datasets** → **New Dataset**
2. Select **Salesforce**
3. Name: `DS_Salesforce_Object`
4. Linked service: `LS_Salesforce`
5. Add parameter: `ObjectName` (String)
6. Object API Name: `@dataset().ObjectName`
7. **Publish**

#### B. ADLS Parquet Dataset (Parameterized)
1. **New Dataset** → **Azure Data Lake Storage Gen2** → **Parquet**
2. Name: `DS_ADLS_Parquet`
3. Linked service: `LS_ADLS_Gen2`
4. Add parameters:
   - `Container` (String)
   - `FolderPath` (String)
   - `FileName` (String)
5. File path:
   - Container: `@dataset().Container`
   - Directory: `@dataset().FolderPath`
   - File: `@dataset().FileName`
6. **Publish**

#### C. Configuration Table Dataset
1. **New Dataset** → **Azure SQL Database**
2. Name: `DS_Config_Table`
3. Linked service: `LS_AzureSqlDB_Config`
4. Table: `dbo.salesforce_ingestion_config`
5. **Publish**

### 3.3 Import Pipeline

1. Navigate to **Author** → **Pipelines**
2. Click **...** → **Import from pipeline template**
3. Select: `adf_pipelines/PL_Salesforce_to_ADLS_Dynamic.json`
4. Map the datasets and linked services
5. **Publish All**

### 3.4 Test Pipeline

1. Click **Debug** to test the pipeline
2. Monitor execution in **Monitor** tab
3. Check ADLS Gen2 for output parquet files:
   - Container: `raw-salesforce`
   - Path: `Account/2024-01-15/Account_20240115_120000.parquet`

---

## Phase 4: Databricks & Unity Catalog Setup (Day 3-4)

### 4.1 Configure Unity Catalog

**Prerequisites:** You need Databricks Account Admin access

1. Navigate to **Databricks Account Console** (accounts.cloud.databricks.com)
2. Go to **Data** → **Metastores**
3. **Create Metastore**:
   - Name: `logistics_metastore`
   - Region: `East US` (same as your workspace)
   - ADLS Gen2 Path: `abfss://metastore@adlslogistics001.dfs.core.windows.net/`
   - Credential: Create using Service Principal from deployment

4. **Assign Metastore** to your workspace: `dbw-logistics-lakehouse`

### 4.2 Create Storage Credential

In Databricks SQL or Notebook:

```sql
-- Get these values from deployment_config.json
CREATE STORAGE CREDENTIAL adls_logistics_credential
WITH (
  AZURE_SERVICE_PRINCIPAL = '<SP_APP_ID>',
  AZURE_CLIENT_SECRET = '<SP_CLIENT_SECRET>',
  AZURE_DIRECTORY_ID = '<TENANT_ID>'
);
```

### 4.3 Create External Location

```sql
CREATE EXTERNAL LOCATION adls_raw_location
  URL 'abfss://raw-salesforce@adlslogistics001.dfs.core.windows.net/'
  WITH (STORAGE CREDENTIAL adls_logistics_credential);

GRANT READ FILES ON EXTERNAL LOCATION adls_raw_location TO `data_engineers`;
```

### 4.4 Create Catalog Structure

```sql
-- Create logistics catalog
CREATE CATALOG IF NOT EXISTS logistics;
USE CATALOG logistics;

-- Create schemas
CREATE SCHEMA IF NOT EXISTS bronze 
  COMMENT 'Raw data from Salesforce';
  
CREATE SCHEMA IF NOT EXISTS silver 
  COMMENT 'Cleaned and standardized data';
  
CREATE SCHEMA IF NOT EXISTS gold 
  COMMENT 'Analytics-ready dimensional data';

-- Grant permissions
GRANT USE CATALOG ON CATALOG logistics TO `data_engineers`;
GRANT CREATE SCHEMA ON CATALOG logistics TO `data_engineers`;
GRANT ALL PRIVILEGES ON SCHEMA bronze TO `data_engineers`;
GRANT ALL PRIVILEGES ON SCHEMA silver TO `data_engineers`;
GRANT ALL PRIVILEGES ON SCHEMA gold TO `data_engineers`;
```

### 4.5 Configure Secret Scope

```bash
# In Databricks CLI or notebook
databricks secrets create-scope --scope logistics_keyvault \
  --scope-backend-type AZURE_KEYVAULT \
  --resource-id /subscriptions/<subscription-id>/resourceGroups/rg-logistics-lakehouse/providers/Microsoft.KeyVault/vaults/kv-logistics-001 \
  --dns-name https://kv-logistics-001.vault.azure.net/
```

### 4.6 Deploy Databricks Notebooks

Create notebooks from the code in `logistics_lakehouse_implementation.md`:

**Bronze Layer:**
1. Create folder: `/Workspace/Logistics/Bronze/`
2. Create notebook: `01_Load_Bronze_Tables`
3. Copy Python code from Section 7.1
4. Save notebook

**Silver Layer:**
1. Create folder: `/Workspace/Logistics/Silver/`
2. Create notebook: `01_Silver_Transformation`
3. Copy Python code from Section 8.1
4. Save notebook

**Gold Layer:**
1. Create folder: `/Workspace/Logistics/Gold/`
2. Create notebook: `01_Create_Dimensions`
3. Copy Python code from Section 9.1
4. Save notebook

**Quality Checks:**
1. Create folder: `/Workspace/Logistics/Quality/`
2. Create notebook: `01_Data_Quality_Checks`
3. Copy Python code from Section 10.1
4. Save notebook

---

## Phase 5: End-to-End Testing (Day 5)

### 5.1 Test Bronze Layer

1. Run ADF pipeline to ingest Salesforce data
2. Open Bronze notebook in Databricks
3. Set parameters:
   - `source_folder`: (leave empty for all)
   - `processing_date`: `2024-01-15`
4. Run all cells
5. Verify tables created: `logistics.bronze.account`, etc.

### 5.2 Test Silver Layer

1. Open Silver notebook
2. Set parameters:
   - `table_name`: `account`
   - `processing_date`: `2024-01-15`
3. Run all cells
4. Verify transformations and data quality
5. Repeat for other tables: `load`, `carrier`, `billing`

### 5.3 Test Gold Layer

1. Open Gold notebook
2. Run all cells
3. Verify dimension tables:
   - `logistics.gold.dim_load`
   - `logistics.gold.dim_carrier`
   - `logistics.gold.dim_customer`
   - `logistics.gold.dim_date`

### 5.4 Run Data Quality Checks

1. Open Quality notebook
2. Run all cells
3. Review data quality report
4. Address any failed checks

---

## Phase 6: Orchestration & Scheduling (Day 5-6)

### 6.1 Create Databricks Jobs

#### Bronze Job
```json
{
  "name": "Bronze_Layer_Ingestion",
  "tasks": [{
    "task_key": "load_bronze",
    "notebook_task": {
      "notebook_path": "/Workspace/Logistics/Bronze/01_Load_Bronze_Tables",
      "base_parameters": {
        "source_folder": "",
        "processing_date": "{{job.start_time.iso_date}}"
      }
    },
    "job_cluster_key": "bronze_cluster"
  }],
  "job_clusters": [{
    "job_cluster_key": "bronze_cluster",
    "new_cluster": {
      "spark_version": "13.3.x-scala2.12",
      "node_type_id": "Standard_DS3_v2",
      "num_workers": 2
    }
  }],
  "schedule": {
    "quartz_cron_expression": "0 30 2 * * ?",
    "timezone_id": "UTC"
  }
}
```

Similar jobs for Silver and Gold layers.

### 6.2 Create ADF Trigger

1. In Data Factory → **Manage** → **Triggers**
2. **New Trigger** → **Schedule**
3. Name: `TR_Daily_Ingestion`
4. Schedule: Daily at 2:00 AM UTC
5. Activate trigger

### 6.3 End-to-End Workflow

Create a master workflow that chains:
1. ADF Pipeline (Salesforce → ADLS)
2. Bronze Layer Job
3. Silver Layer Job
4. Gold Layer Job
5. Quality Checks

---

## 📊 Connecting BI Tools

### Power BI

1. Get Databricks connection details:
   - Server hostname: `<workspace-url>`
   - HTTP Path: From cluster/SQL warehouse
2. In Power BI Desktop:
   - **Get Data** → **Databricks**
   - Enter connection details
   - Select catalog: `logistics`
   - Select schema: `gold`
   - Import dimension tables

### Tableau

1. **Connect** → **Databricks**
2. Enter workspace URL and credentials
3. Select `logistics.gold` schema
4. Build dashboards

---

## 🔍 Monitoring & Maintenance

### Daily Checks

```bash
# Check ADF pipeline runs
az datafactory pipeline-run query-by-factory \
  --resource-group rg-logistics-lakehouse \
  --factory-name adf-logistics-ingestion \
  --last-updated-after "2024-01-15T00:00:00Z"

# Check data freshness in SQL
sqlcmd -S sql-logistics-config.database.windows.net \
       -d logistics_config \
       -Q "SELECT * FROM dbo.vw_ingestion_success_rate"
```

### Databricks Monitoring

```sql
-- Check table sizes
SELECT 
  table_catalog,
  table_schema,
  table_name,
  ROUND(data_length / 1024 / 1024, 2) as size_mb
FROM information_schema.tables
WHERE table_catalog = 'logistics'
ORDER BY size_mb DESC;

-- Check last update times
DESCRIBE HISTORY logistics.silver.load;
```

---

## 🛠️ Troubleshooting

### Common Issues

#### 1. ADF Pipeline Fails - Authentication Error
**Solution:**
- Verify Key Vault secrets
- Check Salesforce credentials validity
- Ensure security token is current

#### 2. Bronze Layer - File Not Found
**Solution:**
- Check ADLS container structure
- Verify ADF pipeline completed successfully
- Check `processing_date` parameter format

#### 3. Silver Layer - Schema Mismatch
**Solution:**
- Run `DESCRIBE TABLE logistics.bronze.account`
- Update transformation functions if schema changed
- Use `option("mergeSchema", "true")` in write

#### 4. Unity Catalog - Permission Denied
**Solution:**
```sql
-- Grant necessary permissions
GRANT USE CATALOG ON CATALOG logistics TO `your_user`;
GRANT ALL PRIVILEGES ON SCHEMA bronze TO `your_user`;
```

---

## 📈 Performance Optimization

### Recommended Settings

**Bronze Layer:**
- Partition by ingestion date
- Optimize after each load
- Retention: 30 days

**Silver Layer:**
- Partition by business date
- Z-Order on primary keys + dates
- Enable auto-optimize

**Gold Layer:**
- No partitioning (smaller tables)
- Z-Order on join keys
- Cache frequently accessed tables

---

## 📚 Additional Resources

- [Full Technical Documentation](logistics_lakehouse_implementation.md)
- [Azure Data Factory Docs](https://docs.microsoft.com/azure/data-factory/)
- [Databricks Unity Catalog](https://docs.databricks.com/data-governance/unity-catalog/)
- [Delta Lake Best Practices](https://docs.delta.io/latest/best-practices.html)

---

## 🤝 Support & Contribution

For issues or questions:
1. Review the troubleshooting section
2. Check Azure service health
3. Review Databricks cluster logs
4. Contact your Azure/Databricks support team

---

## ✅ Deployment Checklist

- [ ] Azure infrastructure created (Phase 1)
- [ ] SQL configuration tables deployed (Phase 2)
- [ ] ADF linked services configured (Phase 3)
- [ ] ADF pipeline imported and tested (Phase 3)
- [ ] Unity Catalog configured (Phase 4)
- [ ] Databricks notebooks deployed (Phase 4)
- [ ] Bronze layer tested (Phase 5)
- [ ] Silver layer tested (Phase 5)
- [ ] Gold layer tested (Phase 5)
- [ ] Data quality checks passing (Phase 5)
- [ ] Jobs and triggers scheduled (Phase 6)
- [ ] BI tool connected (Phase 6)
- [ ] Monitoring configured (Phase 6)

---

## 📝 License

This project is provided as-is for educational and implementation purposes.

---

**Last Updated:** January 2024  
**Version:** 1.0  
**Status:** Production Ready
