# Debugging with the Apache Spark UI
*  To peek at the internals of your Apache Spark application, the three important places to look are:Spark UI, Driver logs and Executor logs
* **Spark UI**:
  * Once you start the job, the Spark UI shows information about what’s happening in your application
  * Streaming tab: you will see a Streaming tab if a streaming job is running in this compute. If there is no streaming job running in this compute, this tab will not be visible.
    * Processing time: As you scroll down, find the graph for Processing Time. This is one of the key graphs to understand the performance of your streaming job. As a general rule of thumb, it is good if you can process each batch within 80% of your batch processing time.
    * If the average processing time is closer or greater than your batch interval, then you will have a streaming application that will start queuing up resulting in backlog soon which can bring down your streaming job eventually.
    * Batch details page: This page has all the details you want to know about a batch. Two key things are:
        * Input: Has details about the input to the batch. In this case, it has details about the Apache Kafka topic, partition and offsets read by Spark Structured Streaming for this batch. In case of TextFileStream, you see a list of file names that was read for this batch. This is the best way to start debugging a Streaming application reading from text files.
        * Processing: You can click the link to the Job ID which has all the details about the processing done during this batch.
  * Job details page: The job details page shows a DAG visualization. The grayed boxes represents skipped stages. Spark is smart enough to skip some stages if they don’t need to be recomputed. If the data is checkpointed or cached, then Spark would skip recomputing those stages
  * Task details page: This is the most granular level of debugging you can get into from the Spark UI for a Spark application. If you are investigating performance issues of your streaming application, then this page would provide information such as the number of tasks that were executed and where they were executed (on which executors) and shuffle information. Ensure that the tasks are executed on multiple executors (nodes) in your compute to have enough parallelism while processing. If you have a single receiver, sometimes only one executor might be doing all the work though you have more than one executor in your compute.
  * **Thread dump**:
      * A thread dump shows a snapshot of a JVM’s thread states.
      * Thread dumps are useful in debugging a specific hanging or slow-running task. To view a specific task’s thread dump in the Spark UI. **Check some videos on this**

* **Driver logs**: Driver logs are helpful for 2 purposes:
  * Exceptions: Sometimes, you may not see the Streaming tab in the Spark UI. This is because the Streaming job was not started because of some exception. You can drill into the Driver logs to look at the stack trace of the exception. In some cases, the streaming job may have started properly. But you will see all the batches never going to the Completed batches section. They might all be in processing or failed state. In such cases too, driver logs could be handy to understand on the nature of the underlying issues.
  * Prints: Any print statements as part of the DAG shows up in the logs too.

* **Executor logs**:
  * Executor logs are sometimes helpful if you see certain tasks are misbehaving and would like to see the logs for specific tasks. From the task details page shown above, you can get the executor where the task was run. Once you have that, you can go to the compute UI page, click the # nodes, and then the master. The master page lists all the workers. You can choose the worker where the suspicious task was run and then get to the log4j output.


# Diagnosing issues with Spark UI

* **Job event Timeline** : https://learn.microsoft.com/en-us/azure/databricks/optimizations/spark-ui-guide/jobs-timeline
   * Failing jobs or failing executors/executores removed:
     ![image](https://github.com/user-attachments/assets/295eda2b-78d3-4971-b6d9-28b022cbd166)
   * The most common reasons for executors being removed are:
      * Autoscaling: In this case it’s expected and not an error
      * Spot instance losses: The cloud provider is reclaiming your VMs.
      * Executors running out of memory
   * Failing jobs:
      * If you see any failing jobs, click on the link in description to get to their pages. Then scroll down to see the failed stage and a failure reason:
        ![image](https://github.com/user-attachments/assets/6e4aae90-cf73-4cba-a89a-09084a7b3312)
      * You may get a generic error. Click on the link in the description again (this time in the page where failure reason is available) to see if you can get more info:
        ![image](https://github.com/user-attachments/assets/a6d6e29c-633b-4a25-9e4b-20e0e86ac3e3)
    * Failing executors:
       * To find out why your executors are failing, you’ll first want to check the compute’s Event log to see if there’s any explanation for why the executors failed. For example, it’s possible you’re using spot instances and the cloud provider is taking them back.
         ![image](https://github.com/user-attachments/assets/e15a2f10-258c-4308-bfb4-5b3b82ebec11)
       * If you don’t see any information in the event log, navigate back to the Spark UI then click the Executors tab. Here you can get the logs from the failed executors:
         ![image](https://github.com/user-attachments/assets/0237dc47-7a1e-472d-ae9d-e29efd6968c3)
  

* **Gaps between Spark jobs**
  ![image](https://github.com/user-attachments/assets/343ead6a-b85f-48cd-970f-b1e45e757c9a)
   * There are a few reasons for the gaps between execution e.g in the above pic. If the gaps make up a high proportion of the time spent on your workload, you need to figure out what is causing these gaps and if it’s expected or not. There are a few things that could be happening during the gaps:
      * There’s no work to do: On all-purpose compute, having no work to do is the most likely explanation for the gaps. Because the cluster is running and users are submitting queries, gaps are expected. These gaps are the time between query submissions.
      * Driver is compiling a complex execution plan: For example, if you use withColumn() in a loop, it creates a very expensive plan to process. The gaps could be the time the driver is spending simply building and processing the plan (even though execution time will be same, creating the plan becomes lengthier). If this is the case, try simplifying the code. Use selectExpr() to combine multiple withColumn() calls into one expression, or convert the code into SQL. You can still embed the SQL in your Python code, using Python to manipulate the query with string functions. This often fixes this type of problem. 
      * Execution of non-spark code: Spark code is either written in SQL or using a Spark API like PySpark. Any execution of code that is not Spark will show up in the timeline as gaps. For example, you could have a loop in Python which calls native Python functions. This code is not executing in Spark. If you see gaps in your timeline caused by running non-Spark code, this means your workers are all idle and likely wasting money during the gaps. Maybe this is intentional and unavoidable, but if you can write this code to use Spark you will fully utilize the cluster
      * Driver is overloaded: To determine if your driver is overloaded, you need to look at the cluster metrics (on DBR 13.0 or later, click Metrics). Notice the Server load distribution visualization. You should look to see if the driver is heavily loaded. Check more on driver overload in the following section.
          * Complete idle cluster:
            ![image](https://github.com/user-attachments/assets/ef5fea43-c14a-474f-902d-5b27c974118b)
          * Driver overloaded cluster: We can see that one square is red, while the others are blue. Roll your mouse over the red square to make sure the red block represents your driver.
            ![image](https://github.com/user-attachments/assets/923586ad-4b5b-42d4-a962-01de3d839949)
 
      * Cluster is malfunctioning: Malfunctioning clusters are rare, but if this is the case it can be difficult to determine what happened. You may just want to restart the cluster to see if this resolves the issue. You can also look into the logs to see if there’s anything suspicious. The Event log tab and Driver logs tabs, highlighted in the screenshot below, will be the places to look. You may want to enable Cluster log delivery in order to access the logs of the workers. You can also change the log level, but you might need to reach out to your Databricks account team for help.
        ![image](https://github.com/user-attachments/assets/a02fee7e-de6b-428b-b2b0-79d9be878b60)


* **Long Jobs: Diagnosing a long stage in Spark**
  * Start by identifying the longest stage of the job. Scroll to the bottom of the job’s page to the list of stages and order them by duration.
    ![image](https://github.com/user-attachments/assets/ff77489a-a626-4798-aa63-29223126137b)
  * To see high-level data about what this stage was doing, look at the Input, Output, Shuffle Read, and Shuffle Write columns (Make note of these numbers as you’ll likely need them later.):
    * Input: How much data this stage read from storage. This could be reading from Delta, Parquet, CSV, etc.
    * Output: How much data this stage wrote to storage. This could be writing to Delta, Parquet, CSV, etc.
    * Shuffle Read: How much shuffle data this stage read.
    * Shuffle Write: How much shuffle data this stage wrote.
  * The number of tasks in the long stage can point you in the direction of your issue. *If you see one task, that could be a sign of a problem. While this one task is running only one CPU is utilized and the rest of the cluster may be idle*. This happens most frequently in the following situations:
     * Expensive UDF on small data
     * Window function without PARTITION BY statement
     * Reading from an unsplittable file type. This means the file cannot be read in multiple parts, so you end up with one big task. Gzip is an example of an unsplittable file type.
     * Setting the multiLine option when reading a JSON or CSV file
     * Schema inference of a large file
     * Use of repartition(1) or coalesce(1)
  * If the stage has more than one task, you should investigate further. Click on the link in the stage’s description to get more info about the longest stage:
    ![image](https://github.com/user-attachments/assets/d0863790-15dc-4a6a-84c1-a5eb9e7bd5da)

* **Spark driver overloaded**
  * The most common reason for driver overload is that there are too many concurrent things running on the cluster. This could be too many streams, queries, or Spark jobs (some customers use threads to run many spark jobs concurrently). If you have too many things running on the cluster simultaneously, then you have three options:
     * Increase the size of your driver
     * Reduce the concurrency
     * Spread the load over multiple clusters
  * It could also be that you’re running non-Spark code on your cluster that is keeping the driver busy.

* **Spark stage high I/O**
  * High I/O: How much data needs to be in an I/O column to be considered high? To figure this out, first start with the highest number in any of the given columns. Then consider the total number of CPU cores you have across all our workers. Generally each core can read and write about 3 MBs per second (Divide your biggest I/O column by the number of cluster worker cores, then divide that by duration seconds).
    ![image](https://github.com/user-attachments/assets/bda162e4-86c6-4324-a8b4-b23f4fb99fa4)

  * High input: If you see a lot of input into your stage, that means you’re spending a lot of time reading data. First, identify what data this stage is reading using DAG. After you identify the specific data, here are some approaches to speeding up your reads:
     * Use Delta
     * Try Photon. It can help a lot with read speed, especially for wide tables
     * Make your query more selective so it doesn’t need to read as much data.
     * Reconsider your data layout so that data skipping is more effective.
     * If you’re reading the same data multiple times, use the Delta cache.
     * if you’re doing a join, consider trying to get DFP working.

  * High output: If you see a lot of output from your stage, that means you’re spending a lot of time writing data. Here are some approaches to resolving this:
     * Are you rewriting a lot of data? If you are rewriting a lot of data:
        * See if you have a merge that needs to be optimized.
        * Use deletion vectors to mark existing rows as removed or changed without rewriting the Parquet file.
        * Enable Photon if it isn’t already. Photon can help a lot with write speed.

* **Skew and spill**
   * The first thing to look for in a long-running stage is whether there’s spill
     ![image](https://github.com/user-attachments/assets/a5dcd3e1-5ec4-4feb-bd64-d1cc474a30e0)
   * Spill is what happens when Spark runs low on memory. It starts to move data from memory to disk, and this can be quite expensive. It is most common during data shuffling.
   * The default setting for the number of Spark SQL shuffle partitions (i.e., the number of CPU cores used to perform wide transformations such as joins, aggregations and so on) is 200, which isn’t always the best value. As a result, each Spark task (or CPU core) is given a large amount of data to process, and if the memory available to each core is insufficient to fit all of that data, some of it is spilled to disk. Spilling to disk is a costly operation, as it involves data serialization, de-serialization, reading and writing to disk, etc. Spilling needs to be avoided at all costs and in doing so, we must tune the number of shuffle partitions. There are a couple of ways to tune the number of Spark SQL shuffle partitions as discussed below.
      * (1)AQE auto-tuning (set spark.sql.shuffle.partitions=auto): Spark AQE has a feature called autoOptimizeShuffle (AOS), which can automatically find the right number of shuffle partitions.
         * **Caveat**: unusually high compression.There are certain limitations to AOS. AOS may not be able to estimate the correct number of shuffle partitions in some circumstances where source tables have an unusually high compression ratio (20x to 40x).There are two ways you can identify the highly compressed tables:
             *  Spark UI SQL DAG:
               ![image](https://github.com/user-attachments/assets/c0291b37-7c1a-497c-840b-d143960c4369)
Although “data size total” metrics in the Exchange node don’t provide the exact size of a table in memory, it can definitely help identify the highly compressed tables. Scan Parquet node provides the precise size of a table in the disk. The Exchange node data size in the aforementioned case is 40x larger than the size on the disk, indicating that the table is probably heavily compressed on the disk.

             * Cache the table: A table can be cached in memory to figure out its actual size in memory. Here’s how to go about it:
               ![image](https://github.com/user-attachments/assets/08182559-eec1-4175-b0b4-2ecd91c4e3a7)

               Refer to the storage tab of Spark UI to find the size of the table in memory after the command above has been completed:
               ![image](https://github.com/user-attachments/assets/588fa4ff-2f9d-4eb7-bbe5-bc0880d58914)
         

         * **Solution:** To counter this effect, reduce the value of the per partition size used by AQE to determine the initial shuffle partition number (default 128MB) as follows:     *set spark.databricks.adaptive.autoOptimizeShuffle.preshufflePartitionSizeInBytes = 16777216* (setting to 16MB for example).

           After lowering the preshufflePartitionSizeInBytes value to 16MB, if AOS is still calculating the incorrect number of partitions and you are still experiencing large data spills, you should further lower the preshufflePartitionSizeInBytes value to 8MB. If this still doesn’t resolve your spill issue, it is best to disable AOS and manually tune the number of shuffle partitions as explained in the next section.

      * (2)Manually fine-tune:
           



 
