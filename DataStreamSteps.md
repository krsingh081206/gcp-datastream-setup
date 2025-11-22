# Stream Data from AlloyDB to BigQuery using DataStream

This guide provides step-by-step instructions to set up a data streaming pipeline from an AlloyDB for PostgreSQL database to a BigQuery dataset using GCP DataStream. All steps are performed using the `gcloud` command-line tool.

## 1. Prerequisite IAM Permissions

Before you begin, ensure the user or service account you're using to run the `gcloud` commands has the necessary permissions.

### A. Permissions for Your User/Service Account

Grant the following roles to the user or service account that will be executing the setup commands:

*   `roles/datastream.admin`: To create and manage DataStream connection profiles and streams.
*   `roles/compute.networkAdmin`: To manage VPC networks for private connectivity.
*   `roles/iam.serviceAccountUser`: To grant permissions to the DataStream service account.
*   `roles/alloydb.viewer`: To view details of the source AlloyDB cluster.
*   `roles/bigquery.admin`: To create and manage the destination BigQuery dataset and tables.

You can grant these roles using the following `gcloud` commands:

```bash
export PROJECT_ID=$(gcloud config get-value project)
export USER_EMAIL=$(gcloud config get-value account) # Or your service account email

gcloud projects add-iam-policy-binding $PROJECT_ID --member="user:$USER_EMAIL" --role="roles/datastream.admin"
gcloud projects add-iam-policy-binding $PROJECT_ID --member="user:$USER_EMAIL" --role="roles/compute.networkAdmin"
gcloud projects add-iam-policy-binding $PROJECT_ID --member="user:$USER_EMAIL" --role="roles/iam.serviceAccountUser"
gcloud projects add-iam-policy-binding $PROJECT_ID --member="user:$USER_EMAIL" --role="roles/alloydb.viewer"
gcloud projects add-iam-policy-binding $PROJECT_ID --member="user:$USER_EMAIL" --role="roles/bigquery.admin"
```

### B. Permissions for the DataStream Service Account

DataStream uses a special, service-managed service account. **You do not need to create this service account manually.** It is automatically created by Google Cloud when you create your first DataStream resource (like a Private Connection). The steps below are ordered correctly to ensure the service account exists before you grant it permissions.

## 2. Step-by-Step `gcloud` Commands for Pipeline Setup

Here are the steps to configure the source database and create the DataStream pipeline.

### Step 1: Enable Necessary APIs

Ensure the required APIs are enabled for your project.

```bash
gcloud services enable \
    datastream.googleapis.com \
    alloydb.googleapis.com \
    bigquery.googleapis.com \
    compute.googleapis.com
```

### Step 2: Configure Source AlloyDB Database

DataStream uses logical replication for change data capture (CDC) from AlloyDB for PostgreSQL.

1.  **Enable Logical Decoding:**
    Set the `alloydb.logical_decoding` database flag on your AlloyDB cluster.

    ```bash
    # Set environment variables for your AlloyDB instance
    export ALLOYDB_CLUSTER="your-alloydb-cluster-id"
    export ALLOYDB_REGION="your-alloydb-region"
    export ALLOYDB_INSTANCE_ID="your-alloydb-primary-instance-id"
   

gcloud alloydb instances update $ALLOYDB_INSTANCE_ID \
  --cluster=$ALLOYDB_CLUSTER \
  --region=$ALLOYDB_REGION \
  --database-flags=alloydb.logical_decoding=on \
  --project=$PROJECT_ID
    ```
    This operation will restart your AlloyDB instances.

2.  **Create a Dedicated Database User:**
    Connect to your primary AlloyDB instance using `psql` or another client and run the following SQL commands to create a user for DataStream with the necessary privileges.

    ```sql
    -- Create a user with replication permissions
    CREATE USER datastream_user WITH REPLICATION PASSWORD 'your-strong-password';

    -- Grant connection access to the database
    GRANT CONNECT ON DATABASE your_database_name TO datastream_user;

    -- Grant usage on the schema
    GRANT USAGE ON SCHEMA public TO datastream_user;

    -- Grant SELECT on tables you want to stream
    GRANT SELECT ON ALL TABLES IN SCHEMA public TO datastream_user;

    -- Ensure future tables also have SELECT permission for the user
    ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO datastream_user;

    -- Grant Replication with user postgres
    ALTER USER postgres WITH REPLICATION;

    -- Create Publication
    CREATE PUBLICATION dev_publication FOR ALL TABLES;

    -- Create Logical Replication Slot
    SELECT PG_CREATE_LOGICAL_REPLICATION_SLOT('dev_slot', 'pgoutput');
    ```
### Step 3: Set up connectivity between Datastream and AlloyDB via Proxy


```bash
gcloud compute instances create-with-container   --zone=us-east4-c ds-tcp-proxy   --container-image gcr.io/dms-images/tcp-proxy   --tags=ds-tcp-proxy   --container-env=SOURCE_CONFIG=10.24.240.2:5432   --can-ip-forward   --network=default   --machine-type=e2-micro

gcloud compute firewall-rules create ds-proxy1 \
  --direction=INGRESS \
  --priority=1000 \
  --target-tags=ds-tcp-proxy \
  --network=default \
  --action=ALLOW \
  --rules=tcp:5432

  ```

### Step 4: Set Up Private Connectivity (This triggers Service Account creation)

Since AlloyDB instances reside within a VPC, DataStream needs a private connection to access it. Creating this resource is the action that will trigger Google Cloud to create the DataStream service account in your project.

```bash
# Set environment variables for your network
export VPC_NETWORK="your-vpc-network-name"
export DS_REGION="your-datastream-region" # e.g., us-central1
export PRIVATE_CONN_ID="alloydb-private-connection"

gcloud datastream private-connections create $PRIVATE_CONN_ID \
    --location=$DS_REGION \
    --display-name="AlloyDB Private Connection" \
    --vpc=$VPC_NETWORK \
    --subnet=10.0.0.0/29 # This is a placeholder range; GCP will find an available /29 range.
```

```bash
# Verify Private Connection Operation Status
gcloud datastream operations describe   projects/$PROJECT_ID/locations/$ALLOYDB_REGION/operations/operation-1763814170716-6442dfd1869d9-024ea3bd-b5f7cf06

```

### Step 4: Find and Grant Permissions to the DataStream Service Account

Now that the service account has been created, you can find its name and grant it the necessary permissions to access AlloyDB and BigQuery.

1.  **Find the DataStream Service Account:**
    First, get your project number. Then, construct the service account email.

    ```bash
    export PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format="value(projectNumber)")
    export DS_SA="service-${PROJECT_NUMBER}@gcp-sa-datastream.iam.gserviceaccount.com"
    echo "DataStream Service Account: $DS_SA"
    ```

2.  **Grant Permissions to the Service Account:**
    Run the following commands to grant the required roles.

    *   **AlloyDB Access:**
        ```bash
        gcloud projects add-iam-policy-binding $PROJECT_ID --member="serviceAccount:$DS_SA" --role="roles/alloydb.client"
        gcloud projects add-iam-policy-binding $PROJECT_ID --member="serviceAccount:$DS_SA" --role="roles/alloydb.viewer"
        ```
    *   **BigQuery Access:**
        ```bash
        gcloud projects add-iam-policy-binding $PROJECT_ID --member="serviceAccount:$DS_SA" --role="roles/bigquery.dataEditor"
        gcloud projects add-iam-policy-binding $PROJECT_ID --member="serviceAccount:$DS_SA" --role="roles/bigquery.user"
        ```

### Step 5: Create DataStream Connection Profiles

Connection profiles store the connection information for your source and destination.

1.  **Create the AlloyDB Source Connection Profile:**

    ```bash
    # Set environment variables
    export ALLOYDB_SOURCE_PROFILE="alloydb-connection-profile" # Need to create via UI
    export DB_NAME="your_database_name"
    export DB_USER="datastream_user"
    export DB_PASSWORD="your-strong-password" # Consider using Secret Manager for production

    gcloud datastream connection-profiles create $ALLOYDB_SOURCE_PROFILE \
        --location=$DS_REGION \
        --type=alloydb \
        --alloydb-hostname=$ALLOYDB_PROXY_IP \
        --alloydb-username=$DB_USER \
        --alloydb-password=$DB_PASSWORD \
        --alloydb-database=$DB_NAME \
        --private-connection=$PRIVATE_CONN_ID \
        --display-name="AlloyDB Source Profile"
    ```

2.  **Create the BigQuery Destination Connection Profile:**

    ```bash
    export BQ_DEST_PROFILE="bigquery-dest-profile"

    gcloud datastream connection-profiles create $BQ_DEST_PROFILE \
        --location=$DS_REGION \
        --type=bigquery \
        --display-name="BigQuery Destination Profile"
    ```

### Step 6: Create and Start the Stream

Finally, create the stream that connects the source and destination profiles and defines what to replicate.

1.  **Create the Stream:**
    This command defines the stream from AlloyDB to BigQuery. The `backfill-all` strategy performs an initial snapshot of existing data, and `continuous` captures ongoing changes.

    You will need two JSON configuration files for the command below. Create `allowlist.json` and `destination-config.json` in your working directory.

    *   **`allowlist.json`**: Specifies which schemas and tables to include.
        ```json
            {
        "includeObjects": {
        "postgresqlSchemas": [
            {
            "schema": "public",
            "postgresqlTables": [
                {
                "table": "orders"
                }
            ]
            }
        ]
        },
        "replicationSlot": "dev_slot",
        "publication": "dev_publication"
    }
        ```

    *   **`destination-config.json`**: Defines how data is written to BigQuery. The `merge` option enables upserts for CDC events.
        ```json
        {
        "sourceHierarchyDatasets": {
            "datasetTemplate": {
            "location": "us-east4"
            }
        },
        "merge": {},
        "dataFreshness": "60s"
        }
        ```

    Now, run the creation command:

    ```bash
    # Set environment variables
    export STREAM_ID="alloydb-to-bq-stream"
    export BQ_DATASET="public" # This dataset must exist in BigQuery Use UI or Commandline to create


    gcloud datastream streams create $STREAM_ID \
        --location=$DS_REGION \
        --display-name="AlloyDB to BigQuery Stream" \
        --source=$ALLOYDB_SOURCE_PROFILE \
        --destination=$BQ_DEST_PROFILE \
        --backfill-all \
        --postgresql-source-config=allowlist.json \
        --bigquery-destination-config=bigquerydestination.json
    ```

2.  **Start the Stream:**
    After creation, the stream is in a `NOT_STARTED` state. Start it to begin replication.

    ```bash
    gcloud datastream streams update $STREAM_ID \
        --location=$DS_REGION \
        --state=RUNNING
    ```

You can monitor the status of your stream using the GCP Console or with `gcloud datastream streams describe $STREAM_ID --location=$DS_REGION`. Data should begin appearing in your BigQuery dataset shortly.