# Memory Management

**https://community.cloudera.com/t5/Community-Articles/Spark-Memory-Management/ta-p/317794#:~:text=Execution%20Memory%20is%20used%20for,for%20the%20hash%20aggregation%20step**
* If the memory allocation is too large when committing, it will occupy resources. If the memory allocation is too small, memory overflow and full-GC problems will occur easily.
* Spark memory management (There will only be one MemoryManager per JVM.) is divided into two types:
    * Static Memory Manager (Static Memory Management- eliminated in Spark 3.0): The size of storage memory, execution memory, and other memory is fixed during application processing, but users can configure it before the application starts.
    * Unified Memory Manager (Unified memory management -default): Execution Memory and Storage Memory are dynamically adjusted. It allocates a region of memory as a unified memory container that is shared by storage and execution.
* JVM memory management is categorized into two types:
    1. On-Heap Memory Management (In-Heap Memory): By default, Spark uses on-heap memory only. Objects are allocated on the JVM Heap and bound by GC.
        * The size of the on-heap memory is configured by the *--executor-memory or spark.executor.memory*.
        * The concurrent tasks running inside Executor share the JVM's on-heap memory. Two main configurations control executor memory allocation:
        ![image](https://github.com/user-attachments/assets/9885f99a-6206-47bb-90f8-21668951f8c7)
        * Apache Spark supports three memory regions:
          ![image](https://github.com/user-attachments/assets/c5f9dfd2-f115-4807-ae3c-1292874d8c7c)
            1. Reserved Memory: Reserved memory is the memory reserved for the system and is used to store Spark's internal objects. Reserved memory’s size is hard coded, and its size cannot be changed in any way without Spark recompilation or setting spark.testing.reservedMemory, which is not recommended as it is a testing parameter not intended to be used in production.  If the executor memory is less than 1.5 times the reserved memory (1.5 * reserved memory = 450MB), then the Spark job will fail with the following exception message:java.lang.IllegalArgumentException: Executor memory 314572800 must be at least 471859200. Please increase executor memory using the --executor-memory option or spark.executor.memory in Spark configuration.
            2. User Memory: User Memory is the memory used to store user-defined data structures, Spark internal metadata, any UDFs created by the user, and the data needed for RDD conversion operations, such as RDD dependency information, etc. For example, we can rewrite Spark aggregation by using the mapPartitions transformation to maintain a hash table for this aggregation to run, which would consume so-called user memory. This memory segment is not managed by Spark. Spark will not be aware of or maintain this memory segment.
            3. Spark Memory: Spark Memory is the memory pool managed by Apache Spark. Spark Memory is responsible for storing intermediate states while doing task execution like joins or storing broadcast variables. All the cached or persistent data will be stored in this segment. This has 2 main segments:
                  * **Storage Memory**: Storage Memory is used for storing all of the cached data, broadcast variables, unrolled data, etc. “Unroll” is essentially a process of deserializing serialized data. Any persistent option, that includes memory in it, will store that data in this segment. Spark clears space for new cache requests by removing old cached objects based on the Least Recently Used (LRU) mechanism.Once the cached data is out of storage, it is either written to disk or recomputed based on configuration. Broadcast variables are stored in the cache at the MEMORY_AND_DISK persistent level. This is where we store cached data, which is long-lived. Storage memory blocks can be evicted from memory and written to disk or recomputed (if the persistence level is MEMORY_ONLY) as required.
                  * **Execution Memory**: Execution Memory is used for storing the objects required during the execution of Spark tasks. For example, it is used to store the shuffle intermediate buffer on the map side in memory. Also, it is used to store a hash table for the hash aggregation step. This pool also supports spilling on disk if not enough memory is available. But Due to the nature of execution memory, blocks cannot be forcefully evicted from this pool; otherwise, execution will break since the block it refers to won’t be found.Execution memory tends to be more short-lived than storage. It is evicted immediately after each operation, making space for the next ones.
                  * Storage and execution pool borrowing rules:
                      * Storage memory can borrow space from execution memory only if blocks are not used in execution memory.
                      * Execution memory can also borrow space from storage memory if blocks are not used in storage memory.
                      * If blocks from Execution memory is used by Storage memory and Execution needs more memory, it can forcefully evict the excess blocks occupied by Storage Memory
                      * If blocks from storage memory are used by execution memory and storage needs more memory, it cannot forcefully evict the excess blocks occupied by execution memory; it will have less memory area. It will wait until Spark releases the excess blocks stored in execution memory and then occupies them.
    2. Off-Heap Memory (External memory):
         * Off-Heap memory means allocating memory objects (serialized to byte array) to memory outside the heap of the Java virtual machine(JVM), which is directly managed by the operating system (not the virtual machine), but stored outside the process heap in native memory (therefore, they are not processed by the garbage collector).
         * The result of this is to keep a smaller heap to reduce the impact of garbage collection on the application.
         * Accessing this data is slightly slower than accessing the on-heap storage, but still faster than reading/writing from a disk. The downside is that the user has to manually deal with managing the allocated memory.
         * Spark can use sum of execution memory from on heap and off heap as execution memory. Same is the case for storage memory.
    3.  In addition to the above two JVM memory types, there is one more segment of memory that is accessed by Spark, i.e., external process memory. This kind of memory is mainly used for PySpark and SparkR applications. This is the memory used by the Python/R process, which resides outside the JVM.
* **Calculate the memory for 5 GB of executor memory: Just for pure spark(not databricks)**
    * To calculate reserved memory, user memory, spark memory, storage memory, and execution memory, we will use the following parameters:
      ![image](https://github.com/user-attachments/assets/d3bf4ab7-455b-46ca-aed0-d3d2084442d3)
      ![image](https://github.com/user-attachments/assets/070c3689-1bfa-44ea-97cc-0724bf424fde)


     

    
    

