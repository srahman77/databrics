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
# Liquid Clustering:
* Delta Lake liquid clustering replaces table partitioning and ZORDER to simplify data layout decisions and optimize query performance.
* Liquid clustering provides flexibility to redefine clustering keys without rewriting existing data, allowing data layout to evolve alongside analytic needs over time.
* Databricks recommends using Databricks Runtime 15.2 and above for all tables with liquid clustering enabled.
* Row-level concurrency is generally available in Databricks Runtime 14.2 and above for all tables with deletion vectors enabled.
* The following are examples of scenarios that benefit from Liquid Clustering:
    * Tables often filtered by high cardinality columns.
    * Tables with significant skew in data distribution.
    * Tables that grow quickly and require maintenance and tuning effort.
    * Tables with concurrent write requirements.
    * Tables with access patterns that change over time.
    * Tables where a typical partition key could leave the table with too many or too few partitions.
* You can enable liquid clustering on an existing table or during table creation.
* Clustering is not compatible with partitioning or ZORDER, and requires that you use Azure Databricks to manage all layout and optimization operations for data in your table.
* After liquid clustering is enabled, run OPTIMIZE jobs as usual to incrementally cluster data, meaning that data is only rewritten as necessary to accommodate data that needs to be clustered. Data files with clustering keys that do not match data to be clustered are not rewritten.
* For tables experiencing many updates or inserts, Databricks recommends scheduling an OPTIMIZE job every one or two hours. Because liquid clustering is incremental, most OPTIMIZE jobs for clustered tables run quickly.
* Enabling LC:  CREATE TABLE table1(col0 int, col1 string) CLUSTER BY (col0); *alter* ALTER TABLE <table_name> CLUSTER BY (<clustering_columns>)
* **Warning** :Tables created with liquid clustering enabled have numerous Delta table features enabled at creation and use Delta writer version 7 and reader version 3. Table protocol versions cannot be downgraded, and tables with clustering enabled are not readable by Delta Lake clients that do not support all enabled Delta reader protocol table features.
* Choose clustering keys: Databricks recommends choosing clustering keys based on commonly used query filters. Clustering keys can be defined in any order. If two columns are correlated, you only need to add one of them as a clustering key. You can specify up to 4 columns as clustering keys. You can only specify columns with statistics collected for clustering keys.
* If you’re converting an existing table, consider the following recommendations:
  ![image](https://github.com/user-attachments/assets/c5888911-a8b4-4484-a20f-b6cd06eda115)
* Clustering on write only gets triggered when data in the transaction meets a size threshold:
  ![image](https://github.com/user-attachments/assets/ade8d98b-6e84-47ca-8fd6-e84446e0263d)
* Because not all operations apply (trigger) liquid clustering, Databricks recommends frequently running OPTIMIZE to ensure that all data is efficiently clustered.
* You can change clustering keys for a table at any time by running an ALTER TABLE command. When you change clustering keys, subsequent OPTIMIZE and write operations use the new clustering approach, but existing data is *not rewritten*.
* You can also turn off clustering by setting the keys to NONE, as in the following example: ALTER TABLE table_name CLUSTER BY NONE;
* Setting cluster keys to NONE does not rewrite data that has already been clustered, but prevents future OPTIMIZE operations from using clustering keys.
* Limitations:
    * In Databricks Runtime 15.1 and below, clustering on write does not support source queries that include filters, joins, or aggregations.
    * Structured Streaming workloads do not support clustering-on-write.
    * You cannot create a table with liquid clustering enabled using a Structured Streaming write. You can use Structured Streaming to write data to an existing table with liquid clustering enabled.
