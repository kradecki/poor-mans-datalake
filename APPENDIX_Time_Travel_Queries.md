# Time Travel Queries

```sql
SELECT * FROM datalake.warehouse."customers$history";
```
The query above yields:
```
            made_current_at            |     snapshot_id     |      parent_id      | is_current_ancestor 
---------------------------------------+---------------------+---------------------+---------------------
 2024-07-09 20:30:51.473 Europe/Warsaw |  759478747922295915 |                NULL | true                
 2024-07-09 20:31:04.797 Europe/Warsaw | 2446064313583682769 |  759478747922295915 | true                
 2024-07-09 20:36:50.996 Europe/Warsaw | 3275283822565654789 | 2446064313583682769 | true                
 2024-07-09 21:51:43.388 Europe/Warsaw | 3286450955092918080 | 3275283822565654789 | true 
 ```

 Using one of the `snapshot_id` we can examine the state of a table as of that particular snapshot:

 ```sql
SELECT * 
FROM datalake.warehouse.customers 
FOR VERSION AS OF 2446064313583682769;
 ```

You can also query the table as it was at a certain point in time in the past. The yielded state will reflect the table "as if it was queried at that time".

 ```sql
SELECT *
FROM datalake.warehouse.customers
FOR TIMESTAMP AS OF TIMESTAMP '2024-07-09 20:30:00';
 ```

## Source: 
- https://trino.io/docs/current/connector/iceberg.html#time-travel-queries