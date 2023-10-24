# Azure Data Engineering Project: Olympics Data

## Azure Pipeline Diagram

![Data Diagram](./img/1.png)

## Prerequisites

1. Azure Account
2. SQL & Python knowledge
3. Data set converted into .csv files and uploaded to GitHub repository

### Resources

- [Spark SQL — PySpark 3.5.0 documentation](https://spark.apache.org/docs/3.5.0/api/python/index.html)

## Step 1: Create Storage Container

- Create a storage account and enable hierarchical file system.
- Create a directory within the container that includes:
  - Raw Data File
  - Transformed Data File

## Step 2: Create Data Factory

- Author > Pipelines > Data Ingestion > Activities > Move and Transform > Copy data
  - Source > New Source > HTTP > Linked Service > New > Enter raw URL
    - Naming convention: nameHTTP
    - Base URL: link to raw
    - Authentication type: anonymous
    - Create > Import schema: none > Ok
  - Sink > New > Azure Data Lake Gen2 > DelimitedText(CSV)
    - Naming convention: nameADLS
  - Linked Services > Select Storage Container created in Step 1 and select the Raw Data folder
    - Naming convention: name.csv
    - Import schema: none
    - Validate and Debug
- Repeat for all the files and connect each "Copy Data" box to each other on completion

![Example 1](./img/2.png)


## Step 3: Create Databricks

- Create compute
  - Single node
  - Runtime version: 12.2 LTS
  - Node type: Standard_DS3_v2 (14 GB memory, 4 cores)
  - Use Photon Acceleration: disabled
- Create New Notebook
  - Attach to your cluster
- Create App registrations
  - Name: app01
  - Save Application (client) ID and Directory (tenant) ID
- Certificates and secrets
  - Create "client secrets"
    - Naming convention: secretkey
    - Save secretkey value

- Application (client) ID: f200b2b2-8afc-40a6-b97d-4c4f1173dccb
- Directory (tenant) ID: 82789861-0998-4d64-916b-6e60e9b78fbc
- Secretkey: bUH8Q~PTxFsIdaAo6vhhyDl7JefzJR~2TDLF1aPl
- Configs

## Step 4: Transform Data

```python
from pyspark.sql.functions import col
from pyspark.sql.types import IntegerType, DoubleType, DateType, BooleanType

configs = {
    "fs.azure.account.auth.type": "OAuth",
    "fs.azure.account.oauth.provider.type": "org.apache.hadoop.fs.azurebfs.oauth2.ClientCredsTokenProvider",
    "fs.azure.account.oauth2.client.id": "f200b2b2-8afc-40a6-b97d-4c4f1173dccb",
    "fs.azure.account.oauth2.client.secret": 'bUH8Q~PTxFsIdaAo6vhhyDl7JefzJR~2TDLF1aPl',
    "fs.azure.account.oauth2.client.endpoint": 'https://login.microsoftonline.com/82789861-0998-4d64-916b-6e60e9b78fbc/oauth2/token'
}

dbutils.fs.mount(
    source="abfss://olympic-data@olympicdatade.dfs.core.windows.net",  # container@storage-account
    mount_point="/mnt/tokyoolymic",
    extra_configs=configs
)
```
- Add permissions to App
- Go to the container, and click on the directory
- Click on the IAM
- Add role assignment > Storage Blob Data Contributor > assign it to "app01"
- Once it's run and mounted, you can remove the mounting code ("dbutils.fs.mount")

```
athletes = spark.read.format("csv").option("header", "true").option("inferSchema", "true").load("/mnt/olympicdata/raw-data/athletes")
coaches = spark.read.format("csv").option("header", "true").option("inferSchema", "true").load("/mnt/olympicdata/raw-data/coaches.csv")
entriesgender = spark.read.format("csv").option("header", "true").option("inferSchema", "true").load("/mnt/olympicdata/raw-data/entriesgender.csv")
medals = spark.read.format("csv").option("header", "true").option("inferSchema", "true").load("/mnt/olympicdata/raw-data/medals.csv")
teams = spark.read.format("csv").option("header", "true").option("inferSchema", "true").load("/mnt/olympicdata/raw-data/teams.csv")
```
### The '.option("inferSchema", "true")' will automatically detect and cast the type of object it detects in the column

```
athletes.show()
```

![Example 2](./img/3.png)


### Calculate the avg # of entries by gender for each discipline
```
avg_entries_by_gender = entriesgender.withColumn(
    "Avg_Female", entriesgender['Female'] / entriesgender['Total']
).withColumn(
    "Avg_Male", entriesgender['Male'] / entriesgender['Total']
)
avg_entries_by_gender.show()
```
![Example 3](./img/4.png)

### Write transformed data in the "transformed-data" folder

```
athletes.repartition(1).write.mode("overwrite").option("header", "true").csv("/mnt/olympicdata/transformed-data/athletes")
coaches.repartition(1).write.mode("overwrite").option("header", "true").csv("/mnt/olympicdata/transformed-data/coaches")
entriesgender.repartition(1).write.mode("overwrite").option("header", "true").csv("/mnt/olympicdata/transformed-data/entriesgender")
medals.repartition(1).write.mode("overwrite").option("header", "true").csv("/mnt/olympicdata/transformed-data/medals")
teams.repartition(1).write.mode("overwrite").option("header", "true").csv("/mnt/olympicdata/transformed-data/teams")
```
![Example 4](./img/5.png)

## Step 5: Load Data into Synapse Analytics

- Create a Synapse Analytics workspace.
- Once the resource is deployed, open it.
- Launch Synapse Analytics.
- In the Synapse Analytics workspace:
   - Go to Home > Data.
   - Select "Add new Table from data lake."
- For each table to be added:
   - Define an external table name, e.g., "athletes."
   - Link it to the only available service.
   - Specify the input file located within the designated folder, starting with "part..."
     
![Example 5](./img/6.png)

- Continue and enable the "First row, Infer column names"
- Create and repeat for all tables
- Validate and remove illegal characters defined for the column names, then Publish

![Example 6](./img/7.png)

- You can now query the data
  
![Example 7](./img/8.png)

## Step 6: Use Azure Synapse as Data Source for Power BI
- Sign in to Power BI Desktop, and select Azure Synapse Workplace (Beta) as the source


