---
authors:
- Hugo Authors
date: "2023-10-23"
excerpt: My learnings when setting up Azure Databricks
hero: /images/on_azureDatabricks/adb_cover.png
title: On Azure Databricks
---

<div style="text-align: justify">

Recently, I explored Azure Databricks for a PoC project with machine learning workload. Since Snowflake and GCP are our base, setting up Databricks with Azure proved to be a bit tricky.

I know, I know, using Databricks on GCP would be way easier in this case. There are several reasons why I couldn't do that. And, who likes it easy? (Me, actually...)

Anyway, it is what it is. Let's begin with my learning journey.


<u><b>
    <p style="font-size:20pt ">
      Setting Up Configurations
    </p>
</b></u>


1 - Installation

Here, we are installing the azure cli, databricks cli and azcopy. 
`azcopy` is a fasat and scalable solutionn to move data across cloud storages.

```powershell
brew update && brew install azure-cli

brew install azcopy

brew install databricks

pip install databricks-cli
```

2 - Authentication

```powershell
az login

azcopy login
```

To check relevant info about our account and the cli, 

```powershell
az account list-locations -o table

az account set --subscription <subscription-id>

az --version
```


To move data across cloud storage, say for example, from Google Cloud Storage to Azure Blob, we will need a GCP service account with sufficient storage priviledge and export creds.json locally.

For more details, check [here](https://learn.microsoft.com/en-us/azure/storage/common/storage-use-azcopy-google-cloud#choose-how-youll-provide-authorization-credentials)

```powershell
export GOOGLE_APPLICATION_CREDENTIALS=/your/file/path/gcp-creds.json
```

<u><b>
    <p style="font-size:20pt ">
      File Cloning with AZCOPY
    </p>
</b></u>

Following the [Azure doc](https://learn.microsoft.com/en-us/azure/storage/common/storage-use-azcopy-blobs-upload#get-started), I then clone data across cloud storage

```powershell
azcopy copy 'https://storage.cloud.google.com/<bucket-name>' 'https://<storage-account>.blob.core.windows.net/<container-name>' --recursive=true
```

After data transferred, we can check the blob objects via - 

```powershell
az storage blob list --account-name <storage-account> --container-name <container-name> --output table
```

To bulk delete files under a folder
```powershell
#!/bin/bash
RESOURCE_GROUP="grp-name"
ACCOUNT_NAME="acc-name"
PREFIX="file-prefix"
CONTAINER="container-name"
EXCLUDE_FOLDER="exclude-subfolder"

# Optionally as a first step
# List all file inside container container-name/subfolder
az storage account keys list --resource-group $RESOURCE_GROUP --account-name $ACCOUNT_NAME --query '[0].value' -o tsv | \
xargs -I {} az storage blob list --account-name $ACCOUNT_NAME --account-key {} --container-name $CONTAINER --prefix subfolder- --query "[?name][].name" -o tsv | grep -v '/$EXCLUDE_FOLDER/'
```

```powershell
#!/bin/bash
RESOURCE_GROUP="grp-name"
ACCOUNT_NAME="acc-name"
PREFIX="file-prefix"
CONTAINER="container-name"
EXCLUDE_FOLDER="exclude-subfolder"

# delete all file with prefix file-prefix inside container container-name/subfolder

az storage account keys list --resource-group $RESOURCE_GROUP --account-name $ACCOUNT_NAME --query '[0].value' -o tsv | \
xargs -I {} az storage blob list --account-name $ACCOUNT_NAME --account-key {} --container-name $CONTAINER --prefix $PREFIX- --query "[?name][].name" -o tsv | grep -v '/$EXCLUDE_FOLDER/' | \
xargs -I {} az storage blob delete --account-name $ACCOUNT_NAME --account-key $(az storage account keys list --resource-group $RESOURCE_GROUP --account-name $ACCOUNT_NAME --query '[0].value' -o tsv) --container-name $CONTAINER --name {}
```


<u><b>
    <p style="font-size:20pt ">
      Working with Azure Databricks
    </p>
</b></u>


<u>Access blob data</u>

After cloning the data, we need access the data.

1 - Setting up [KeyVault]( https://learn.microsoft.com/en-us/azure/key-vault/general/integrate-databricks-blob-storage)


```powershell
az storage account keys list -g groupName -n groupname

# An example
az storage account keys list -g contosoResourceGroup5 -n contosoblobstorage5
```


```powershell
# Create a Key Vault -- for secret setting purpose after
az keyvault create --name contosoblobstorage5 --resource-group contosoResourceGroup5 --location southcentralus
```


Using contosoKeyVault10 as example provided on the [Azure documentation]( https://learn.microsoft.com/en-us/azure/key-vault/general/integrate-databricks-blob-storage)
```powershell
 # Create the secret
az keyvault secret set --vault-name contosoKeyVault10 --name storageKey --value "value of your key1"
```

2 - Create Scope on Databricks backed by Azure, go to the link, follow the steps and create scope. Note that this only need tobe created once, there can be multiple secrets under the scope

https://'adb-instance'#secrets/createScope


To get the below info, 

Go to Azure Portal > Home > Subscription > my sub = MS Enterprise > Resources under Settings > 
find your keyvault under resources > Properties under setting


- Scope Name = dbutils.secrets.get(scope = "<scope-name-in-databricks>", ...)
- DNS name = Vault URI
- Resource ID = Resource ID


3 - Configure databricks cli

We need to obtain an access token first

Go to user setting -> access token -> generate token

```powershell
databricks configure --token
```


url = https://'your-adb-instance'.azuredatabricks.net/


token = The token obtained above


4 - Working with databricks cli

```powershell
# check scope created
databricks secrets list-scopes
```

```powershell
# List secret in a given scope
databricks secrets list --scope <scope-name>
```

To create secrets under scope, in this case our secret name is `storageKey`

```powershell
az keyvault secret set --vault-name contosoKeyVault10 --name storageKey --value "<your-key>"

az keyvault secret show --name "storageKey" --vault-name "contosoKeyVault10" --query "value"
```

```powershell
# Extra - if we want to set a secret key with creds on a local folder
az keyvault secret set --vault-name contosoKeyVault10 --name gsaPrivateKey --file "/your/file/path/gcp_key.txt"
```

```powershell
# Extra2 - Delete secret under a keyVault
az keyvault secret delete --vault-name contosoKeyVault10 --name gsaPrivateKey
```

```powershell
# if fail to run secret set, re-run the below
az storage account keys list -g contosoResourceGroup5 -n contosoblobstorage5

az account set --subscription <subscription-id>
az keyvault secret list --vault-name contosoKeyVault10
```


Create ADB Secrets Scope backed by Databricks

```powershell
databricks secrets  create-scope --scope scope-name

# To verifiy has the scope created successfully
databricks secrets list-scopes

# Create secret in the scope
databricks secrets put --scope scope-name --key secrete-name
# Then it will pop up a .txt window, paste the secret value there and type `:wq` to save(write) and quit

# Verify secrets has added successfully
databricks secrets list --scope scope-name 

# Verfiy my ACL access of the scope
databricks secrets get-acl --scope scope-name --principal email@email.com

# Give permission to other
databricks secrets put-acl --scope scope-name --principal email@email.com --permission READ     
```

5 - Accessing blob data in Azure Databricks

Finally, after all the set up, we can now try access our data from the storage.

Reading the data into a Spark DF - execute the below in a databricks notebook cell

```python
dbutils.fs.mount(
source = "wasbs://<container-name>@<storage-account>.blob.core.windows.net",
mount_point = "/mnt/blobstorage",
extra_configs = {"fs.azure.account.key.<storage-account>.blob.core.windows.net":
dbutils.secrets.get(scope = "<scope-name-in-databricks>", key = "storageKey")}) # storageKey is the key we set on the step above

df = spark.read.json("/mnt/blobstorage/<file-name>")

df.show()
```

We can also create a table in Azure Databricks from storage data.

To do so, we will first need a SAS Token

```powershell
# To get SAS Token
az storage container generate-sas \
    --account-name <account-name> \
    --name <container-name> \
    --permissions acdlrw \
    --expiry 2023-09-23T00:00Z \
    --auth-mode login \
    --as-user
```

Note of permission parameter
```md
--permissions
The permissions the SAS grants. Allowed values: (a)dd (c)reate (d)elete (e)xecute (f)ilter_by_tags (i)set_immutability_policy (l)ist (m)ove (r)ead (t)ag (w)rite (x)delete_previous_version (y)permanent_delete. Do not use if a stored access policy is referenced with --id that specifies this value. Can be combined.
```

Then we can load the data to the destination table.

```sql
CREATE SCHEMA BRONZE_SCHEMA;
CREATE TABLE BRONZE_SCHEMA.TBL_NAME;
```

Loading from json
```sql
COPY INTO BRONZE_SCHEMA.TBL_NAME
FROM 'wasbs://<container-name>@<storage-account>.blob.core.windows.net/<file-prefix>-*' WITH (
  CREDENTIAL (AZURE_SAS_TOKEN = <the-sas-token-obtain-from-above>)
FILEFORMAT = JSON
COPY_OPTIONS ('mergeSchema' = 'true')
```

Loading from csv
```sql
COPY INTO BRONZE_SCHEMA.TBL_NAME
FROM 'wasbs://<container-name>@<storage-account>.blob.core.windows.net/<file-prefix>-*' WITH (
  CREDENTIAL (AZURE_SAS_TOKEN = <the-sas-token-obtain-from-above>)
FILEFORMAT = CSV
FORMAT_OPTIONS ('mergeSchema' = 'true',
                'header' = 'false')
COPY_OPTIONS ('mergeSchema' = 'true')
```


Depending on the above format options for headers, col. name might need to be adjusted
```sql
ALTER TABLE BRONZE_SCHEMA.TBL_NAME SET TBLPROPERTIES (
   'delta.columnMapping.mode' = 'name',
   'delta.minReaderVersion' = '2',
   'delta.minWriterVersion' = '5');

ALTER TABLE BRONZE_SCHEMA.TBL_NAME RENAME COLUMN _c0 TO COL_NAME1;
ALTER TABLE BRONZE_SCHEMA.TBL_NAME RENAME COLUMN _c1 TO COL_NAME2;
```

5 - Configure spark cluster so that it can connect to GCS Bucket
```powershell
spark.databricks.delta.preview.enabled true
spark.hadoop.google.cloud.auth.service.account.enable true

spark.hadoop.fs.gs.project.id <gcp-project-id>
spark.hadoop.fs.gs.auth.service.account.private.key {{secrets/<adb-scope>/gsaPrivateKey}}
spark.hadoop.fs.gs.auth.service.account.email <svc-acc>@<gcp-project-id>.iam.gserviceaccount.com
spark.hadoop.fs.gs.auth.service.account.private.key.id {{secrets/<adb-scope>/gsaPrivateKeyId}}
spark.hadoop.fs.azure.account.key.{account}.blob.core.windows.net {{secrets/blobStorageScope/storageKey}}

PYSPARK_PYTHON=/databricks/python3/bin/python3
```

Reading data from cloud storage (GCS bucket/ Azure Blob) - execute the below in a databricks notebook cell

```python
# GCS - csv
df = spark.read.format("csv").option("compression", "gzip").load("gs://gcs-bucket/FILE_NAME_*.csv.gz")

# Config for reading zipped csv from unloading data from Snowflake
df = spark.read.format("csv") \
    .option("compression", "gzip") \
    .option("delimiter", "\t") \
    .option("inferSchema", "true") \
    .load("gs://gcs-bucket/FILE_NAME_*.csv.gz")

# Azure
df = spark.read.json("/mnt/blobstorage/subfolder/dest-folder-name/*.json")

df.limit(10).display()
```

Saving Spark DF to GCS - execute the below in a databricks notebook cell

```python
current_datetime = datetime.now().strftime("%Y%m%d%H%M%S")
TEMPORARY_TARGET=f"gs://gcs-bucket/spark-tmp/{current_datetime}"
DESIRED_TARGET=f"gs://gcs-bucket/PRED_DATA-{current_datetime}.csv"

# This will save file in 1 json file
prediction_final_df.coalesce(1).write.option("header", "true").mode("overwrite").csv(TEMPORARY_TARGET)

temporary_csv = os.path.join(TEMPORARY_TARGET, dbutils.fs.ls(TEMPORARY_TARGET)[3][1])

dbutils.fs.cp(temporary_csv, DESIRED_TARGET)
```

Saving Spark DF to Azure Blob
```python
dbutils.fs.mount(
source = f"wasbs://{container-name}@{account-name}.blob.core.windows.net",
mount_point = "/mnt/blobstorage",
extra_configs = {f"fs.azure.account.key.{account-name}.blob.core.windows.net": dbutils.secrets.get(scope = f'{scopeName}', key = f'{storage-key}')
                 })
```

```python
# Write the DataFrame as a single JSON file to Azure Blob Storage
df.coalesce(1).write.format("json").mode("overwrite").save("/mnt/blobstorage/subfoler/dest-folder-name")

# Write to a folder
df.write.format("json").mode("overwrite").save("/mnt/blobstorage/subfoler/dest-folder-name")
```

Save df as a delta table in databricks
```python
df.write.format('delta') \
    .mode('overwrite') \
    .option("mergeSchema", "true") \
    .saveAsTable('BRONZE_LAYER.TBL_NAME')
```

6 - Inserting Data into Databricks using Python

One of the good thing I like about Databricks' python connector is that it allows to bulk load as much data as I want, and snowflake's connector has a limit of 16384.

```python
from databricks import sql
from typing import Optional, Iterator
from pydantic import BaseModel, ValidationError

def adbconnect():
    try:
        return sql.connect(
            server_hostname="<adb-subscription>.azuredatabricks.net",
            http_path="/sql/1.0/warehouses/<wh-id>",
            access_token="<access-token>",
        )
    except Exception as e:
        raise DatabricksError("Could not connect to Azure Databricks") from e
        

def query(conn, sql: str) -> list[dict]:
    cursor = conn.cursor()
    try:
        cursor.execute(sql)
        result = cursor.fetchall()
    except Exception as e:
        logger.exception(f"Error executing query: {sql}")
        raise DatabricksError("Error executing query") from e
    finally:
        cursor.close()
    return result

def format_value(value):
    """Need to reformat data value to avoid insert failure"""
    if value is None or value == '\\':
        return "NULL"
    elif isinstance(value, str):
        modified_value = value.replace('"', "'")
        return f'"{modified_value}"'
    elif isinstance(value, datetime):
        return f'"{value}"'
    else:
        return str(value)

def insert_data_updates(conn, data: list[DataModel]):
    try:
        data_fields: list[str] = list(DataModel.__fields__.keys())

        values_str = ",\n".join(
            "("
            + ", ".join(format_value(getattr(d, field)) for field in data_fields)
            + ")"
            for d in data
        )

        result = f"""INSERT INTO BRONZE_SCHEMA.TBL_NAME ({", ".join(data_fields)})  
        VALUES\n{values_str};"""
        conn.cursor().execute(result)

        conn.commit()
    except Exception as e:
        conn.rollback()
        raise DatabricksError("Error inserting data, transaction rolled back") from e
```

<u><b>
    <p style="font-size:20pt ">
      Applying the above knowledge and the lessons I've learnt
    </p>
</b></u>

Background: 
I have to migrate large amount of sensor data to Databricks.

Initially, I plan to migrate the data as the below steps.
1. Move data from GCS to Azure Container
2. Execute the `COPY INTO` command in Azure Databricks to load the data into delta table

So ideally, the steps is running the below in terminal
```powershell
azcopy login

azcopy copy 'https://storage.cloud.google.com/<bucket-name>' 'https://<storage-account>.blob.core.windows.net/<container-name>' --recursive=true
```

Then in Azure Databricks Notebook, execute the below
```sql
COPY INTO BRONZE_SCHEMA.TBL_NAME
FROM 'wasbs://<container-name>@<storage-account>.blob.core.windows.net/folder/subfolder/file-prefix-*' WITH (
  CREDENTIAL (AZURE_SAS_TOKEN = <the-sas-token-obtain-from-above>)
FILEFORMAT = JSON
COPY_OPTIONS ('mergeSchema' = 'true')
```

However, when I executed the above in databricks, it returned the error messages
```md
IllegalArgumentException: 'java.net.URISyntaxException: Relative path in 
absolute URI: sensor-data-2024-01-01 00:00:00.000'
```
After some googling and the help from ChatGPT, I realised the issue is with naming of those json files, mainly due to the whitespaces.

Then on stackoverflow, I found this [post](https://stackoverflow.com/questions/73950317/handling-spaces-in-the-abfss-using-copy-into-with-azure-databricks), so I tried something like this

```sql
COPY INTO BRONZE_SCHEMA.TBL_NAME
FROM 'wasbs://<container-name>@<storage-account>.blob.core.windows.net/folder/subfolder/' WITH (
  CREDENTIAL (AZURE_SAS_TOKEN = <the-sas-token-obtain-from-above>)
FILEFORMAT = JSON
PATTERN='file-prefix-*'
COPY_OPTIONS ('mergeSchema' = 'true')
```

But I am still getting the same error message. I finally realised that as long as I am trying to load my files using regex expression, it will failed because of the naming. That means, if I execute the below, the file will be copied successfully.
```sql
COPY INTO BRONZE_SCHEMA.TBL_NAME
FROM 'wasbs://<container-name>@<storage-account>.blob.core.windows.net/folder/subfolder/sensor-data-2024-01-01 00:00:00.000' WITH (
  CREDENTIAL (AZURE_SAS_TOKEN = <the-sas-token-obtain-from-above>)
FILEFORMAT = JSON
COPY_OPTIONS ('mergeSchema' = 'true')
```

It is when I tried to bulk load everything using regular expression, the execution will fail.

Some approaches we can take to tackle this is,

1 - rename all of the file using az cli
```powershell
#!/bin/bash

ACCOUNT_NAME="acc-name"
ACCOUNT_KEY="acc-key"  
CONTAINER_NAME="container-name"
SAS_TOKEN="token"

# Source and destination prefix
SOURCE_PREFIX="container/path/"
DESTINATION_PREFIX="final-path/"

az storage blob list \
  --account-name $ACCOUNT_NAME \
  --account-key $ACCOUNT_KEY \
  --container-name $CONTAINER_NAME \
  --prefix $SOURCE_PREFIX \
  --sas-token $SAS_TOKEN \
  --output tsv \
  --query "[?name][].name" | while IFS=$'\t' read -r blobname; do

    # New blob name with spaces replaced by underscores
    NEW_BLOB_NAME=$(echo "$blobname" | tr ' ' '_')

    # Copy blob to new name
    az storage blob copy start \
      --account-name $ACCOUNT_NAME \
      --account-key $ACCOUNT_KEY \
      --destination-blob "$NEW_BLOB_NAME" \
      --destination-container $CONTAINER_NAME \
      --source-uri "https://${ACCOUNT_NAME}.blob.core.windows.net/${CONTAINER_NAME}/${blobname}?${SAS_TOKEN}" \

    # Monitor the status of the blob copy to wait for the operation to complete
    copy_status="pending"
    while [ "$copy_status" != "success" ]; do
        copy_status=$(az storage blob show \
            --name "$NEW_BLOB_NAME" \
            --container-name $CONTAINER_NAME \
            --account-name $ACCOUNT_NAME \
            --sas-token $SAS_TOKEN \
            --query "properties.copy.status" --output tsv)
        sleep 1
    done

    # Delete the old blob after successful copy
    az storage blob delete \
      --account-name $ACCOUNT_NAME \
      --account-key $ACCOUNT_KEY \
      --container-name $CONTAINER_NAME \
      --name "$blobname" \
      --sas-token $SAS_TOKEN \

    echo "Renamed $blobname to $NEW_BLOB_NAME"
done
```

2 - Or we can leverage `gsutil` to rename the file on the GCS side then use azcopy to move the file back to Azure and load into Databricks

```powershell
# Copy files over if don't want to rename directly on the current bucket
# Using cp
gsutil cp -r gs://bucket-name/folder1/folder_to_copy gs://bucket-name/folder1/new_folder

# Using rsync - will be more expensive
gsutil rsync -r gs://bucket-name/folder1/folder_to_copy gs://bucket-name/folder1/new_folder
```

```powershell
# Rename files
gsutil ls gs://<bucket>/<folder_with_the_files_to_rename>/ | \
  while read f; do
    gsutil -m mv "$f" "${f// /-}";
  done;
```

```powershell
azcopy copy 'https://storage.cloud.google.com/<bucket-name>' 'https://<storage-account>.blob.core.windows.net/<container-name>' --recursive=true
```

And load the properly named files into Azure as a last step.
```sql
COPY INTO BRONZE_SCHEMA.TBL_NAME
FROM 'wasbs://<container-name>@<storage-account>.blob.core.windows.net/folder/subfolder/file-prefix-*' WITH (
  CREDENTIAL (AZURE_SAS_TOKEN = <the-sas-token-obtain-from-above>)
FILEFORMAT = JSON
COPY_OPTIONS ('mergeSchema' = 'true')
```

However, this is unnecessarily lengthy, as the data have to travel across multiple cloud and likely different regions, it will be costly. Also if the GCS storage is not hot sotrage, it will be even more costly to copy, rename and move acorss cloud.

So, in this case, a more direct way to load these sensor data into databricks is by the below steps.

1 - Unload data from Snowflake to GCS

Execute the below in Snowflake
```sql
-- Clear stage if you prefer to do so
REMOVE @ENV_DATA_TRANSFER;

COPY INTO 'gcs://gcs-buscket-name/SENSOR_DATA_TBL'
FROM "DATABASE"."SCHEMA"."SENSOR_DATA_TBL"
STORAGE_INTEGRATION = <gcp-storage-integration>
overwrite=TRUE;
```

2 - Read data from GCS to Azure databricks directly

Make sure cluster is configured properly
```powershell
spark.databricks.delta.preview.enabled true
spark.hadoop.google.cloud.auth.service.account.enable true

spark.hadoop.fs.gs.project.id <gcp-project-id>
spark.hadoop.fs.gs.auth.service.account.private.key {{secrets/<adb-scope>/gsaPrivateKey}}
spark.hadoop.fs.gs.auth.service.account.email <svc-acc>@<gcp-project-id>.iam.gserviceaccount.com
spark.hadoop.fs.gs.auth.service.account.private.key.id {{secrets/<adb-scope>/gsaPrivateKeyId}}
spark.hadoop.fs.azure.account.key.{account}.blob.core.windows.net {{secrets/blobStorageScope/storageKey}}



PYSPARK_PYTHON=/databricks/python3/bin/python3
```

```python
df = spark.read.format("csv") \
    .option("compression", "gzip") \
    .option("delimiter", "\t") \
    .option("inferSchema", "true") \
    .load("gs://gcs-buscket-name/SENSOR_DATA_TBL_*.csv.gz")


df.limit(10).display()

# Write as delta table
df.write.format('delta') \
    .mode('overwrite') \
    .option("mergeSchema", "true") \
    .saveAsTable('BRONZE_SCHEMA.SENSOR_DATA_TBL')
    
# Write to a folder if want to have the file to be stored in Azure Container
df.write.format("json").mode("overwrite").save("/mnt/blobstorage/folder/subfolder")
```

And finally, if we need to grat user access to external file system to avoid insufficient priviledges, the below commands are needed to be executed as admin.
```sql
GRANT SELECT ON ANY FILE TO `user@email.com`;

GRANT MODIFY ON ANY FILE TO `user@email.com`;
```

<br />
That's it for now, hope you found this helpful.

Thank you for reading and have a nice day!

