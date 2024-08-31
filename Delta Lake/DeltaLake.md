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
* *https://learn.microsoft.com/en-us/azure/databricks/delta/tutorial* : for practice
* Delete from a table:

  deltaTable = DeltaTable.forName(spark, "main.default.people_10m")

  *Declare the predicate by using a SQL-formatted string.*

   deltaTable.delete("birthDate < '1955-01-01'")

  *Declare the predicate by using Spark SQL functions.*

   deltaTable.delete(col('birthDate') < '1960-01-01')

