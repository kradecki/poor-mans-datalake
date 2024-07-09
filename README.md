# Building a Poor man's Datalake.
Let's build our own Snowflake-like Data Lake using Minio, Nessie, Trino and Apache Iceberg.

## 1. Clone this repository and configure environmental variables/
Start by cloning this repository on your local machine.

In the same directory where your newly obtained `docker-compose.yml` file is located create an `.env` file with relevant variables. Replace `<your input>` with your preferred settings:

```sh
cat > .env << EOF
MINIO_ROOT_USER=<your input>
MINIO_ROOT_PASSWORD=<your input>
MINIO_LOCAL_S3_STORAGE=<your input>
NESSIE_LOCAL_CATALOG_PATH=<your input>
EOF
```
where:

- `MINIO_ROOT_USER` is your master minio admin user, eg. `admin`.
- `MINIO_ROOT_PASSWORD` is your master minio admin password, eg. `secretpassword`.
- `MINIO_LOCAL_S3_STORAGE` is the directory on your local machine where the S# objects will be created; it will be bounted in Minio's docker image, eg. `/User/kradcki/data/s3`.
- `NESSIE_LOCAL_CATALOG_PATH` is the directory on your local machine where Nessie will create RocksDB calatalog to persist across docker container restarts, eg. `/Users/kradecki/data/rocksdb`.

## 2. Launch Minio, Nessie and Trine Services using Docker Compose

Examine the downloaded `docker-compose.yml` to make sure you're fine with the settings. If so launch the relevant services.

```sh
docker compose up
```

## 3. Start Trino CLI file.

Refer to the official documentation on how to obtain and launch the most up to date Trino CLI (Command line interface).

```sh
./trino http://localhost:8080
```

## 4. Create a Catalog

Think of a Catalog as a Metadata Store. In ur patricular scenario it will hold relevant data about Apache Iceberg metadata files and manifest files.

The SQL script beloe defines both the Metadata layer (Nessie) and the Storage Layer (Minio).

```sql
CREATE  CATALOG datalake USING iceberg
  WITH (
    "iceberg.catalog.type" = 'nessie',
    "iceberg.nessie-catalog.uri" = 'http://nessie:19120/api/v2',
    "iceberg.nessie-catalog.ref" = 'main',
    "iceberg.file-format" = 'PARQUET',
    "iceberg.nessie-catalog.default-warehouse-dir" = 's3a://datalake/',
    "fs.native-s3.enabled" = 'true',
    "s3.endpoint" = 'http://minio:9000',
    "s3.region" = 'us-east-1',
    "s3.path-style-access" = 'true',
    "s3.aws-access-key" = 'admin',
    "s3.aws-secret-key" = 'paracetamol'
  );
```


## 5. Create a Schema

```sql
CREATE SCHEMA datalake.warehouse;
```

## 6. Create a Table

The table below uses `AVRO` format which is suitable for OLTP-like traffic. For OLAP-heavy workloads, you should opt for `PARQUET` (preferred) or `ORC` formats.

```sql
CREATE TABLE datalake.warehouse.customers (
  firstname VARCHAR,
  lastname VARCHAR,
  citizenship VARCHAR,
  city VARCHAR,
  email VARCHAR
)
WITH (format = 'AVRO');
```
Then let's insert a sample row...

```sql
INSERT INTO 
  datalake.warehouse.customers 
VALUES 
  ('Krzysztof', 'Radecki', 'PL', 'Gdańsk', 'k.radecki@example.com');
```
... and query the table to check it was successfully written:

```sql
SELECT * 
FROM
  datalake.warehouse.customers; 
```

## 7. Clean Up
```sql
DROP TABLE datalake.warehouse.customers;
DROP SCHEMA datalake.warehouse;
DROP CATALOG datalake;
```
