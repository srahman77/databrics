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

