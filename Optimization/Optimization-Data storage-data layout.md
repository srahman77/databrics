Before we start with Databricks query optimization, we should optimize how data is stored in the phyical location. It starts with selecting the right file formats and then arranging the data in the data lake. Here we will disucuss on the data layout in the blob location.

Here we will go through Hive-Style partitioning, Data Skipping, Z-Order and Liquid Clustering


**Hive-Style partitioning:**
Data is distributed across different folders based on the value of the partitioned column defined. The Partitioned column is not required to be provided in the schema as this column can be fetched by query engine from the folder structure.

* When to use Hive-Style partitioning?
i)We know how the data will be distributed for the partitioned keys. It should not create highly skewed partiotions
ii)Partition columns should have low cardianilty. Too many distinct values will create high number of partitions which will end up in creating small file problems. Hive-Style partition is advised when the table size is in TB level and each partition is in GB level. Fewer files to read is better, but again it should not be like a single too large file. That will eventually start to impact read performance. So the balance is -GB level partitions

* Limitation? After defining partition columns, they cannot be changed directly. We need to create temp table, recreate the main table with updated partitions columns and then reload the the data from temp to main table.


**Data Skipping using File statistics:**
* Data skipping information is collected automatically when you write data into a **Delta table**. Delta Lake takes advantage of this information (minimum and maximum values, null counts, and total records per file) at query time to provide faster queries.
* Used to skip files that do not have the predicate
* File level statistics are collected for first 32 columns by default. The value can be configured by _dataSkippingNumIndexedCols_
* So use Keys,High Cardianility columns to the left side and timestamp, long string etc to the right of the _dataSkippingNumIndexedCols_ configyred values.
* _dataSkippingStatsColumns_ (Databricks Runtime 13.3 LTS and above): Specify a list of column names for which Delta Lake collects statistics! _Supersedes dataSkippingNumIndexedCols_
* The information for each file has 1) the file name 2)num of records in the file 3) min,max,null counts for the required columns
**_Note_**: Parquet stores min/max statistics for row groups, but those are not stored inside the row groups themselves but in the file footer instead. As a result, if none of the row groups match, then it is not necessary to read any part of the file other than the footer. Whereas Delta lake stores the stats directly for file level (using the row group level details?).So the combination of file level stats from delta lake (which file to check) and the row group level information from those particular concerned parquet files (which row group to check now) gives the best search result!
* Limitation: When we have say OR condition in the predicate with a column that can have overlapping values like say  balance. The same "balance amount" can spread across mutliple files causing scanning of almost all the files, then data stats does not work much. Basically unreated columns in the predicate can make data skipping ineffective.

**Zordering**
 CC: _https://www.linkedin.com/pulse/z-order-visualization-implementation-nick-karpov_; _https://www.youtube.com/watch?v=5t6wX28JC_M_ (_Delta Lake Deep Dive: Liquid Clustering_)

* To overcome the limitation of data skipping mentioned above, we have zordering.
* Z-Order in Delta Lake is a sort and repartition. Its actual behavior and implementation is to sort the data into buckets, and repartition the data according to those buckets. When we sort the data into buckets, we use a special kind of sorting that collocates, or places similar data into the same bucket.
* Z-ordering is not idempotent but aims to be an incremental operation. The time it takes for Z-ordering is not guaranteed to reduce over multiple runs. However, if no new data was added to a partition that was just Z-ordered, another Z-ordering of that partition will not have any effect.
* Z-ordering aims to produce evenly-balanced data files with respect to the number of tuples, but not necessarily data size on disk
* Z-Order in Delta Lake is a heuristic, not an index. An index in a database is a data structure that provides a direct link to a single row of data for fast retrieval. Z-Order can indicate which files might have the required data, but cannot confirm exactly which file contains the exact required rows, like an index. So the query might still read files that don’t have the required data in order to guarantee the correct result.
* In normal file level stats, we have say stats collected on ID and Balance amount. Now the min and max of the balance amount of each file will cause lots of overlapping values. Hence _where id= or balance=_ will lead to scanning of almost all the files
* Now with ZORDER BY (id,balance) will collocate the related informtion (groups multiple statistics into single dimention) in the same set of files.
* Zorder Recommendations:
    1) Commonly queries columns/join Keys should be used in Zorder columns
    2) Only Zorder by columns that have collected stats (_dataSkippingNumIndexedCols_ or _dataSkippingStatsColumns_)
    3) The effectiveness of Zordering drops with each added columns. And each column adds in the Zorder tries to eliminate the intersection for the respective columns. So more columns added means more dimensional points to be intersection free- which starts to become impossible/ineffective
    4) Zorder always comes with Optimize command
* Zorder only improves the data skipping. It does not give the perfact solution for 100% data skipping. (Best case scenario is no intersection- hence perfact data skipping)
* To reduce itersection (reduce the skewness in the interleaved column genetrated for zorder), Delta Lake Spark connector implementation uses an additional noise column to reduce skew. This is enabled by default that adds one more column in addition to the user specified columns when producing the z-value, .
  ![image](https://github.com/srahman77/databrics/assets/58270885/936386a3-3fce-444c-8e5a-471890e25db4)
  ( credit: https://www.youtube.com/watch?v=5t6wX28JC_M)
* In the above pic, we can see that 1st and last partition have overlapping ids. So it did not resolve the unwanted file scanning completely- but improved. e.g if we need any id between 35300 to 71000 0r any balance between 20000 and 30000, we need to scan only 2nd partition.
* For DF:  df.optimize().executeZOrderBy("x","y")
  For SQL: OPTIMIZE Detla_Table ZORDER BY ("x","y")

* Zorder Limitation:
    1)Optimize Zorder By rewrites all the data - high write operation
    2)It runs as a single operation and there is no intermendiate checkpoints for failover recovery
    3)No table level defination of Zorder. Hence everytime we run zorder, we need to provide the list of columns also.

** Unlike Hive partitioning and Z-ordering, liquid clustering keys can be chosen purely based on the query predicate pattern, with no worry about future changes, cardinality, key order, file size, and potential data skew. Liquid clustering is flexible and adaptive (hence, its name), meaning that

   1. Clustering keys can be changed without necessarily rebuilding the entire table
   2. It eliminates the concept of partitions and can dynamically merge or further divide files in order to arrive at a balanced dataset with the ideal number and size of files.

