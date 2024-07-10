# Building a Poor man's Datalake: 🦩 + 🦕 + 🐰 + 🧊 = 🚀
Let's build our own Snowflake-like Data Lake using 
- Minio 🦩
- Nessie 🦕
- Trino 🐰
- Apache Iceberg 🧊

<div style="border: 3px solid black; background-color: #ffc700; border-radius: 10px; padding: 10px; color: black; font-family: monospace;">
  <div style="color: #00404f; font-weight: bold;">
    IMPORTANT DISCLAIMERS:
  </div>

  1. The code snippets below include dummy usernames and passwords. I strongly recommend against using them in your deployments.
  2. I'm not a DevOps engineer. The Docker Compose file works for me, but it could most likely be improved.
  3. I use an ARM-based MacBook, and the code was tested using that particular setup.
</div>


## 1. Clone this repository and configure environmental variables
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
- `MINIO_ROOT_PASSWORD` is your master minio admin password, eg. `paracetamol`.
- `MINIO_LOCAL_S3_STORAGE` is the directory on your local machine where the S# objects will be created; it will be bounted in Minio's docker image, eg. `/User/kradcki/data/s3`.
- `NESSIE_LOCAL_CATALOG_PATH` is the directory on your local machine where Nessie will create RocksDB calatalog to persist across docker container restarts, eg. `/Users/kradecki/data/rocksdb`.

## 2. Launch Minio, Nessie and Trine Services using Docker Compose

Examine the provided `docker-compose.yml` to make sure you're fine with the settings. If so launch the relevant services.

```sh
docker compose up
```

## 3. Launch Trino CLI

Refer to the official documentation on how to obtain and launch the most up to date Trino CLI (Command Line Interface).

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
The changes made with the SQL statement above can also be observed directly on the file system:

```sh
sh-5.1$ pwd
/etc/trino/catalog

sh-5.1$ ls -la
total 32
drwxr-xr-x 1 trino trino 4096 Jul  9 18:30 .
drwxr-xr-x 1 trino trino 4096 Jun 28 00:32 ..
-rw-r--r-- 1 trino trino  417 Jul  9 18:30 datalake.properties
-rw-r--r-- 1 trino trino   19 Jun 28 00:32 jmx.properties
-rw-r--r-- 1 trino trino   22 Jun 28 00:32 memory.properties
-rw-r--r-- 1 trino trino   45 Jun 28 00:32 tpcds.properties
-rw-r--r-- 1 trino trino   43 Jun 28 00:32 tpch.properties

sh-5.1$ cat datalake.properties 
#Tue Jul 09 18:30:21 GMT 2024
connector.name=iceberg
fs.native-s3.enabled=true
iceberg.catalog.type=nessie
iceberg.file-format=PARQUET
iceberg.nessie-catalog.default-warehouse-dir=s3a\://datalake/
iceberg.nessie-catalog.ref=main
iceberg.nessie-catalog.uri=http\://nessie\:19120/api/v2
s3.aws-access-key=admin
s3.aws-secret-key=paracetamol
s3.endpoint=http\://minio\:9000
s3.path-style-access=true
s3.region=us-east-1
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

This step is optional but should you need to cleanup your data lake, here's your one-stop-shop:

```sql
DROP TABLE datalake.warehouse.customers;
DROP SCHEMA datalake.warehouse;
DROP CATALOG datalake;
```

## Additional info
https://trino.io/docs/current/connector/iceberg.html#time-travel-queries
https://trino.io/docs/current/connector/iceberg.html#metadata-tables