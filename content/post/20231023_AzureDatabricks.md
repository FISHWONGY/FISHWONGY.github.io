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

```powershell
 # Create the secret
az keyvault secret set --vault-name contosoKeyVault10 --name storageKey --value "value of your key1"
```

2 - Create Scope on Databricks, go to the link, follow the steps and create scope. Note that this only need tobe created once, there can be multiple secrets under the scope

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


3 - Working with databricks cli

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

4 - Accessing blob data in Azure Databricks

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

```sql
COPY INTO BRONZE_SCHEMA.TBL_NAME
FROM 'wasbs://<container-name>@<storage-account>.blob.core.windows.net/<file-prefix>-*' WITH (
  CREDENTIAL (AZURE_SAS_TOKEN = <the-sas-token-obtain-from-above>)
FILEFORMAT = JSON
COPY_OPTIONS ('mergeSchema' = 'true')
```


5 - Configure spark cluster so that it can connect to GCS Bucket
```powershell
spark.databricks.delta.preview.enabled true
spark.hadoop.google.cloud.auth.service.account.enable true
spark.master local[*, 4]
spark.databricks.cluster.profile singleNode

spark.hadoop.fs.gs.project.id <gcp-project-id>
spark.hadoop.fs.gs.auth.service.account.private.key {{secrets/<adb-scope>/gsaPrivateKey}}
spark.hadoop.fs.gs.auth.service.account.email <svc-acc>@<gcp-project-id>.iam.gserviceaccount.com
spark.hadoop.fs.gs.auth.service.account.private.key.id {{secrets/<adb-scope>/gsaPrivateKeyId}}


PYSPARK_PYTHON=/databricks/python3/bin/python3
```

Reading data from GCS bucket - execute the below in a databricks notebook cell

```python
df = spark.read.format("csv").option("compression", "gzip").load("gs://gcs-bucket/FILE_NAME_*.csv.gz")
```

Saving Spark DF to GCS - execute the below in a databricks notebook cell

```python
current_datetime = datetime.now().strftime("%Y%m%d%H%M%S")
TEMPORARY_TARGET=f"gs://gcs-fbucket/spark-tmp/{current_datetime}"
DESIRED_TARGET=f"gs://gcs-fbucket/PRED_DATA-{current_datetime}.csv"

prediction_final_df.coalesce(1).write.option("header", "true").mode("overwrite").csv(TEMPORARY_TARGET)

temporary_csv = os.path.join(TEMPORARY_TARGET, dbutils.fs.ls(TEMPORARY_TARGET)[3][1])

dbutils.fs.cp(temporary_csv, DESIRED_TARGET)
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
            access_token="<access-to>",
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


That's it for now, hope you found this helpful.

Thank you for reading and have a nice day!

