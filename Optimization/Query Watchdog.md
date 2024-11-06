* A challenge with interactive data workflows is handling large queries. This includes queries that generate too many output rows, fetch many external partitions, or compute on extremely large data sets. These queries can be extremely slow, saturate compute resources, and make it difficult for others to share the same compute.
* Query Watchdog is a process that prevents queries from monopolizing compute resources by examining the most common causes of large queries and terminating queries that pass a threshold
* Query Watc:hdog is enabled for all all-purpose computes created using the UI.
* Example of a disruptive query:
  Suppose there are two tables that each have a million rows.
  ![image](https://github.com/user-attachments/assets/2fb6f865-fbd1-4b48-99d2-a2cecc64bab7)
    * These table sizes are manageable in Apache Spark. However, they each include a join_key column with an empty string in every row.
    * In the following code, the analyst is joining these two tables on their keys, which produces output of one trillion results, *and all of these are produced on a single executor (the executor that gets the " " key)*:
      ![image](https://github.com/user-attachments/assets/138b9a0b-5329-46fe-9eb5-00440090f3d6)
    * This query appears to be running. But without knowing about the data, the analyst sees that there’s “only” a single task left over the course of executing the job. The query never finishes.
    * We can enable and configure Query Watchdog to handle such queries gracefully (Not only does Query Watchdog prevent users from monopolizing compute resources for jobs that will never complete, it also saves time by fast-failing a query that would have never completed).
* Enable and configure Query Watchdog:
    * Enable Watchdog with spark.databricks.queryWatchdog.enabled
    * Configure the task runtime with spark.databricks.queryWatchdog.minTimeSecs : minimum time a given task in a query must run before cancelling
    * Display output with spark.databricks.queryWatchdog.minOutputRows: minimum number of output rows for a task in that query.
    * Configure the output ratio with spark.databricks.queryWatchdog.outputRatioThreshold (op/ip ratio): To a prevent a query from creating too many output rows for the number of input rows
    * If you configure Query Watchdog in a notebook, the configuration does not persist across compute restarts. If you want to configure Query Watchdog for all users of a compute, we recommend that you use a compute configuration.
* Detect query on extremely large dataset:
    * Another typical large query may scan a large amount of data from big tables/datasets. The scan operation may last for a long time and saturate compute resources (even reading metadata of a big Hive table can take a significant amount of time).
    * You can set maxHivePartitions to prevent fetching too many partitions from a big Hive table. Similarly, you can also set maxQueryTasks to limit queries on an extremely large dataset.

      spark.conf.set("spark.databricks.queryWatchdog.maxHivePartitions", 20000)

      spark.conf.set("spark.databricks.queryWatchdog.maxQueryTasks", 20000)  
    * When should you enable Query Watchdog? Query Watchdog should be enabled for ad hoc analytics compute where SQL analysts and data scientists are sharing a given compute and an administrator needs to make sure that queries “play nicely” with one another.
    * When should you disable Query Watchdog? In general we do not advise eagerly cancelling queries used in an ETL scenario because there typically isn’t a human in the loop to correct the error. We recommend that you disable Query Watchdog for all but ad hoc analytics compute.

