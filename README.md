# Azure Data Pipeline Project

##  Project Overview

This end-to-end Azure Data Engineering project demonstrates how to build a complete data pipeline that ingests, transforms, and stores structured data using the Azure ecosystem. It showcases secure and scalable best practices using 
Azure Data Factory, Azure Databricks, Azure Data Lake Gen2 and Azure Key Vault for secret management.

---

## Key Components and Flow



###  1. Data Ingestion (ADF → Blob Storage)

- We upload the raw CSV file `sample_sales_data_10000.csv` to Azure Blob Storage inside the `sourcefilepath` container.
- Azure Data Factory (ADF) is used to create a **Copy Data** pipeline.
- The pipeline copies the file from the **source container** to the **destination container** in **Azure Data Lake Storage Gen2**.




###  2. Data Transformation (Databricks + PySpark)

- The '.dbc' notebook ('READ_ADLS_DATA.dbc') is imported into Azure Databricks.
- Using **PySpark**, we:
  - Read the CSV file from ADLS Gen2
  - Perform transformations (e.g., removing nulls, renaming columns & datatypes, filtering records)
  - Write the cleaned data back to ADLS Gen2 in **Parquet** format

###  3. Secure Connection from ADB to ADLS (via Key Vault)

We connect Databricks to ADLS Gen2 using (Azure Key Vault) to store secrets.

####  Step-by-step Connection Flow:

1. Register an App in Azure Active Directory (Service Principal)
   - Generate 'client_id', 'tenant_id', and 'client_secret

2. Store Secrets in Azure Key Vault:
   - Stored these values to build an connection
     - 'client_id', 
     - 'tenant_id', and 
     -'client_secret

3. Create a Databricks Secret Scope
   '''bash
   databricks secrets create-scope --scope kv-scope


4. Mapping the key vault to secret scope

  databricks secrets put --scope kv-scope --key client-id
  databricks secrets put --scope kv-scope --key tenant-id
  databricks secrets put --scope kv-scope --key client-secret

5.Used the  Secrets in Databricks Notebook to Connect to ADLS

configs = {
    "fs.azure.account.auth.type": "OAuth",
    "fs.azure.account.oauth.provider.type": "org.apache.hadoop.fs.azurebfs.oauth2.ClientCredsTokenProvider",
    "fs.azure.account.oauth2.client.id": dbutils.secrets.get(scope="kv-scope", key="client-id"),
    "fs.azure.account.oauth2.client.secret": dbutils.secrets.get(scope="kv-scope", key="client-secret"),
    "fs.azure.account.oauth2.client.endpoint": "https://login.microsoftonline.com/" + dbutils.secrets.get(scope="kv-scope", key="tenant-id") + "/oauth2/token"
}

dbutils.fs.mount(
    source="abfss://destinationfilepath@yourstorageaccount.dfs.core.windows.net/",
    mount_point="/mnt/salesdata",
    extra_configs=configs
)

6.Here we can Read and Write Using Mount Point

df = spark.read.option("header", True).csv("/mnt/salesdata/sample_sales_data_10000.csv")
df_clean = df.dropna().dropDuplicates()
df_clean.write.mode("overwrite").parquet("/mnt/salesdata/cleaned_sales_data/")


