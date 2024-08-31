* Delta Lake is the optimized storage layer that provides the foundation for tables in a lakehouse on Databricks
* In pyspark, for create or replace: *df.writeTo("main.default.people_10m").createOrReplace()*;                                                for new table: *df.saveAsTable("main.default.people_10m")*
* In spark sql, to insert data to a table,
  *COPY INTO main.default.people_10m
  FROM '/Volumes/main/default/my-volume/export.csv'
  FILEFORMAT = CSV
  FORMAT_OPTIONS ( 'header' = 'true', 'inferSchema' = 'true' )*
* In Databricks Runtime 13.3 LTS and above, you can use CREATE TABLE LIKE to create a new empty Delta table that duplicates the schema and table properties for a source Delta table
* To use pyspark for creating empty tables:
  *DeltaTable.createIfNotExists(spark)
  .tableName("main.default.people_10m")
  .addColumn("id", "INT")
  .addColumn("firstName", "STRING")
  .addColumn("middleName", "STRING")
  .addColumn("lastName", "STRING", comment = "surname")
  .addColumn("gender", "STRING")
  .addColumn("birthDate", "TIMESTAMP")
  .addColumn("ssn", "STRING")
  .addColumn("salary", "INT")
  .execute()*
* Merge in Pyspark:
  
  *from delta.tables import DeltaTable \n
  deltaTable = DeltaTable.forName(spark, 'main.default.people_10m')*

  *(deltaTable.alias("people_10m")
  .merge(
    people_10m_updates.alias("people_10m_updates"),
    "people_10m.id = people_10m_updates.id")
  .whenMatchedUpdateAll()
  .whenNotMatchedInsertAll()
  .execute())*

  we cannot directly use the target table(main.default.people_10m) in the merge, hence we use the 1st 2 steps to create a DeltaTable object for the target table and then appy merge on that table. For Spark SQl, we directly use the target table for merge like we do
* Appendig data to a table: equivalent of insert into of sql in pyspark:

  df.write.mode("append").saveAsTable("main.default.people_10m")

  Use .mode("overwrite") for insert overwrite

* Update in pyspark:

  from delta.tables import *
  from pyspark.sql.functions import *

  deltaTable = DeltaTable.forName(spark, "main.default.people_10m")

   *Declare the predicate by using a SQL-formatted string.*

   deltaTable.update(
   condition = "gender = 'F'",
   set = { "gender": "'Female'" }
   )

  *Declare the predicate by using Spark SQL functions*

  deltaTable.update(
  condition = col('gender') == 'M',
  set = { 'gender': lit('Male') }
  )

* Delete from a table:

  deltaTable = DeltaTable.forName(spark, "main.default.people_10m")

  *Declare the predicate by using a SQL-formatted string.*

   deltaTable.delete("birthDate < '1955-01-01'")

  *Declare the predicate by using Spark SQL functions.*

   deltaTable.delete(col('birthDate') < '1960-01-01')
* Display table history (s=describe history in sql):

  deltaTable = DeltaTable.forName(spark, "main.default.people_10m")
  display(deltaTable.history())
* Query prev version:

  deltaTable = DeltaTable.forName(spark, "main.default.people_10m")
  deltaHistory = deltaTable.history()
  display(deltaHistory.where("version == 0"))

  **Or:**

  display(deltaHistory.where("timestamp == '2024-05-15T22:43:15.000+00:00'"))


  To Create a DF from prev version:

  df = spark.read.option('versionAsOf', 0).table("main.default.people_10m")
  **Or:**
  df = spark.read.option('timestampAsOf', '2024-05-15T22:43:15.000+00:00').table("main.default.people_10m")
  display(df)

* Optimize: After you have performed multiple changes to a table, you might have a lot of small files. To improve the speed of read queries, you can use the optimize operation to collapse small files into larger ones

   deltaTable = DeltaTable.forName(spark, "main.default.people_10m")
   deltaTable.optimize()

* Zorder By:

   deltaTable = DeltaTable.forName(spark, "main.default.people_10m")
   deltaTable.optimize().executeZOrderBy("gender")

*  Vacuum:
    from delta.tables import *
    deltaTable = DeltaTable.forName(spark, "main.default.people_10m")
    deltaTable.vacuum()
 
 * Some points on time travel:
   * Data files are deleted when VACUUM runs against a table. Delta Lake manages log file removal automatically after checkpointing table versions.
   * In order to increase the data retention threshold for Delta tables, you must configure the following table properties:
      * *delta.logRetentionDuration = "interval <interval>"*: controls how long the history for a table is kept. The default is interval 30 days.
      * *delta.deletedFileRetentionDuration = "interval <interval>"*: determines the threshold VACUUM uses to remove data files no longer referenced in the current table version. The default is interval 7 days.
      * *spark.databricks.delta.retentionDurationCheck.enabled to false*: by default there is a safety interval enabled. So if you set a retentionperiod lower than that interval (7 days), data in that interval will not be deleted TO delete the data for less than 7 days, set the retentionDurationCheck to false.
   * You need both the log and data files to time-travel to a previous version.

     Vacuum - does not delete the log files. It only deletes the data files, which are never deleted automatically unless you run the vacuum. Log files are automatically cleaned up after new checkpoints are added.

     logRetentionDuration - Each time a checkpoint is written, Databricks automatically cleans up log entries older than the retention interval. For *delta.logRetentionDuration'='interval  days’*, when a new checkpoint is written, it clears the logs older than 8 days. Once this happens, you should not be able to do time travel as log files are now unavailable for that version.
     


* *https://learn.microsoft.com/en-us/azure/databricks/delta/tutorial* : for practice
