# Optimize performance with caching on Azure Databricks:
* Azure Databricks uses disk caching (previously known as delta cache/DBIO cache) to accelerate data reads by creating copies of remote Parquet data files in nodes’ local storage using a fast intermediate data format. The data is cached automatically whenever a file has to be fetched from a remote location. Successive reads of the same data are then performed locally, which results in significantly improved reading speed.
* Disk caching behavior is a proprietary Azure Databricks feature. This name change from delta cache/DBIO cache seeks to resolve confusion that it was part of the Delta Lake protocol.
* In SQL warehouses and Databricks Runtime 14.2 and above, the CACHE SELECT command is ignored. An enhanced disk caching algorithm is used instead.
* Disk cache vs. Spark cache: Azure Databricks recommends using automatic disk caching.
  ![image](https://github.com/user-attachments/assets/f8460bdd-8781-4c83-805f-230833ac064b)
* Disk cache consistency: The disk cache automatically detects when data files are created, deleted, modified, or overwritten and updates its content accordingly. You can write, modify, and delete table data with no need to explicitly invalidate cached data. Any stale entries are automatically invalidated and evicted from the cache.
* Selecting instance types to use disk caching:
    * The recommended (and easiest) way to use disk caching is to choose a worker type with SSD volumes when you configure your cluster. Such workers are enabled and configured for disk caching
    * The disk cache is configured to use at most half of the space available on the local SSDs provided with the worker nodes

* Configure the disk cache:
    * Azure Databricks recommends that you choose cache-accelerated worker instance types for your compute. Such instances are automatically configured optimally for the disk cache
    * When a worker is decommissioned, the Spark cache stored on that worker is lost. So if autoscaling is enabled, there is some instability with the cache. Spark would then need to reread missing partitions from source as needed.
    * To configure how the disk cache uses the worker nodes’ local storage, specify the following Spark configuration settings **during cluster creation**:
        * spark.databricks.io.cache.maxDiskUsage: disk space per node reserved for cached data in bytes (spark.databricks.io.cache.maxDiskUsage 50g)
        * spark.databricks.io.cache.maxMetaDataCache: disk space per node reserved for cached metadata in bytes (spark.databricks.io.cache.maxMetaDataCache 1g)
        * spark.databricks.io.cache.compression.enabled: should the cached data be stored in compressed format (spark.databricks.io.cache.compression.enabled false)
    * Enable or disable the disk cache: spark.conf.set("spark.databricks.io.cache.enabled", "[true | false]").  Disabling the cache does not result in dropping the data that is already in the local storage. Instead, it prevents queries from adding new data to the cache and reading data from the cache.

# Photon accelerated updates:
* Support for Photon accelerated updates is in Public Preview in Databricks Runtime 12.2 LTS and above.
* Photon leverages deletion vectors to accelerate updates by reducing the frequency of full file rewrites during data modification on Delta tables. Photon optimizes DELETE, MERGE, and UPDATE operations.
* You enable support for deletion vectors on a Delta Lake table by setting a Delta Lake table property:

  ALTER TABLE <table-name> SET TBLPROPERTIES ('delta.enableDeletionVectors' = true);

# Archival support in Azure Databricks:
* This feature is in Public Preview for Databricks Runtime 13.3 LTS and above.
* Archival support in Azure Databricks introduces a collection of capabilities that enable you to use cloud-based lifecycle policies on cloud object storage containing Delta tables.
* Archival support only allows queries that can be answered correctly without touching archived files. These queries include those that either: Query metadata only OR Have filters that do not require scanning any archived files.
* All queries that require data in archived files fail.
* Without archival support, operations against Delta tables might break because data files or transaction log files have moved to archived locations and are unavailable when queried. Archival support introduces optimizations to avoid querying archived data when possible. It also adds new syntax to identify files that must be restored from archival storage to complete queries.
* Enabling archival support for a table in Azure Databricks does not create or alter lifecycle policies defined for your cloud object storage. For desired results, your cloud lifecycle policy and the delta.timeUntilArchived setting should be equal.
![image](https://github.com/user-attachments/assets/c7767bed-1431-47f8-8a12-ea59e7a86085)
* **Early failure and error messages**:
* For queries that must scan archived files to generate correct results, configuring archival support for Delta Lake ensures the following:
    * Queries fail early if they attempt to access archived files, reducing wasted compute and allowing users to adapt and re-run queries quickly.
    * Error messages inform users that a query has failed because the query attempted to access archived files.
    * Users can generate a report of files that must be restored using the SHOW ARCHIVED FILES syntax.
* Show archived files: To identify files that must be restored to complete a given query, use *SHOW ARCHIVED FILES FOR table_name [ WHERE predicate ];*
* This operation returns URIs for archived files as a Spark DataFrame. Restore the necessary archived files following documented instructions from your object storage provider.
* Databricks recommends providing predicates that include fields on which data is partitioned, z-ordered, or clustered to reduce the number of files that must be restored.
* Limitations:
   * No support exists for lifecycle management policies that are not based on file creation time. This includes access-time-based policies and tag-based policies.
   * You cannot use DROP COLUMN on a table with archived files.
   * REORG TABLE APPLY PURGE makes a best-effort attempt, but only works on deletion vector files and referenced data files that are not archived. PURGE cannot delete archived deletion vector files.
   * Extending the lifecycle management transition rule results in unexpected behavior
   * If you change the time interval for your cloud lifecycle management transition rule, you must update the property delta.timeUntilArchived.

# Dynamic file pruning
* Dynamic File Pruning (DFP), a new data-skipping technique, which can significantly improve queries with selective joins on non-partition columns on tables in Delta Lake, now enabled by default in Databricks Runtime.
* Prior to Dynamic File Pruning, file pruning only took place when queries contained a literal value in the predicate (and use that in delta log stats). DFP  works for both literal filters as well as join filters.
* This means that Dynamic File Pruning now allows star schema queries to take advantage of data skipping at file granularity (join filters on the fact table are unknown at query compilation time).
* ![image](https://github.com/user-attachments/assets/f14b46f5-2870-46b0-8d84-0635688b0ce0)
**Star Schema Join withiout DFP**
* e.g SELECT sum(ss_quantity) 
    FROM store_sales 
    JOIN item ON ss_item_sk = i_item_sk
    WHERE i_item_id = 'AAAAAAAAICAAAAAA'
* store_sales is *probe* sode and item is build *side*
* It specifies the predicate on the dimension table (item), not the fact table (store_sales). This means that filtering of rows for store_sales would typically be done as part of the JOIN operation since the values of ss_item_sk are not known until after the SCAN and FILTER operations take place on the item table.
* ![image](https://github.com/user-attachments/assets/cda9a450-b4c3-4a2e-b1da-4f7d875a1514)
* only 48K rows meet the JOIN criteria yet over 8.6B records had to be read from the store_sales table
**Star Schema Join with Dynamic File Pruning** 
![image](https://github.com/user-attachments/assets/e54593d6-a7a7-4533-a215-c11bb62304ed)
* The number of scanned rows has been reduced from 8.6 billion to 66 million rows. Whereas the improvement is significant, we still read more data than needed because DFP operates at the granularity of files instead of rows
* DFP is automatically enabled in Databricks Runtime 6.1 and higher, and applies if a query meets the following criteria:
  * The inner table (probe side) being joined is in Delta Lake format
  * The join type is INNER or LEFT-SEMI
  * The join strategy is **BROADCAST HASH JOIN** (the joining data set has to be broadcasted to the probe tables)
  * You must use Photon-enabled compute to use dynamic file pruning in MERGE, UPDATE, and DELETE statements. Only SELECT statements leverage dynamic file pruning when Photon is not used.
* Dynamic file pruning is especially efficient for non-partitioned tables, or for joins on non-partitioned columns.
* The performance impact of dynamic file pruning is often correlated to the clustering of data so consider using Z-Ordering to maximize the benefit.
**Configuration**
* park.databricks.optimizer.dynamicFilePruning (default is true): The main flag that directs the optimizer to push down filters. When set to false, dynamic file pruning will not be in effect.
* spark.databricks.optimizer.deltaTableSizeThreshold (default is 10,000,000,000 bytes (10 GB)): Represents the minimum size (in bytes) of the Delta table on the probe side of the join (the right table is called the build side, and the left table is called the probe side.) required to trigger dynamic file pruning. If the probe side is not very large, it is probably not worthwhile to push down the filters and we can just simply scan the whole table.
* spark.databricks.optimizer.deltaTableFilesThreshold (default is 10): Represents the number of files of the Delta table on the probe side of the join required to trigger dynamic file pruning.

# Low shuffle merge on Azure Databricks"
* Databricks low shuffle merge provides better performance by processing unmodified rows in a separate, more streamlined processing mode, instead of processing them together with the modified rows.
* Many MERGE workloads only update a relatively small number of rows in a table. However, Delta tables can only be updated on a per-file basis.
* When the MERGE command needs to update or delete a small number of rows that are stored in a particular file, then it must also process and rewrite all remaining rows that are stored in the same file, even though these rows are unmodified.
* Prior to low shiffle merge, they were processed in the same way as modified rows, passing them through multiple shuffle stages and expensive calculations. In low shuffle merge, the unmodified rows are instead processed without any shuffles, expensive processing, or other added overhead.
* The earlier MERGE implementation caused the data layout of unmodified data to be changed entirely (due to the shuffle), resulting in lower performance on subsequent operations.
* Low shuffle merge tries to preserve the existing data layout of the unmodified records, including Z-order optimization on a best-effort basis. Hence, with low shuffle merge, the performance of operations on a Delta table will degrade more slowly after running one or more MERGE commands
* Low shuffle merge is enabled by default in Databricks Runtime 10.4 and above. 
**e.g:**
* The Merge operation joins the source and destination (Step 1 shown below) which, in Apache Spark™, shuffles the rows in the table, breaking the existing ordering of the table. MERGE then executes the changeset against the joined table, writing the resulting set of rows into a new file
![image](https://github.com/user-attachments/assets/c0d2d5a5-827a-4904-8059-3c91e7ada8d1)
* Low-Shuffle MERGE optimizes execution by removing any updated and deleted rows (Step 2 shown below) from the original file while writing any new and updated rows (Step 1 shown below) in a separate file
 ![image](https://github.com/user-attachments/assets/86aa7602-a0f5-41fc-baea-dcb7a2ae4f65)
* No of files will increase though! So we will run optimize to handle small file problem

  
