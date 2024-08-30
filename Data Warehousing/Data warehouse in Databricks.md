* Data warehousing refers to collecting and storing data from multiple sources so it can be quickly accessed for business insights and reporting.
* Databricks SQL is the collection of services that bring data warehousing capabilities and performance to your existing data lakes. Databricks SQL supports open formats and standard ANSI SQL
* The data warehouse is modeled at the silver layer and feeds specialized data marts in the gold layer.
* Refresh operrations for materialised views:
    * Incremental refresh: An incremental refresh processes changes in the underlying data after the last refresh and then appends that data to the table. Depending on the base tables and included operations, only certain types of materialized views can be incrementally refreshed. When materialized views are created using a SQL warehouse or serverless Delta Live Tables pipeline, they are automatically incrementally refreshed if their queries are supported. If a query includes unsupported expressions for an incremental refresh, a full refresh will be performed, potentially resulting in additional costs.
    * Full refresh: A full refresh truncates the table and reprocesses all data available in the source with the latest definition. It is not recommended to perform full refreshes on sources that don’t keep the entire data history or have short retention periods, such as Kafka, because the full refresh truncates the existing data. You may be unable to recover old data if the data is no longer available in the source.
# Serverless SQL warehouses: 
* Databricks SQL delivers optimal price and performance with serverless SQL warehouses. Key advantages of serverless warehouses over pro and classic models include:
    * Instant and elastic compute: Eliminates waiting for infrastructure resources and avoids resource over-provisioning during usage spikes. Intelligent workload management dynamically handles scaling.
    * Minimal management overhead: Capacity management, patching, upgrades, and performance optimization are all handled by Azure Databricks, simplifying operations and leading to predictable pricing.
    * Lower total cost of ownership (TCO): Automatic provisioning and scaling of resources as needed helps avoid over-provisioning and reduces idle times, thus lowering TCO. 
* To decrease query latency for a given serverless SQL warehouse:
   * If queries are spilling to disk, increase the t-shirt size.
   * If the queries are highly parallelizable, increase the t-shirt size.
   * If you are running multiple queries at a time, add more clusters for autoscaling.
* To reduce costs, try to step down in t-shirt size without spilling to disk or significantly increasing latency.
* To help right-size your serverless SQL warehouse, use the following tools:
   * Monitoring page: look at the peak query count. If the peak queued is commonly above one, add clusters. The maximum number of queries in a queue for all SQL warehouse types is 1000. See Monitor a SQL warehouse.
   * Query history. See Query history.
   * Query profiles (look for Bytes spilled to disk above 1)
* Serverless autoscaling and query queuing: Intelligent Workload Management (IWM) is a set of features that enhances the ability of serverless SQL warehouses to process large numbers of queries quickly and cost-effectively. The key difference lies in the AI capabilities in Databricks SQL to respond dynamically to workload demands rather than using static thresholds.When a query arrives to the warehouse, IWM predicts the cost of the query. At the same time, IWM is real-time monitoring the available compute capacity of the warehouse. Next, using machine learning models, IWM predicts if the incoming query has the necessary compute available on the existing compute. If it doesn’t have the compute needed, then the query is added to the queue. If it does have the compute needed, the query begins executing immediately. IWM monitors the queue is monitored approximately every 10 seconds. If the queue is not decreasing quickly enough, autoscaling kicks in to rapidly procure more compute. Once new capacity is added, queued queries are admitted to the new clusters.

# Classical and Pro DWH:
* Azure Databricks limits the number of queries on a cluster assigned to a SQL warehouse based on the cost to compute their results. Upscaling of clusters per warehouse is based on query throughput, the rate of incoming queries, and the queue size. Azure Databricks recommends a cluster for every 10 concurrent queries.
* Azure Databricks adds clusters based on the time it would take to process all currently running queries, all queued queries, and the incoming queries expected in the next two minutes.
     * If less than 2 minutes, don’t upscale.
     * If 2 to 6 minutes, add 1 cluster.
     * If 6 to 12 minutes, add 2 clusters.
     * If 12 to 22 minutes, add 3 clusters.
     * Otherwise, Azure Databricks adds 3 clusters plus 1 cluster for every additional 15 minutes of expected query load.
     * In addition, a warehouse is always upscaled if a query waits for 5 minutes in the queue.
     * If the load is low for 15 minutes, Azure Databricks downscales the SQL warehouse. It keeps enough clusters to handle the peak load over the last 15 minutes.
* Azure Databricks queues queries when all clusters assigned to the warehouse are executing queries at full capacity or when the warehouse is in the STARTING state. The maximum number of queries in a queue for all SQL warehouse types is 1000.
* Azure Databricks queues queries when all clusters assigned to the warehouse are executing queries at full capacity or when the warehouse is in the STARTING state. The maximum number of queries in a queue for all SQL warehouse types is 1000.
