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
