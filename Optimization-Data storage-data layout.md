Before we start with Databricks query optimization, we should optimize how data is stored in the phyical location. It starts with selecting the right file formats and then arranging the data in the data lake. Here we will disucuss on the data layout in the blob location.

Here we will go through Hive-Style partitioning, Data Skipping, Z-Order and Liquid Clustering


**Hive-Style partitioning:**
Data is distributed across different folders based on the value of the partitioned column defined. The Partitioned column is not required to be provided in the schema as this column can be fetched by query engine from the folder structure.

* When to use Hive-Style partitioning?
i)We know how the data will be distributed for the partitioned keys. It should not create highly skewed partiotions
ii)Partition columns should have low cardianilty. Too many distinct values will create high number of partitions which will end up in creating small file problems. Hive-Style partition is advised when the table size is in TB level and each partition is in GB level. Fewer files to read is better, but again it should not be like a single too large file. That will eventually start to impact read performance. So the balance is -GB level partitions

* Limitation? After defining partition columns, they cannot be changed directly. We need to create temp table, recreate the main table with updated partitions columns and then reload the the data from temp to main table.


**Data Skipping using File statistics:**
* Used to skip files that do not have the predicate
* File level statistics are collected for first 32 columns by default. The value can be configured by #dataSkippingNumIndexedCols#


