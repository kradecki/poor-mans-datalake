# LEVEL 1: Quickstart - Let's just make it work
In the first iteration, we will ignore aspects such as securing each endpoint or enforcing UI access with a password for particular services. The aim is to have a locally running playground where we can explore the features of a Data Lake.


## 1. Clone this repository and configure environmental variables
Start by cloning this repository on your local machine.

In the same directory where your newly obtained `docker-compose.yml` file is located create an `.env` file with relevant variables. Replace `<your input>` with your preferred settings:

```sh
cat > .env << EOF
MINIO_ROOT_USER=<your input>
MINIO_ROOT_PASSWORD=<your input>
MINIO_LOCAL_S3_STORAGE=<your input>
NESSIE_LOCAL_CATALOG_PATH=<your input>
TRINO_LOCAL_CATALOG_PATH=<your input>
EOF
```
where:

- `MINIO_ROOT_USER` is your master minio admin user, eg. `admin`.
- `MINIO_ROOT_PASSWORD` is your master minio admin password, eg. `paracetamol`.
- `MINIO_LOCAL_S3_STORAGE` is the directory on your local machine where the S# objects will be created; it will be bounted in Minio's docker image, eg. `/User/kradcki/data/s3`.
- `NESSIE_LOCAL_CATALOG_PATH` is the directory on your local machine where Nessie will create RocksDB calatalog to persist across docker container restarts, eg. `/Users/kradecki/data/rocksdb`.
- `TRINO_LOCAL_CATALOG_PATH` is the directory, where trino creates properties files once you create a catalog using SQL `CREATE CATALOG` statement; alternatively, the catalogs files can be directly edited here; eg. `/Users/kradecki/data/trino`.

## 2. Launch Minio, Nessie and Trino Services using Docker Compose

Examine the provided `docker-compose.yml` to make sure you're fine with the settings. If so launch the relevant services.

```sh
docker compose up -d
```

## 3. Launch Trino CLI

Refer to the official documentation on how to obtain and launch the most up to date Trino CLI (Command Line Interface).

```sh
./trino http://localhost:8080
```

## 4. Create a Catalog

Think of a Catalog as a Metadata Store. In our particular scenario it will hold relevant data about Apache Iceberg metadata files and manifest files.

The SQL script beloe configure connection to both: the Metadata layer (Nessie) and the Storage Layer (Minio).

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
The changes made with the SQL statement above can also be observed directly on the file system (in the Trino Container):

```sh
sh-5.1$ pwd
/etc/trino/catalog

sh-5.1$ ls -la
total 32
drwxr-xr-x 1 trino trino 4096 Jul  9 18:30 .
drwxr-xr-x 1 trino trino 4096 Jun 28 00:32 ..
-rw-r--r-- 1 trino trino  417 Jul  9 18:30 datalake.properties

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
Sources:
- https://trino.io/docs/current/object-storage/metastores.html#nessie-catalog
- https://trino.io/docs/current/installation/containers.html#configuring-trino

## 5. Create a Schema

```sql
CREATE SCHEMA datalake.warehouse;
```

## 6. Create a Table

The table below uses `PARQUET` format, which is just like `ORC` suitable for OLAP-heavy workloads (analytics). For OLTP-like traffic (insert-heavy, no analytics), you can opt for `AVRO`.

It was designed to work with the official NYC TLC Trip Record Data that you can obtain [here](https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page).

```sql
CREATE TABLE datalake.warehouse.nyc_taxi (
    vendor_id INTEGER COMMENT 'ID of the vendor',
    tpep_pickup_datetime TIMESTAMP COMMENT 'Pick-up date and time',
    tpep_dropoff_datetime TIMESTAMP COMMENT 'Drop-off date and time',
    passenger_count BIGINT COMMENT 'Number of passengers',
    trip_distance DOUBLE COMMENT 'Distance of the trip in miles',
    ratecode_id BIGINT COMMENT 'Rate code for the trip',
    store_and_fwd_flag VARCHAR COMMENT 'Store and forward flag',
    pu_location_id INTEGER COMMENT 'Pick-up location ID',
    do_location_id INTEGER COMMENT 'Drop-off location ID',
    payment_type BIGINT COMMENT 'Payment type',
    fare_amount DOUBLE COMMENT 'Fare amount in dollars',
    extra DOUBLE COMMENT 'Extra charges',
    mta_tax DOUBLE COMMENT 'MTA tax amount',
    tip_amount DOUBLE COMMENT 'Tip amount',
    tolls_amount DOUBLE COMMENT 'Tolls amount',
    improvement_surcharge DOUBLE COMMENT 'Improvement surcharge',
    total_amount DOUBLE COMMENT 'Total amount charged',
    congestion_surcharge DOUBLE COMMENT 'Congestion surcharge',
    airport_fee DOUBLE COMMENT 'Airport fee'
) 
COMMENT 'Table containing NYC TLC Trip Record Data'
WITH (format = 'PARQUET');
```
Then let's insert a sample row...

```sql
INSERT INTO 
  datalake.warehouse.yellow_tripdata 
VALUES
 (
    1,
    TIMESTAMP '2024-04-01 00:02:40',
    TIMESTAMP '2024-04-01 00:30:42',
    0,
    5.2,
    1,
    'N',
    161,
    7,
    1,
    29.6,
    3.5,
    0.5,
    8.65,
    0.0,
    1.0,
    43.25,
    2.5,
    0.0
);
```
... and query the table to check it was successfully written:

```sql
SELECT * 
FROM
  datalake.warehouse.yellow_tripdata; 
```

## 7. Clean Up

This step is optional but should you need to cleanup your data lake, here's your one-stop-shop:

```sql
DROP TABLE datalake.warehouse.yellow_tripdata;
DROP SCHEMA datalake.warehouse;
DROP CATALOG datalake;
```