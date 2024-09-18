# Vacuum
* Predictive optimization: Predictive optimization removes the need to manually manage maintenance operations for Unity Catalog managed tables on Azure Databricks. With predictive optimization enabled, Azure Databricks automatically identifies tables that would benefit from maintenance operations (Vacuum and Optimize) and runs them for the user.
* Predictive optimization does not work on External tables.
* VACUUM might leave behind empty directories after removing all files from within them. Subsequent VACUUM operations delete these empty directories.
* VACUUM removes all files from directories not managed by Delta Lake, ignoring directories beginning with _ or .. If you are storing additional metadata like Structured Streaming checkpoints within a Delta table directory, use a directory name such as _checkpoints.
* Data for change data feed is managed by Delta Lake in the *_change_data* directory and removed with VACUUM (even though it uses a _, but it is predifined)
* Bloom filter indexes use the *_delta_index* directory managed by Delta Lake. VACUUM cleans up files in this directory. Same as above
* When disk caching is enabled, a cluster might contain data from Parquet files that have been deleted with VACUUM. Therefore, it may be possible to query the data of previous table versions whose files have been deleted. Restarting the cluster will remove the cached data
* VACUUM table_name DRY RUN    -- do dry run to get the list of files to be deleted
* VACUUM table_name RETAIN 100 HOURS  -- vacuum files not required by versions more than 100 hours old
* VACUUM for  metadata-only deletes:
    * Soft-deletes:  Soft-deletes do not rewrite data or delete data files, but rather use metadata files to indicate that some data values have changed
    * The *REORG TABLE* command provides the *APPLY (PURGE)* syntax to rewrite data to apply soft-deletes
    * Operations that create soft-deletes in Delta Lake include the following:
        * Dropping columns with column mapping enabled.
        * Deleting rows with deletion vectors enabled.
        * Any data modifications on Photon-enabled clusters when deletion vectors are enabled.
    * With soft-deletes enabled, old data may remain physically present in the table’s current files even after the data has been deleted or updated. To remove this data physically from the table, complete the following steps:
        * Run REORG TABLE ... APPLY (PURGE). After doing this, the old data is no longer present in the table’s current files, but it is still present in the older files that are used for time travel.
        * Run VACUUM to delete these older files (of course when the retention duration is expired).
* Cluster Size for Vacuum:
    * To select the correct cluster size for VACUUM, it helps to understand that the operation occurs in two phases:
        * The job begins by using all available executor nodes to list files in the source directory in parallel. This list is compared to all files currently referenced in the Delta transaction log to identify files to be deleted. The driver sits idle during this time.
        * The driver then issues deletion commands for each file to be deleted. File deletion is a driver-only operation, meaning that all operations occur in a single node while the worker nodes sit idle.
    * To optimize cost and performance, Databricks recommends the following, especially for long-running vacuum jobs:
        * Run vacuum on a cluster with auto-scaling set for 1-4 workers, where each worker has 8 cores.
        * Select a driver with between 8 and 32 cores. Increase the size of the driver to avoid out-of-memory (OOM) errors.
        * If VACUUM operations are regularly deleting more than 10 thousand files or taking over 30 minutes of processing time, you might want to increase either the size of the driver or the number of workers.
* How frequently should you run vacuum? Based on your business requirements of timetravel duration. But the lesser the frequency, the higher will be the storage cost
* To run vacuum on a table for lesst than the retention duration, we need to disable the below safety config

     *spark.databricks.delta.retentionDurationCheck.enabled to false.*

# Optimize:
* Bin-packing optimization is idempotent, meaning that if it is run twice on the same dataset, the second run has no effect.
* Bin-packing aims to produce evenly-balanced data files with respect to their size on disk, but not necessarily number of tuples per file. However, the two measures are most often correlated. (on the other hand, Z-ordering aims to produce evenly-balanced data files with respect to the number of tuples, but not necessarily data size on disk)
* Optimize type : Bin-packing and Z-ordering. ( Bin Packing is defined as the placement of a set of different-sized items into identical bins so that the number of bins used is minimized) 
  
# Liquid Clustering:
* LC dbricks design doc https://docs.google.com/document/d/1FWR3odjOw4v4-hjFy_hVaNdxHVs4WuK1asfB6M6XEMw/edit
* Delta Lake liquid clustering replaces table partitioning and ZORDER to simplify data layout decisions and optimize query performance.
* You can say LC is an enhanced Zorder. LC uses an optimized hilbert curve and does better data layout then Zorder (this is not the only diff though)
* Why is the Z-curve worse than Hilbert? The HiBin-packing aims to produce evenly-balanced data files with respect to their size on disk, but not necessarily number of tuples per file. However, the two measures are most often correlated.Hillbert curve gives us a nice property that adjacent points on the curve always have a distance of 1.The Z-curve doesn’t have this
property. Z-curve’s adjacent points don’t always have distance = 1 and it has large jumps in the curve.
* Also, everytime you do optimization with zorder, it rewrites the whole table (even if no new files have been
added since the last time). And you have to keep on doing this optimiziation as after a few duration query will become slow. So its like a seesaw with zorder- full rewtrite-good performance-performance start to hit- full rewrite. LC avoids full rewrite. A stable Zcube will never get re-optimized 
* Liquid clustering provides flexibility to redefine clustering keys without rewriting existing data (e.g if you want to change the partition columns in a patiotoned table, you need to rewrite the whole table), allowing data layout to evolve alongside analytic needs over time.
* Databricks recommends using Databricks Runtime 15.2 and above for all tables with liquid clustering enabled.
* Row-level concurrency is generally available in Databricks Runtime 14.2 and above for all tables with deletion vectors enabled.
* The following are examples of scenarios that benefit from Liquid Clustering. So if hive style partition works fine and does not cause any of the below issues
    * Tables often filtered by high cardinality columns.
    * Tables with significant skew in data distribution.
    * Tables that grow quickly and require maintenance and tuning effort.
    * Tables with concurrent write requirements.
    * Tables with access patterns that change over time.
    * Tables where a typical partition key could leave the table with too many or too few partitions.
* You can enable liquid clustering on an existing table or during table creation.
* Clustering is not compatible with partitioning or ZORDER, and requires that you use Azure Databricks to manage all layout and optimization operations for data in your table.
* After liquid clustering is enabled, run OPTIMIZE jobs as usual to incrementally cluster data, meaning that data is only rewritten as necessary to accommodate data that needs to be clustered (distributed and sorted). Data files with clustering keys that do not match data to be clustered are not rewritten.
* For tables experiencing many updates or inserts, Databricks recommends scheduling an OPTIMIZE job every one or two hours. Because liquid clustering is incremental, most OPTIMIZE jobs for clustered tables run quickly.
* Enabling LC:  CREATE TABLE table1(col0 int, col1 string) CLUSTER BY (col0); *alter* ALTER TABLE <table_name> CLUSTER BY (<clustering_columns>)
* In upcoming release, *cluster by auto* will be available which will pick the cluster columns automatically based on the workload
* **Warning** :Tables created with liquid clustering enabled have numerous Delta table features enabled at creation and use Delta writer version 7 and reader version 3. Table protocol versions cannot be downgraded, and tables with clustering enabled are not readable by Delta Lake clients that do not support all enabled Delta reader protocol table features.
* Choose clustering keys: Databricks recommends choosing clustering keys based on commonly used query filters. Clustering keys can be defined in any order. If two columns are correlated, you only need to add one of them as a clustering key. You can specify up to 4 columns as clustering keys. You can only specify columns with statistics collected for clustering keys.
* If you’re converting an existing table, consider the following recommendations:
  ![image](https://github.com/user-attachments/assets/c5888911-a8b4-4484-a20f-b6cd06eda115)
* Clustering on write only gets triggered when data in the transaction meets a size threshold:
  ![image](https://github.com/user-attachments/assets/ade8d98b-6e84-47ca-8fd6-e84446e0263d)
* Because not all operations apply (trigger) liquid clustering, Databricks recommends frequently running OPTIMIZE to ensure that all data is efficiently clustered.
* You can change clustering keys for a table at any time by running an ALTER TABLE command. When you change clustering keys, subsequent OPTIMIZE and write operations use the new clustering approach, but existing data is *not rewritten*.
* The clustering columns are persisted in *AddFile* using the ZCUBE_ZORDER_BY tag to indicate which clustering columns these files are clustered to. When picking candidate files, dbricks filter out files with a different set of clustering columns.
* LC does not allow dropping clustering columns, but users can always do ALTER TABLE CLUSTER BY to move columns out of the clustering columns and then drop them.
* You can also turn off clustering by setting the keys to NONE, as in the following example: ALTER TABLE table_name CLUSTER BY NONE;
* Setting cluster keys to NONE does not rewrite data that has already been clustered, but prevents future OPTIMIZE operations from using clustering keys.
* Limitations:
    * In Databricks Runtime 15.1 and below, clustering on write does not support source queries that include filters, joins, or aggregations.
    * Structured Streaming workloads do not support clustering-on-write.
    * You cannot create a table with liquid clustering enabled using a Structured Streaming write. You can use Structured Streaming to write data to an existing table with liquid clustering enabled.
* *Even though Liquid clustering looks great, but do not simply replace the hive-based partition strategy by https://www.reddit.com/r/databricks/comments/1cyj7cz/worse_performance_of_liquid_clustering_vs/*
* Problem with Partition tables: Hard to configure correctly
     * over partitioning- Too many small files
     * under partitioning- NO skipping benifits
     * Skewed Partition

Even if you setup correctly, in real life you will still end up with too huge files or too many small files per partition. And you have to run optimize to get better performance. ButIf you can setup the partition table correctly- then yeah- go for it!!  
* Partition vs LC quick comp
  ![image](https://github.com/user-attachments/assets/39b1bd98-56dc-45e5-ba00-aa690fe8a133)

* Concurrency used to work only in partitioned table and you have to add a filter condition saying which partition a particular merge would be running. In LC, concurrency works from out of the box- no filter column needed
* DBR 14.2+ allows row level locking- so better concurrency
**Making merge faster with LC (https://www.youtube.com/watch?v=yZmrpXJg-G8)**
* Key optimization in Merge:
     * LC (Cluster By the merge keys using LC)
     * Deletion Vector
     * Dynamic File prunning (DFP)
     * Bloom filter join: Create a bloom filter (bitmap) from source table (with the joined columns' values) and share it accross the worker nodes which are reading the target files. Then this filter selects only the rows that will have the maching rows (values) and those only gets transfered over network(shuffled) for final joining
* cluster by (merge_keys)- this enables DFP and *turns on deletion vector automatically!!*

**How merge works**
* Find the files of the target table that matches the keys
* join these particular files on the kesy with source tables and apply the changes (updates,deletes)
* Commit changes- here the older matcing files will be deleted as new files gets created with insreted records along with the updates and deleted records.

 **answers from databricks communty**
* Liquid Clustering and Z-Ordering both use 100 GB ZCubes but differ in their optimization and performance characteristics. Liquid Clustering maintains ZCube IDs in the transaction log and optimizes data only within unclustered ZCubes, making it efficient for write-heavy operations with minimal reorganization. In contrast, Z-Ordering does not track ZCube IDs and reorganizes the entire table or partitions during optimization, which can result in heavier write operations but may offer better read performance. Liquid Clustering is ideal for scenarios with frequent updates, while Z-Ordering is suited for read-heavy workloads.
* Suppose if we receive incremental data with some modifications to existing record and in this case whether existing clustered ZCubes will be re-organized again? : Technically, due to Delta Lake's history, on file-level you don't update existing files, you always create new ones. Hence already clusterd zcubes will not be clustered
  *Now if we keep on receiving update/deletes for existing data, the size of the clustered zcubes will keep on decreasing resulting into changing the clustered zcube to partial zcubes* (
zcubes can be partial of stable depending on min_cube_size (100 gb) config. zcubes target size (best-effort) can be tuned using target_cube_size(150gb). stable zcubes can be downgraded to partial if their size decrease to less than min_cube_size by delete (vacuum) operation)
* LC improved query efficiency, but increased no of files. Why?
     * Liquid clustering uses the Hilbert Curve, a continuous fractal space-filling curve, as a multi-dimensional clustering technique. It significantly improves data skipping over traditional ZORDER.
     * Instead of partitioning data into fixed-size blocks (as in ZORDER), liquid clustering creates groups of files with Hilbert-clustered data, known as ZCubes. These ZCubes are produced by the same OPTIMIZE command.
     * The file count can get icreased- this behaviour is expected due to the way liquid clustering works. When you optimize a table, it reorganizes the data into ZCubes(first time it will reorganize whole table- so there can be a sudden jump in file nos). Some files may be split or merged to form these ZCubes.
     * If your data had skewed distributions (e.g., some values are more frequent than others), liquid clustering might create more files to evenly distribute the data.
     * Despite the increased file count, query performance should improve due to better data skipping and locality.
 
# Cloning in Delta Table:
* Create a copy of an existing Delta Lake table on Azure Databricks at a specific version using the clone command
* Clones can be either deep or shallow.
* A deep clone is a clone that copies the source table data to the clone target in addition to the metadata of the existing table. Additionally, stream metadata is also cloned such that a stream that writes to the Delta table can be stopped on a source table and continued on the target of a clone from where it left off. For deep clones only, stream and COPY INTO metadata are also cloned. Metadata not cloned are the table Shallow clones reference data files in the source directory. If you run vacuum on the source table, clients can no longer read the referenced data files and a FileNotFoundException is thrown. In this case, running clone with replace over the shallow clone repairs the clone. If this occurs often, consider using a deep clone instead which does not depend on the source table. and user-defined commit metadata.
* A shallow clone is a clone that does not copy the data files to the clone target. The table metadata is equivalent to the source. These clones are cheaper to create. The metadata that is cloned includes: schema, partitioning information, invariants, nullability.
* Any changes made to either deep or shallow clones affect only the clones themselves and not the source table.
* Shallow clones reference data files in the source directory. If you run vacuum on the source table, clients can no longer read the referenced data files and a FileNotFoundException is thrown. In this case, running clone with replace over the shallow clone repairs the clone. If this occurs often, consider using a deep clone instead which does not depend on the source table.
* Cloning with *replace* to a target that already has a table at that path creates a Delta log if one does not exist at that path. You can clean up any existing data by running vacuum.
* For existing Delta tables, a new commit is created that includes the new metadata and new data from the source table. This new commit is incremental, meaning that only new changes since the last clone are committed to the table.
* Syntax:
   * CREATE TABLE target_table CLONE source_table; -- Create a deep clone of source_table as target_table
   * CREATE OR REPLACE TABLE target_table CLONE source_table; -- Replace the target
   * CREATE TABLE IF NOT EXISTS target_table CLONE source_table; -- No-op if the target table exists
   * CREATE TABLE target_table SHALLOW CLONE source_table;
   * CREATE TABLE target_table SHALLOW CLONE source_table VERSION AS OF version;
   * CREATE TABLE target_table SHALLOW CLONE source_table TIMESTAMP AS OF timestamp_expression; -- timestamp can be like “2019-01-01” or like date_sub(current_date(), 1)
* You can sync deep clones incrementally to maintain an updated state of a source table for disaster recovery (check more on it)
* You can only clone Unity Catalog managed tables to Unity Catalog managed tables and Unity Catalog external tables to Unity Catalog external tables. VACUUM behavior differs between managed and external tables.
* For managed UC tables, VACUUM operations against either the source or target of a shallow clone operation might delete data files from the source table.
* For external tables, VACUUM operations only remove data files from the source table when run against the source table.
* Only data files not considered valid for the source table or any shallow clone against the source are removed.
* If multiple shallow clones are defined against a single source table, running VACUUM on any of the cloned tables does not remove valid data files for other cloned tables.
* You cannot share shallow clones using Delta Sharing(In UC).
* You cannot nest shallow clones, meaning you cannot make a shallow clone from a shallow clone.
* For managed tables, dropping the source table breaks the target table for shallow clones. Data files backing external tables are not removed by DROP TABLE operations, and so shallow clones of external tables are not impacted by dropping the source.
* Unity Catalog allows users to UNDROP managed tables for around 7 days after a DROP TABLE command. In Databricks Runtime 13.3 LTS and above, managed shallow clones based on a dropped managed table continue to work during this 7 day period. If you do not UNDROP the source table in this window, the shallow clone stops functioning once the source table’s data files are garbage collected.

# Data Skipping:
* delta.dataSkippingNumIndexedCols: Increase or decrease the number of columns on which Delta collects statistics. Depends on column order
* delta.dataSkippingStatsColumns: Specify a list of column names for which Delta Lake collects statistics. Supersedes dataSkippingNumIndexedCols.
* Updating this property does not automatically recompute statistics for existing data. Rather, it impacts the behavior of future statistics collection when adding or updating data in the table.
* In Databricks Runtime 14.3 LTS and above, you can manually trigger the recomputation of statistics for a Delta table using the following command:
     *ANALYZE TABLE table_name COMPUTE DELTA STATISTICS*
* Long strings are truncated during statistics collection. You might choose to exclude long string columns from statistics collection, especially if the columns aren’t used frequently for filtering queries.

# Change Data Feed (Go through while learning staructured streaming)

# Constraints on Az Databricks:
* Azure Databricks supports standard SQL constraint management clauses. All constraints on Azure Databricks require Delta Lake (Delta Live Tables has a similar concept known as expectations). Constraints fall into two categories:
     * Enforced contraints ensure that the quality and integrity of data added to a table is automatically verified.
     * Informational primary key and foreign key constraints encode relationships between fields in tables and are not enforced.
* **Enforced contraints**: When a constraint is violated, the transaction fails with an error. Two types of constraints are supported:
     * NOT NULL: indicates that values in specific columns cannot be null.
         * CREATE TABLE people10m (id INT NOT NULL, ssn STRING, salary INT);
         * ALTER TABLE people10m ALTER COLUMN middleName DROP NOT NULL;
         * ALTER TABLE people10m ALTER COLUMN ssn SET NOT NULL;
         * Before adding a NOT NULL constraint to a table, Azure Databricks verifies that all existing rows satisfy the constraint.
         * If you specify a NOT NULL constraint on a column nested within a struct, the parent struct must also be not null. Columns nested within array or map types do not accept NOT NULL constraints.
     * CHECK: indicates that a specified boolean expression must be true for each input row.
         * You manage CHECK constraints using the ALTER TABLE ADD CONSTRAINT and ALTER TABLE DROP CONSTRAINT commands. ALTER TABLE ADD CONSTRAINT verifies that all existing rows satisfy the constraint before adding it to the table.
         *  ALTER TABLE people10m ADD CONSTRAINT dateWithinRange CHECK (birthDate > '1900-01-01');

            ALTER TABLE people10m DROP CONSTRAINT dateWithinRange;
* **PK and FKs**:
     * Primary key and foreign key constraints are available in Databricks Runtime 11.3 LTS and above, and are fully GA in Databricks Runtime 15.2 and above.
     * Primary key and foreign key constraints require Unity Catalog and Delta Lake. Primary and foreign keys are informational only and are not enforced. Foreign keys must reference a primary key in another table.
     *  Create a table with a primary key
        > CREATE TABLE persons(first_name STRING NOT NULL, last_name STRING NOT NULL, nickname STRING,
                       CONSTRAINT persons_pk PRIMARY KEY(first_name, last_name));

        create a table with a foreign key
        > CREATE TABLE pets(name STRING, owner_first_name STRING, owner_last_name STRING,
                    CONSTRAINT pets_persons_fk FOREIGN KEY (owner_first_name, owner_last_name) REFERENCES persons);

        Create a table with a single column primary key and system generated name
        > CREATE TABLE customers(customerid STRING NOT NULL PRIMARY KEY, name STRING);

        Create a table with a names single column primary key and a named single column foreign key
        > CREATE TABLE orders(orderid BIGINT NOT NULL CONSTRAINT orders_pk PRIMARY KEY,
                      customerid STRING CONSTRAINT orders_customers_fk REFERENCES customers);

# Delta Best Practices:
* When deleting and recreating a table in the same location, you should always use a CREATE OR REPLACE TABLE statement as dropping-recreating can result in unexpected results for concurrent operations.
* In *Create Or Replace* the table history is maintained during the atomic data replacement, concurrent transactions can validate the version of the source table referenced, and therefore fail or reconcile concurrent transactions as necessary without introducing unexpected behavior or results.
* DROP TABLE has different semantics depending on the type of table and whether the table is registered to Unity Catalog or the legacy Hive metastore.
  ![image](https://github.com/user-attachments/assets/5344fb8d-609b-4c59-979c-487e0fa651cd)
*  Unity Catalog maintains a history of Delta tables using an internal table ID
*  CREATE OR REPLACE TABLE has the same semantics regardless of the table type or metastore in use. The following are important advantages of CREATE OR REPLACE TABLE:
     * Table contents are replaced, but the table identity is maintained.
     * The table history is retained, and you can revert the table to an earlier version with the RESTORE command.
     * The operation is a single transaction, so there is never a time when the table doesn’t exist.
     * Concurrent queries reading from the table can continue without interruption. Because the version before and after replacement still exists in the table history, concurrent queries can reference either version of the table as necessary.
* Replace the content or schema of a table:
     * Sometimes you may want to replace a Delta table. For example:You discover the data in the table is incorrect and want to replace the content. Or You want to rewrite the whole table to do incompatible schema changes
     * While you can delete the entire directory of a Delta table and create a new table on the same path, it’s not recommended because:
          * Deleting a directory is not efficient. A directory containing very large files can take hours or even days to delete.
          * You lose all of the content in the deleted files; it’s hard to recover if you delete the wrong table.
          * The directory deletion is not atomic. While you are deleting the table a concurrent query reading the table can fail or see a partial table.
     * If you don’t need to change the table schema, you can delete data from a Delta table and insert your new data, or update the table to fix the incorrect values. Deletion removes the data from the latest version of the Delta table but does not remove it from the physical storage until the old versions are explicitly vacuumed.
     * If you want to change the table schema, you can replace the whole table atomically. Delta lake supports the following schema changes: Adding new columns (at arbitrary positions),Reordering existing columns,Renaming existing columns
 
       REPLACE TABLE <your-table> AS SELECT ... -- Managed table (replace as in create or replace)

       REPLACE TABLE <your-table> LOCATION "<your-table-path>" AS SELECT ... -- External table
     * There are multiple benefits with this approach:
          * Overwriting a table is much faster because it doesn’t need to list the directory recursively or delete any files.
          * The old version of the table still exists. If you delete the wrong table you can easily retrieve the old data using time travel. See Work with Delta Lake table history.
          * It’s an atomic operation. Concurrent queries can still read the table while you are deleting the table.
          * Because of Delta Lake ACID transaction guarantees, if overwriting the table fails, the table will be in its previous state.

* Spark caching: Databricks does not recommend that you use Spark caching for the following reasons:
     * You lose any data skipping that can come from additional filters added on top of the cached DataFrame.
     * The data that gets cached might not be updated if the table is accessed using a different identifier.

* Differences between Delta Lake and Parquet on Apache Spark:
     * REFRESH TABLE: Delta tables always return the most up-to-date information, so there is no need to call REFRESH TABLE manually after changes.
     * Add and remove partitions: Delta Lake automatically tracks the set of partitions present in a table and updates the list as data is added or removed. As a result, there is no need to run ALTER TABLE [ADD|DROP] PARTITION or MSCK for delta, but needed for parquet. When creating a table using PARTITIONED BY clause, partitions are generated and registered in the Hive metastore. However, if the partitioned table is created from existing data, partitions are not registered automatically in the Hive metastore. User needs to run REPAIR TABLE to register the partitions. REPAIR TABLE on a non-existent table or a table without partitions throws an exception. Remember, this is only when you create a partitioned table from an existing data. Table repair is not required when you ingest data to partiotioned table, e.g inserting into a parquet paritioned tables(insertts as partitioned lets say). But if you drop the table and recreate it, run msck repair

* Improve performance for Delta Lake merge:
    * Reduce the search space for matches
    * Compact files (optimize)
    * Control the shuffle partitions for writes: The merge operation shuffles data multiple times to compute and write the updated data. The number of tasks used to shuffle is controlled by the Spark session configuration spark.sql.shuffle.partitions. Setting this parameter not only controls the parallelism but also determines the number of output files. Increasing the value increases parallelism but also generates a larger number of smaller data files.
    *  For partitioned tables, merge can produce a much larger number of small files than the number of shuffle partitions. This is because *every shuffle task can write multiple files in multiple partitions*, and can become a performance bottleneck. You can reduce the number of files by enabling optimized writes
    *  Low Shuffle Merge: Low Shuffle Merge provides an optimized implementation of MERGE that provides better performance for most common workloads.In addition, it preserves existing data layout optimizations such as Z-ordering on unmodified data.
*  data recency : At the beginning of each query, Delta tables auto-update to the latest version of the table. This process can be observed in notebooks when the command status reports: Updating the Delta table's state. However, when running historical analysis on a table, you may not necessarily need up-to-the-last-minute data, especially for tables where streaming data is being ingested frequently. In these cases, queries can be run on stale snapshots of your Delta table. This approach can lower latency in getting results from queries. You can configure tolerance for stale data by setting the Spark session configuration spark.databricks.delta.stalenessLimit with a time string value such as 1h or 15m. This configuration is session specific, and doesn’t affect other clients accessing the table. If the table state has been updated within the staleness limit, a query against the table returns results without waiting for the latest table update. This setting never prevents your table from updating, and when stale data is returned, the update processes in the background. If the last table update is older than the staleness limit, the query does not return results until the table state update completes.
* Manage column-level statistics in checkpoints (e.g one check point after 10 json delta log):
     * You manage how statistics are written in checkpoints using the table properties delta.checkpoint.writeStatsAsJson and delta.checkpoint.writeStatsAsStruct. If both table properties are false, Delta Lake cannot perform data skipping.
     * Column-level statistics are stored as a struct and a JSON (for backwards compatibility). The struct format makes Delta Lake reads much faster, because:
         * Delta Lake doesn’t perform expensive JSON parsing to obtain column-level statistics.
         * Parquet column pruning capabilities significantly reduce the I/O required to read the statistics for a column.
# Table protocol
* Every Delta table has a protocol specification which indicates the set of features that the table supports. The protocol specification is used by applications that read or write the table to determine if they can handle all the features that the table supports. If an application does not know how to handle a feature that is listed as supported in the protocol of a table, then that application is not be able to read or write that table (read protocol and the write protocol).
* When you enable Delta features on a table, the table protocol is automatically upgraded. Databricks recommends against changing the minReaderVersion and minWriterVersion table properties. Changing these table properties does not prevent protocol upgrade. Setting these values to a lower value does not downgrade the table.
* Azure Databricks provides limited support for dropping table features. *Check on this whenever needed*

# Table Size:
  * The table size reported for tables backed by Delta Lake on Azure Databricks differs from the total size of corresponding file directories in cloud object storage
  * Table sizes reported in Azure Databricks through UIs and DESCRIBE commands refer to the total size of data files on disk for those files referenced in the current version of the Delta table.
  * The size of the table reported after OPTIMIZE is generally smaller than the size before OPTIMIZE runs, because the total size of data files referenced by the current table version decreases with data compaction.

# Some notes on Merge:
* There can be any number of whenMatched and whenNotMatched clauses.
* whenMatched clauses can have at most one update and one delete action.
* Each whenMatched clause can have an optional condition. If this clause condition exists, the update or delete action is executed for any matching source-target row pair only when the clause condition is true.
* If there are multiple whenMatched clauses, then they are evaluated in the order they are specified. *All whenMatched clauses, except the last one, must have conditions*.
* whenNotMatched clauses can have only the insert action
* If there are multiple whenNotMatched clauses, then they are evaluated in the order they are specified. All whenNotMatched clauses, except the last one, must have conditions.
* whenNotMatchedBySource clauses are executed when a target row does not match any source row based on the merge condition. e.g use case: you need to delete records from target table if the expected records did not come from source.
* whenNotMatchedBySource clauses can specify delete and update actions.
* If there are multiple whenNotMatchedBySource clauses, then they are evaluated in the order they are specified. All whenNotMatchedBySource clauses, except the last one, must have conditions.
* (This for *update and not for delete*) By definition, whenNotMatchedBySource clauses do not have a source row to pull column values from, and so source columns can’t be referenced. For each column to be modified, you can either specify a literal or perform an action on the target column, such as SET target.deleted_count = target.deleted_count + 1.
* A merge operation can fail if multiple rows of the source dataset match and the merge attempts to update the same rows of the target Delta table. According to the SQL semantics of merge, such an update operation is ambiguous as it is unclear which source row should be used to update the matched target row.

# Delta table properties reference:
* Delta Lake reserves Delta table properties starting with delta.
* All operations that set or update table properties conflict with other concurrent write operations, causing them to fail. Databricks recommends you modify a table property only when there are no concurrent write operations on the table.
* **How do table properties and SparkSession properties interact?**
     * Delta Lake reserves Delta table properties starting with delta.
     * Delta table properties are set per table. If a property is set on a table, then this is the setting that is followed by default.
     * Some table properties have associated SparkSession configurations which always take precedence over table properties. Some examples include the spark.databricks.delta.autoCompact.enabled and spark.databricks.delta.optimizeWrite.enabled configurations, which turn on auto compaction and optimized writes at the SparkSession level rather than the table level. Databricks recommends using table-scoped configurations for most workloads.
     * For every Delta table property you can set a default value for new tables using a SparkSession configuration, overriding the built-in default. This setting only affects new tables and does not override or replace properties set on existing tables. The prefix used in the SparkSession is different from the configurations used in the table properties, as shown in the following table:
       ![image](https://github.com/user-attachments/assets/ce3d2a1a-dbed-40b5-acfd-f4d49f231f75)
     * Refer https://learn.microsoft.com/en-us/azure/databricks/delta/table-properties for available delta properties

* AutoCompat, OptimizeWrite, Optimise (Auto-optimize (retired term now) meant autocompact and optimizewrite):
     * Optimized Write combines many small writes to the same partition into one larger write operation. It is an optimization performed before the data is written to your Delta table(so describe hist will not show it). Auto Compaction combines many small files into larger, more efficient files. You may need to perform a VACUUM operation afterwards to clean up the remaining small files.
     * Optimized writes are most effective for partitioned tables, as they reduce the number of small files written to each partition by adding another layer of shuffles. Auto optimize will try to create files of 128 MB within each partition. On the other hand, explicit optimize will compress more and create files of 1 GB each (default value, params: spark.databricks.delta.optimize.maxFileSize). The 1 GB default file size was selected from years of customer usage showing that this file size works well on common computational instances.
     * You might have code that runs coalesce(n) or repartition(n) just before you write out your data to control the number of files written. Optimized writes eliminates the need to use this pattern.
     * Auto Compaction is only triggered for partitions or tables that have at least a certain number of small files. *spark.databricks.delta.autoCompact.minNumFiles*. Default 50. Default o/p file size 128 mb.
     * You can control the output file size by setting the Spark configuration spark.databricks.delta.autoCompact.maxFileSize
     * Auto compaction occurs after a write to a table has succeeded and runs synchronously on the cluster that has performed the write. So the write time could increase. Auto compaction only compacts files that haven’t been compacted previously.
     * Why need auto compaction if optimize write was used?  When you are writing frequent small updates to a table. In this case, the files will still be small, even after an Optimized Write.
     * ![image](https://github.com/user-attachments/assets/10a58946-7d51-40cc-8059-81ab1e68653d)
     * In Databricks Runtime 10.4 LTS and above, auto compaction and optimized writes are always enabled for MERGE, UPDATE, and DELETE operations. You cannot disable this functionality.

# Generated columns:
* This feature is in Public Preview as of 12 sep,2024.
* Delta Lake supports generated columns which are a special type of column whose values are automatically generated based on a user-specified function over other columns in the Delta table.
* When you write to a table with generated columns and you do not explicitly provide values for them, Delta Lake automatically computes the values. However, if you explicitly provide values for them, the values must satisfy the constraint (<value> <=> <generation expression>) IS TRUE or the write will fail with an error.
* Syntax
     * SQL : CREATE TABLE default.people10m (id INT, firstName STRING, birthDate TIMESTAMP, dateOfBirth DATE GENERATED ALWAYS AS (CAST(birthDate AS DATE)))
     * Python: DeltaTable.create(spark) \  .tableName("default.people10m") \  .addColumn("id", "INT") \  .addColumn("firstName", "STRING")   .addColumn("birthDate", "TIMESTAMP") \  .addColumn("dateOfBirth", DateType(), generatedAlwaysAs="CAST(birthDate AS DATE)") \  .execute()
* Generated columns are stored as if they were normal columns. That is, they occupy storage.
* The following restrictions apply to generated columns:
      * A generation expression can use any SQL functions in Spark that always return the same result when given the same argument values, except the following types of functions:User-defined functions,Aggregate functions,Window functions,Functions returning multiple rows
* If a generated column is used in the partitioned by clause, then even if you query the table with the base column of the generated column awhere clause, delta lake can still use partion filter in the query even though partitioned column was not used in the predicate. e.g *CREATE TABLE events(
eventId BIGINT,
data STRING,
eventType STRING,
eventTime TIMESTAMP,
eventDate date GENERATED ALWAYS AS (CAST(eventTime AS DATE))
)
PARTITIONED BY (eventType, eventDate)*

   Query: *SELECT * FROM events WHERE eventTime >= "2020-10-01 00:00:00" <= "2020-10-01 12:00:00"*

   Delta Lake automatically generates a partition filter so that the preceding query only reads the data in partition          date=2020-10-01 even if a partition filter (eventDate) is not specified (since eventDate is derived from eventTime)
* If we change the generation expression, then schema needs to be overwrite ("overwriteSchema", "True" in pyspark) as generated column is considered part of the schema.
# Identity columns: 
* Delta Lake identity columns are a type of generated column that assigns unique values for each record inserted into a table.
* Declaring an identity column on a Delta table disables concurrent transactions. Only use identity columns in use cases where concurrent writes to the target table are not required.
* Syntax: CREATE TABLE table_name (
  identity_col BIGINT GENERATED BY DEFAULT AS IDENTITY,
  other_column ...)
* You can optionally specify the following: A starting value and A step size, which can be positive or negative.
* Values assigned by identity columns are unique and increment in the direction of the specified step, and in multiples of the specified step size, but are not guaranteed to be contiguous. For example, with a starting value of 0 and a step size of 2, all values are positive even numbers but some even numbers might be skipped.
* When using the clause GENERATED BY DEFAULT AS IDENTITY, insert operations can specify values for the identity column. Modify the clause to be GENERATED ALWAYS AS IDENTITY to override the ability to manually set values.
* Identity columns only support the BIGINT type, and operations fail if the assigned value exceeds the range supported by BIGINT.
* Limitations:
     * Concurrent transactions are not supported on tables with identity columns enabled.
     * You cannot partition a table by an identity column.
     * You cannot use ALTER TABLE to ADD, REPLACE, or CHANGE an identity column.
     * You cannot update the value of an identity column for an existing record.
     * To change the IDENTITY value for an existing record, you must delete the record and INSERT it as a new record.

# Schema enforcement:
* Schema enforcement does not apply to tables backed by external data (not external tables).
**For Insert**
* Azure Databricks enforces the following rules when inserting data into a table:
     * All inserted columns must exist in the target table.
     * All column data types must match the column data types in the target table.
**For Merge**
* If the data type in the source statement does not match the target column, MERGE tries to safely cast column data types to match the target table.
* The columns that are the target of an UPDATE or INSERT action must exist in the target table.
* When using INSERT * or UPDATE SET * syntax:
     * Columns in the source dataset not present in the target table are ignored
     * The source dataset must have all the columns present in the target table.
 
# File size tuning:
* Autotune file size based on workload: Databricks recommends setting the table property delta.tuneFileSizesForRewrites to true for all tables that are targeted by many MERGE or DML operations, regardless of Databricks Runtime, Unity Catalog, or other optimizations. When set to true, the target file size for the table is set to a much lower threshold, which accelerates write-intensive operations. If not explicitly set, Azure Databricks automatically detects if 9 out of last 10 previous operations on a Delta table were MERGE operations and sets this table property to true. You must explicitly set this property to false to avoid this behavior.
* Autotune file size based on table size:
     * Autotune file size based on table size:To minimize the need for manual tuning, Azure Databricks automatically tunes the file size of Delta tables based on the size of the table. Azure Databricks will use smaller file sizes for smaller tables and larger file sizes for larger tables so that the number of files in the table does not grow too large. Azure Databricks does not autotune tables that you have tuned with a specific target size or based on a workload with frequent rewrites.
     * The target file size is based on the current size of the Delta table. For tables smaller than 2.56 TB, the autotuned target file size is 256 MB. For tables with a size between 2.56 TB and 10 TB, the target size will grow linearly from 256 MB to 1 GB. For tables larger than 10 TB, the target file size is 1 GB.
     * When the target file size for a table grows, existing files are not re-optimized into larger files by the OPTIMIZE command. A large table can therefore always have some files that are smaller than the target size. If it is required to optimize those smaller files into larger files as well, you can configure a fixed target file size for the table using the delta.targetFileSize table property.
* Limit rows written in a data file:
     * Occasionally, tables with narrow data might encounter an error where the number of rows in a given data file exceeds the support limits of the Parquet format. To avoid this error, you can use the SQL session configuration spark.sql.files.maxRecordsPerFile (maxRecordsPerFile for dataframe api) to specify the maximum number of records to write to a single file for a Delta Lake table. Specifying a value of zero or a negative value represents no limit.
     * When maxRecordsPerFile is specified, the value of the SQL session configuration spark.sql.files.maxRecordsPerFile is ignored.
     * Databricks **does not recommend** using this option unless it is necessary to avoid the aforementioned error. This setting might still be necessary for some Unity Catalog managed tables with very narrow data.

# Selectively overwrite data with Delta Lake:
* Azure Databricks leverages Delta Lake functionality to support two distinct options for selective overwrites:
     * The *replaceWhere* option atomically replaces all records that match a given predicate.
     * You can replace directories of data based on how tables are partitioned using dynamic partition overwrites.
     * For most operations, Databricks recommends using replaceWhere to specify which data to overwrite.
* Arbitrary selective overwrite with replaceWhere: You can selectively overwrite only the data that matches an arbitrary expression. The following command atomically replaces events in January in the target table, which is partitioned by start_date, with the data in replace_data:

  INSERT INTO TABLE events REPLACE WHERE start_date >= '2017-01-01' AND end_date <= '2017-01-31' SELECT * FROM replace_data
*Check more on it*

# Schema Update:
* Delta Lake lets you update the schema of a table. The following types of changes are supported:
     * Adding new columns (at arbitrary positions)
     * Reordering existing columns
     * Renaming existing columns
     * You can make these changes explicitly using DDL or implicitly using DML. An update to a Delta table schema is an operation that conflicts with all concurrent Delta write operations.
 * You can change a column’s type or name or drop a column by rewriting the table.

# Enable schema evolution: 
* You can enable schema evolution by doing one of the following:
   * Set the .option("mergeSchema", "true") to a Spark DataFrame write or writeStream operation.
   * Use MERGE WITH SCHEMA EVOLUTION syntax.
   * Set the Spark conf spark.databricks.delta.schema.autoMerge.enabled to true for the current SparkSession.
* Databricks recommends enabling schema evolution for each write operation rather than setting a Spark conf. When you use options or syntax to enable schema evolution in a write operation, this takes precedence over the Spark conf.
* Enable schema evolution for writes to add new columns:
      * Columns that are present in the source query but missing from the target table are automatically added as part of a write transaction when schema evolution is enabled
      * New columns are added to the end of the table schema
      * A column in the target table is not present in the source table. The target schema is left unchanged; the values in the additional target column are either left unchanged (for UPDATE) or set to NULL (for INSERT).
* Merge schema evolution:

   MERGE WITH SCHEMA EVOLUTION INTO target
USING source
ON source.key = target.key
WHEN MATCHED THEN
  UPDATE SET *
WHEN NOT MATCHED THEN
  INSERT *
WHEN NOT MATCHED BY SOURCE THEN
* Dealing with NullType columns in schema updates: Because Parquet doesn’t support NullType, NullType columns are dropped from the DataFrame when writing into Delta tables, but are still stored in the schema. When a different data type is received for that column, Delta Lake merges the schema to the new data type. If Delta Lake receives a NullType for an existing column, the old schema is retained and the new column is dropped during the write.
* By default, overwriting the data in a table does not overwrite the schema. When overwriting a table using mode("overwrite") without replaceWhere, you may still want to overwrite the schema of the data being written. You replace the schema and partitioning of the table by setting the overwriteSchema option to *true*

# Type widening: This feature is in Public Preview in Databricks Runtime 15.2 and above.

* Tables with type widening enabled allow you to change column data types to a wider type without rewriting underlying data files. 
* ![image](https://github.com/user-attachments/assets/5e750a9e-e202-43f5-bce7-23a7fa55f86b)
* You can enable type widening on an existing table by setting the delta.enableTypeWidening table property to true:

  ALTER TABLE <table_name> SET TBLPROPERTIES ('delta.enableTypeWidening' = 'true')

  CREATE TABLE T(c1 INT) TBLPROPERTIES('delta.enableTypeWidening' = 'true')

  ALTER TABLE <table_name> ALTER COLUMN <col_name> TYPE <new_type>

# Partitioning:
* Tables with less than 1 TB of data mostly do not require partitions.
* Databricks recommends all partitions contain at least a gigabyte of data. Tables with fewer, larger partitions tend to outperform tables with many smaller partitions.
* **Use ingestion time clustering**:
     * By using Delta Lake and Databricks Runtime 11.3 LTS or above, unpartitioned tables you create benefit automatically from ingestion time clustering
     * Ingestion time provides similar query benefits to partitioning strategies based on datetime fields without any need to optimize or tune your data.
     * To maintain ingestion time clustering when you perform a large number of modifications using UPDATE or MERGE statements on a table, Databricks recommends running OPTIMIZE with ZORDER BY using a column that matches the ingestion order. For instance, this could be a column containing an event timestamp or a creation date.
     * *Not that useful in prod as there will be many updates/merge and then zorder will anyways be required*
* **Do Delta Lake and Parquet share partitioning strategies?**:
     * Delta Lake uses Parquet as the primary format for storing data, and some Delta tables with partitions specified demonstrate organization similar to Parquet tables stored with Apache Spark. 
     * Apache Spark uses Hive-style partitioning when saving data in Parquet format. Hive-style partitioning **is not** part of the Delta Lake protocol, and workloads should not rely on this partitioning strategy to interact with Delta tables.
     * While Azure Databricks and Delta Lake build upon open source technologies like Apache Spark, Parquet, Hive, and Hadoop, partitioning motivations and strategies useful in these technologies do not generally hold true for Azure Databricks.
     * If you do choose to partition your table, consider the following facts before choosing a strategy:
          * Transactions are not defined by partition boundaries. Delta Lake ensures ACID through transaction logs, so you do not need to separate a batch of data by a partition to ensure atomic discovery.
          * Azure Databricks compute clusters do not have data locality tied to physical media. Data ingested into the lakehouse is stored in cloud object storage. While data is cached to local disk storage during data processing, Azure Databricks uses file-based statistics to identify the minimal amount of data for parallel loading.

* **If partitions are so bad, why do some Azure Databricks features use them?**
* Partitions can be beneficial, especially for very large tables. Many performance enhancements around partitioning focus on very large tables (hundreds of terabytes or greater).
* Many customers migrate to Delta Lake from Parquet-based data lakes. The CONVERT TO DELTA statement allows you to convert an existing Parquet-based table to a Delta table without rewriting existing data. As such, many customers have large tables that inherit previous partitioning strategies. Some optimizations developed by Databricks seek to leverage these partitions when possible, mitigating some potential downsides for partitioning strategies not optimized for Delta Lake.
* Is it possible to outperform Azure Databricks built-in optimizations with custom partitioning?: Some experienced users of Apache Spark and Delta Lake might be able to design and implement a pattern that provides better performance than ingestion time clustering. Implementing a bad partitioning stategy can have very negative repercussions on downstream performance and might require a full rewrite of data to fix. Databricks recommends that most users use default settings to avoid introducing expensive inefficiencies.
* **How do Z-order and partitions work together?**
* You cannot combine files across partition boundaries, and so Z-order clustering can only occur within a partition. For unpartitioned tables, files can be combined across the entire table.
* Partitioning works well only for low or known cardinality fields (for example, date fields or physical locations), but not for fields with high cardinality such as timestamps. Z-order works for all fields, including high cardinality fields and fields that may grow infinitely
* You cannot Z-order on fields used for partitioning.

**Pros and cons of Hive-style partitioning**
*https://delta.io/blog/pros-cons-hive-style-partionining/#:~:text=Hive%2Dstyle%20partitioning%20physically%20separates,dataset%20that's%20partitioned%20by%20country.&text=Let's%20look%20at%20how%20Hive,our%20attention%20to%20the%20downsides.*
* Parquet files have file level statistics (such as min/max/null counts for each column chunk) in the footer
* The hive-style partition details are captured in the metatstore in spark
* On the other hand, column stats are stored in delta log in delta lake
* Suppose you want to run the following query: select count(*) from the_table where country = 'Angola'. This query will run faster if the data lake is partitioned by the country column. The query engine only needs to list and read the data files in the country='Angola' directory. It can skip the data files in the other directories. Engines need to run file listing operations to determine the files that must be read for different queries. Hive-style partitioning allows the query engine to read less files for certain queries.
* File listing operations execute differently depending on the underlying storage system. A file listing operation on a Unix-based file system like Mac OS is executed differently than on a cloud based key-value store, like Amazon S3.
* *Globbing nested directories is much slower on key-value object stores. Slower file listing operations is just one way Hive-style partitioning can make queries slower.*
* Hive-style partitioning can also make the small file problem worse. Hive-style partitioning requires data to be stored in separate files. Query engines generally run slower if the data lake contains a lot of small files. The query will generally run faster on 100 files that are 1 GB each compared to 10,000 files that are 0.01 GB each.

**Hive-style partitioning for Lakehouse storage systems**: Let’s dive into the architecture of a Lakehouse storage system to understand how they list files different and why physical disk partitioning isn’t even necessary.
* Lakehouse storage systems (like Delta Lake) store data in Parquet files and metadata about the files (including the partition structure) in the transaction log.
* Engines find the file paths in the transaction log when querying Lakehouse storage systems - they don’t need to perform file listing operations (sufficient reason to avoid data partitioning!). Avoiding file listing operations is one major advantage of a Lakehouse storage system over a data lake.
* Engines don’t need physical disk partitioning to enjoy the data skipping benefits of disk partitioning when Lakehouse storage systems are used. The engine can get all the file skipping benefits from consulting the transaction log. It doesn’t need to glob directories to discern which files contain certain partition values.
* The only reason Delta Lake supports physical partitioning is for compatibility with other engines that support Hive-style partitioning and to make conversions possible.

**Hive-style partitioning for concurrency**
* There are certain types of operations that can bypass concurrency issues via Hive-style partitioning.
* concurrent update and delete operations on the same data may conflict and will throw errors.
* You can bypass these concurrency errors on a Hive-style partitioned table by running the update and delete operations on non-overlapping partitions. e.g: UPDATE table WHERE date > '2010-01-01'  and DELETE table WHERE date < '2010-01-01'
* Hive-style partitioning allows users to bypass concurrency issues in certain circumstances, but as we’ve previously mentioned, separating data into subdirectories isn’t strictly necessary to get a full division of data. Logical partitioning (separating data into separate files and recording this in the transaction log) is sufficient and physical partitioning (separate data into separate folders) isn’t strictly necessary.
        
   


