## Query Lifecycle in Trino-Nessie-Minio-Iceberg Architecture for

1. **Query Execution Start**:
    - Trino receives the SQL query `SELECT * FROM table`.

2. **Catalog Lookup**:
    - Trino uses the Iceberg catalog to locate the table definition and metadata.
    - The catalog points to Nessie for the latest table state.

3. **Nessie Interaction**:
    - Trino queries Nessie to get the current metadata location for the table.
    - Nessie returns the commit ID and the path to the latest metadata file (e.g., `metadata.json`).

4. **Metadata Retrieval**:
    - Trino accesses the metadata file in Minio (using the path provided by Nessie).
    - The metadata file contains information about schema, partitioning, and the location of manifest lists.

5. **Manifest List Loading**:
    - Trino reads the manifest list file(s) from Minio (e.g., `manifest_list.avro`).
    - The manifest list points to the specific manifest files that contain detailed information about data files.

6. **Manifest File Reading**:
    - Trino reads the manifest file(s) from Minio (e.g., `manifest.avro`).
    - The manifest files list all the data files (e.g., `datafile.parquet`) and relevant metadata such as file size, row count, and partition information.

7. **Data File Access**:
    - Trino accesses the data files from Minio (e.g., `datafile.parquet`).
    - The data files contain the actual table data that matches the schema and partitions specified in the metadata and manifests.

8. **Query Processing**:
    - Trino processes the data files according to the query.
    - It reads the data from the Parquet files, applies any filters, projections, or aggregations as needed.

9. **Result Compilation**:
    - Trino compiles the results of the query.
    - The data is gathered from all the data files, processed, and formatted as per the SQL query requirements.

10. **Query Results Return**:
    - Trino returns the final result set to the user/client who executed the `SELECT * FROM table`.