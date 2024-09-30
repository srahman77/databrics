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
  ![image](https://github.com/user-attachments/assets/f14b46f5-2870-46b0-8d84-0635688b0ce0)
**Star Schema Join withiout DFP**
* e.g SELECT sum(ss_quantity) 
    FROM store_sales 
    JOIN item ON ss_item_sk = i_item_sk
    WHERE i_item_id = 'AAAAAAAAICAAAAAA'
* store_sales is *probe* sode and item is build *side*
* Keep the dim tables at the right side of a join for better performance/best practice! See the details down below (left relation of a left outer join cannot be broadcasted)
* It specifies the predicate on the dimension table (item), not the fact table (store_sales). This means that filtering of rows for store_sales would typically be done as part of the JOIN operation since the values of ss_item_sk are not known until after the SCAN and FILTER operations take place on the item table.
  ![image](https://github.com/user-attachments/assets/cda9a450-b4c3-4a2e-b1da-4f7d875a1514)
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
* No of files will increase though! So we will run optimize to handle small file problem.

# AQE:
* Adaptive query execution (AQE) is query re-optimization that occurs during query execution.
* The motivation for runtime re-optimization is that Azure Databricks has the most up-to-date accurate statistics at the end of a shuffle and broadcast exchange (referred to as a query stage in AQE). As a result, Azure Databricks can opt for a better physical strategy, pick an optimal post-shuffle partition size and number, or do optimizations that used to require hints, for example, skew join handling.
* AQE is enabled by default. It has 4 major features:
    * Dynamically changes sort merge join into broadcast hash join.
    * Dynamically coalesces partitions (combine small partitions into reasonably sized partitions) after shuffle exchange. Very small tasks have worse I/O throughput and tend to suffer more from scheduling overhead and task setup overhead. Combining small tasks saves resources and improves cluster throughput.
    * Dynamically handles skew in sort merge join and shuffle hash join by splitting (and replicating if needed) skewed tasks into roughly evenly sized tasks.
    * Dynamically detects and propagates empty relations.
* AdaptiveSparkPlan node:
  * AQE-applied queries contain one or more AdaptiveSparkPlan nodes, usually as the root node of each main query or sub-query. Before the query runs or when it is running, the isFinalPlan flag of the corresponding AdaptiveSparkPlan node shows as false; after the query execution completes, the isFinalPlan flag changes to true.
  * The query plan diagram evolves as the execution progresses and reflects the most current plan that is being executed. Nodes that have already been executed (in which metrics are available) will not change, but those that haven’t can change over time as the result of re-optimizations.
    ![image](https://github.com/user-attachments/assets/705b74c9-657e-48ba-8ef3-91abd4052b18)
* Runtime statistics:
  * Each shuffle and broadcast stage contains data statistics.
  * Before the stage runs or *when the stage is running*, the statistics are compile-time estimates, and the flag *isRuntime* is false, for example: Statistics(sizeInBytes=1024.0 KiB, rowCount=4, isRuntime=false);
  * After the stage execution completes, the statistics are those collected at runtime, and the flag isRuntime will become true, for example: Statistics(sizeInBytes=658.1 KiB, rowCount=2.81E+4, isRuntime=true)
  * The following is a DataFrame.explain example:
    ![image](https://github.com/user-attachments/assets/d887f63e-5a5d-4359-9f7e-a75403cb059d)
    ![image](https://github.com/user-attachments/assets/3ff95f47-94a5-43b6-ab82-9cc79ab58d2b)
  * As **SQL EXPLAIN** does not execute the query, the current plan is always the same as the initial plan and does not reflect what would eventually get executed by AQE.
    ![image](https://github.com/user-attachments/assets/c4a74e47-5020-4c47-bed5-f22f97ce4743)
* AQE Configuration:
**Dynamically coalesce partitions**
  * spark.databricks.optimizer.adaptive.enabled (Default value: true): Whether to enable or disable adaptive query execution.
  * spark.sql.shuffle.partitions (Default value: 200): The default number of partitions to use when shuffling data for joins or aggregations. Setting the value auto enables auto-optimized shuffle, which automatically determines this number based on the query plan and the query input data size.
  * spark.databricks.adaptive.autoBroadcastJoinThreshold (Default value: 30MB): The threshold to trigger sort merge join into broadcast hash join
  * spark.sql.adaptive.coalescePartitions.enabled (Default value: true):Whether to enable or disable partition coalescing.
  * spark.sql.adaptive.advisoryPartitionSizeInBytes (Default value: 64MB): The target size after coalescing. The coalesced partition sizes will be close to but no bigger than this target size.
  * spark.sql.adaptive.coalescePartitions.minPartitionSize (Default value: 1MB): The minimum size of partitions after coalescing. The coalesced partition sizes will be no smaller than this size.
  * spark.sql.adaptive.coalescePartitions.minPartitionNum (Default value: 2x no. of cluster cores): The minimum number of partitions after coalescing. *Not recommended, because setting explicitly overrides spark.sql.adaptive.coalescePartitions.minPartitionSize*.
**Dynamically handle skew join**
  * spark.sql.adaptive.skewJoin.enabled(Default value: true): Whether to enable or disable skew join handling.
  * spark.sql.adaptive.skewJoin.skewedPartitionFactor (Default value: 5): A factor that when multiplied by the median partition size contributes to determining whether a partition is skewed.
  * spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes (Default value: 256MB): A threshold that contributes to determining whether a partition is skewed.
  * A partition is considered skewed when both (partition size > skewedPartitionFactor * median partition size) and (partition size > skewedPartitionThresholdInBytes) are true.
**Dynamically detect and propagate empty relations**
  * spark.databricks.adaptive.emptyRelationPropagation.enabled (Default value: true): Whether to enable or disable dynamic empty relation propagation.

**FAQs on AQE**
* Why didn’t AQE broadcast a small join table?
  
  If the size of the relation expected to be broadcast does fall under this threshold but is still not broadcast:
  * Check the join type. Broadcast is not supported for certain join types, for example, the *left relation of a LEFT OUTER JOIN or right relation of a RIGHT OUTER JOIN cannot be broadcast*. Some more stack overflow poinyts on this:
      * Big-Table left outer join Small-Table -- Broadcast Enabled; Small-Table left outer join Big-Table -- Broadcast Disabled: The reason for this is that Spark shares the small table (also known as the broadcast table) to all data nodes where the big table data is present. In your case, you need all the data from the small table but only the matching data from the big table. Spark cannot determine whether a particular record was matched at another data node or if there was no match at all, so there is ambiguity when selecting all the records from the small table if it were distributed. As a result, Spark won't use broadcast join in this scenario.
      * Why ambiguous result?: In a left outer join, at least one record from the left table should be included in the output. If there's no match at worker 1 for a record, sending it back to the driver, and subsequently finding a match on worker 2, can lead to ambiguous results. The output may include the record from worker 1 (without a match with the right table) and the matched record from worker 2. As you mentioned, if we only send record back to driver incase of a match, then it will be Equi-join not left outer join.
  * It can also be that the relation contains a lot of empty partitions, in which case the majority of the tasks can finish quickly with sort merge join or it can potentially be optimized with skew join handling. AQE avoids changing such sort merge joins to broadcast hash joins if the percentage of non-empty partitions is lower than spark.sql.adaptive.nonEmptyPartitionRatioForBroadcastJoin.

* Should I still use a broadcast join strategy hint with AQE enabled?
  * Yes. A statically planned broadcast join is usually more performant than a dynamically planned one by AQE as AQE might not switch to broadcast join until after performing shuffle for both sides of the join (by which time the actual relation sizes are obtained). So using a broadcast hint can still be a good choice if you know your query well. AQE will respect query hints the same way as static optimization does, but can still apply dynamic optimizations that are not affected by the hints.

* What is the difference between skew join hint and AQE skew join optimization? Which one should I use?
  * It is recommended to rely on AQE skew join handling rather than use the skew join hint, because AQE skew join is completely automatic and in general performs better than the hint counterpart.
 
* Why didn’t AQE adjust my join ordering automatically?
  * Dynamic join reordering is not part of AQE.

* Why didn’t AQE detect my data skew?

  There are two size conditions that must be satisfied for AQE to detect a partition as a skewed partition:
  * The partition size is larger than the spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes (default 256MB)
  * The partition size is larger than the median size of all partitions times the skewed partition factor spark.sql.adaptive.skewJoin.skewedPartitionFactor (default 5)
  * In addition, skew handling support is limited for certain join types, for example, in LEFT OUTER JOIN, only skew on the left side can be optimized.


# Predictive I/O 
* Predictive I/O is exclusive to the Photon engine on Azure Databricks.
* Predictive I/O is a collection of Azure Databricks optimizations that improve performance for data interactions. Predictive I/O capabilities are grouped into the following categories:
  * Accelerated reads reduce the time it takes to scan and read data.
  * Accelerated updates reduce the amount of data that needs to be rewritten during updates, deletes, and merges.
* Predictive I/O reads are supported by the serverless and pro types of SQL warehouses, and Photon-accelerated clusters running Databricks Runtime 11.3 LTS and above.
* Predictive I/O improves scanning performance by applying deep learning techniques to do the following:
  * Determine the most efficient access pattern to read the data and only scanning the data that is actually needed.
  * Eliminate the decoding of columns and rows that are not required to generate query results.
  * Calculate the probabilities of the search criteria in selective queries matching a row. As queries run, we use these probabilities to anticipate where the next matching row would occur and only read that data from cloud storage.
* Predictive I/O leverages deletion vectors to accelerate updates by reducing the frequency of full file rewrites during data modification on Delta tables. Predictive I/O optimizes DELETE, MERGE, and UPDATE operations.
* Use predictive I/O to accelerate updates: Predictive I/O for updates are used automatically for all tables that have deletion vectors enabled using the following Photon-enabled compute types:
  * Serverless SQL warehouses.
  * Pro SQL warehouses.
  * Clusters running Databricks Runtime 14.0 and above.

# Cost-based optimizer
* Spark SQL can use a cost-based optimizer (CBO) to improve query plans. This is especially useful for queries with multiple joins. For this to work it is critical to collect table and column statistics and keep them up to date.
* To keep the statistics up-to-date, run *ANALYZE TABLE* after writing to the table.
* The rowCount statistic is especially important for queries with multiple joins. If rowCount is missing, it means there is not enough information to calculate it (that is, some required columns do not have statistics).
* For example, CBO might switch from a hash join to a sort-merge join based on data size and distribution, optimizing performance
* Few points in UI for example:
![image](https://github.com/user-attachments/assets/e65ff27d-fa0f-47e4-80da-e4d530cc3c4d)
  * A line such as *rows output: 2,451,005 est: N/A* means that this operator produces approximately 2M rows and there were no statistics available.
  * A line such as *rows output: 2,451,005 est: 1616404 (1X)* means that this operator produces approx. 2M rows, while the estimate was approx. 1.6M and the estimation error factor was 1.
  * A line such as *rows output: 2,451,005 est: 2626656323 (1000X)* means that this operator produces approximately 2M rows while the estimate was 2B rows, so the estimation error factor was 1000.
* The CBO is enabled by default. You disable the CBO by: spark.conf.set("spark.sql.cbo.enabled", false)
 

# Range join optimization
* A range join occurs when two relations are joined using a point in interval or interval overlap condition. The range join optimization support in Databricks Runtime can bring orders of magnitude improvement in query performance, but requires careful manual tuning.
* Point in interval range join: A point in interval range join is a join in which the condition contains predicates specifying that a value from one relation is between two values from the other relation. For example:

  SELECT * FROM points JOIN ranges ON points.p BETWEEN ranges.start and ranges.end;

  SELECT * FROM points JOIN ranges ON points.p >= ranges.start AND points.p < ranges.end;

  SELECT * FROM points JOIN ranges ON points.p >= ranges.start AND points.p < ranges.start + 100;

  SELECT * FROM points1 p1 JOIN points2 p2 ON p1.p >= p2.p - 10 AND p1.p <= p2.p + 10;

* Interval overlap range join: An interval overlap range join is a join in which the condition contains predicates specifying an overlap of intervals between two values from each relation. For example:
  
   SELECT * FROM r1 JOIN r2 ON r1.start < r2.end AND r2.start < r1.end;

   SELECT * FROM r1 JOIN r2 ON r1.start < r2.start + 100 AND r2.start < r1.start + 100;

* Range join optimization: The range join optimization is performed for joins that:
  * Have a condition that can be interpreted as a point in interval or interval overlap range join.
  * All values involved in the range join condition are of a numeric type (integral, floating point, decimal), DATE, or TIMESTAMP.
  * All values involved in the range join condition are of the same type. In the case of the decimal type, the values also need to be of the same scale and precision.
  * It is an INNER JOIN, or in case of point in interval range join, a LEFT OUTER JOIN with point value on the left side, or RIGHT OUTER JOIN with point value on the right side.
  * Have a bin size tuning parameter

 * Bin size:
   * The bin size is a numeric tuning parameter that splits the values domain of the range condition into multiple bins of equal size. For example, with a bin size of 10, the optimization splits the domain into bins that are intervals of length 10. If you have a point in range condition of p BETWEEN start AND end, and start is 8 and end is 22, this value interval overlaps with three bins of length 10 – the first bin from 0 to 10, the second bin from 10 to 20, and the third bin from 20 to 30. Only the points that fall within the same three bins need to be considered as possible join matches for that interval. For example, if p is 32, it can be ruled out as falling between start of 8 and end of 22, because it falls in the bin from 30 to 40.
   * For DATE values, the value of the bin size is interpreted as days. For example, a bin size value of 7 represents a week.
   * For TIMESTAMP values, the value of the bin size is interpreted as seconds. If a sub-second value is required, fractional values can be used. For example, a bin size value of 60 represents a minute, and a bin size value of 0.1 represents 100 milliseconds.
   * You can specify the bin size either by using a range join hint in the query or by setting a session configuration parameter. The range join optimization is applied only if you manually specify the bin size.
   * hint e.g : SELECT /*+ RANGE_JOIN(points, 10) */ * FROM points JOIN ranges ON points.p >= ranges.start AND points.p < ranges.end;
   * See more whenever needed

# Isolation levels and write conflicts on Azure Databricks
* 
   
  
  
 

  

  
