* **There are mltiple reasons that causes OOM even though spark allows data spill to disk**

1. Spilling does not happen in real-time as fast as the data is in being read/written in memory meaning Spark can still be in the process of spilling to disk.Once the spill starts, the data is going to be serialized prior to the spill. If the memory is already full, then the serialization cannot be carried out effectively. Hence an OOM error occurs.
2. Even if the disk space is super large, it cannot be the one stop solution as - if the data is larger, Spark won't be able to spill additional data to disk which can lead to OutOfMemoryError.
4. User Memory :While storage memory and execution memory can spill to the disk, User memory can not. This means if user memory exceeds the limit it throws OOM Error. (User Memory: 25% of allocated executor memory. This section is used to store all user-defined data objects (eg., Hashmap, UDFs from user, etc) that are needed for RDD conversion operations. User memory is managed by Spark)
5. Spark can spill data to block to vacate the memory for processing, however most of the in-memory structures are not split able. Spark cannot split in-memory data structures used for processing work such as sorting, hashing, aggregation, join, hence they must fit in the available on heap memory and we cannot spill half of the data structures on to disk.
6. Spill to disk and garage collection are features of JVM on-heap, but the off heap memory is not subject to spill and garbage collection. Spark overhead memory is always off heap and out of memory exception are caused by overhead memory exceeding its limit.
7. Example from Stack overflow: Let's understand this with one example: consider a CSV file with 10GB and executor memory as 1 GB. And HDFS block size as 128 MB

    * Case 1: 10GB file with each record (i.e row / line ) 1 MB we have approx. 10K records. 
There would be approx 80 blocks. These blocks would be internally separated by records. Meaning a single record will not span across two blocks. 
Now, in this case, the file is read in 128MB part which is obviously lesser than 1GB executor space; also InputSplit does not have to access multiple blocks. Hence the file will be processed smoothly without OOM.
    * Case 2: 10GB file with each record 1.5 GB we have approx 6-7 records. Again there would be approx 80 blocks. But these blocks are linked as the record in one block is spilling to another block. So to read 1 record you have to access 12 blocks simultaneously. Now when the spark is reading the first block of 128 MB it sees(InputSplit) that the record is not finished, it has to read the second blocks as well and it continues till the 8th block(1024MB). Now when it tries to read the 9th block it cannot fit into 1 GB memory and hence it gives OOM exception.  best example is XML files or gz compressed csv files which are not splittable.  




