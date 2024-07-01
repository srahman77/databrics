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
* To overcome the limitation of data skipping mentioned above, we have zordering.
* Zordering gpoups multiple statistics into single dimention
* Zordering organizes the data in such a way that makes the file skipping effective.
* So in normal file level stats, we have say stats collected on ID and Balance amount. Now the min and max of the balance amount of each file will cause lots of overlapping values. Hence _where id= or balance=_ will lead to scanning of almost all the files
* Now with ZORDER BY (id,balance) will collocate the related informtion (multi dimentional) in the same set of files.
* Zorder Recommendations:
    1) Commonly queries columns/join Keys should be used in Zorder columns
    2) Only Zorder by columns that have collected stats (_dataSkippingNumIndexedCols_ or _dataSkippingStatsColumns_)
    3) The effectiveness of Zordering drops with each added columns
    4) Zorder always comes with Optimize command
* Zorder only improves the data skipping. It does not give the perfact solution for 100% data skipping.
  ![image](https://github.com/srahman77/databrics/assets/58270885/936386a3-3fce-444c-8e5a-471890e25db4)



