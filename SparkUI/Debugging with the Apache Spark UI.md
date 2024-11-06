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
