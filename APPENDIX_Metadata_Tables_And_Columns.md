# Metadata Tables and Columns

Matedata tables allow you to explore metadata specific to each table. Trino exposes them as `"tablename$<property>"`. Below are a few examples:

Show all snaphots of a particular table:
```sql
SELECT * FROM datalake.warehouse."customers$snapshots";
```

Show all files (incl. file-relevant data) holding a data of a particular table:
```sql
SELECT * FROM datalake.warehouse."customers$files";
```

Available Metadata tables are:
- `$properties`
- `$history`
- `$metadata_log_entries`
- `$snapshots`
- `$manifests`
- `$partitions`
- `$files`
- `$refs`

In addition the Iceberg connector automatically exposes path metadata as a hidden column in each table:
- `$path` - Full file system path name of the file for this row
- `$file_modified_time` - Timestamp of the last modification of the file for this row

## Source:
- https://trino.io/docs/current/connector/iceberg.html#metadata-tables
- https://trino.io/docs/current/connector/iceberg.html#metadata-columns