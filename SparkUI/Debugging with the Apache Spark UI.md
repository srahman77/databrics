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
      * Driver is compiling a complex execution plan: 
      * Execution of non-spark code
      * Driver is overloaded
      * Cluster is malfunctioning

     


