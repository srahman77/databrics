# Data Governance
* 4 key functionalities of data governance are:
  1. Data Access Control- Who has access to which data
  2. Data access audit- Capture and record all access to data
  3. Data Lineage- Capture upstream sources and downstream consumers
  4. Data Discovery- Ability to search for and discover authorised assets
* Why need seperate Data Governance when cloud already provides data security?
    The built in data security that the cloud vendors provide is for file level access control. There is no access policy for row/level column level control. Only option is to create a file with required sub rows/columns - causing duplicate data. Also, even on file level, if the data team needs to restructure the files, then the access control on the files also needs to be re-applied accordingly.
  * Here comes the UC for unify governance accross clouds
  * Before UC, databricks provided some level of access control, but the security model was permissive by default. It used to requre a carefull adminstration of access control list and the compute resources accessing the data to yield a secure solution. Also the ACLs were used to define as a property of the workspace, so sclaing the project to multi worskpace or multi cloud environment was a real challange. UC lives outside the workspace! So it spans workspace and clouds (__Unify governance accross clouds__).
  * Unify Data and AI assets: Centrally share,audit,secure and manage all data types. No need to replicate security settings across different systems. Also, UC can perform audits on any query on the data. Data lineage also captures and displayed for all tables
  * Unify existing catalogs without much hard work
* Legacy Data governance solution: Table access control and Azure Data Lake Storage credential passthrough (legacy data governance feature that allows you authenticate automatically to Azure Storage from Azure Databricks clusters using the same Microsoft Entra ID identity that you use to log into Azure Databricks-- what we used previously)
* UC Vs NonUC major difference:
![image](https://github.com/user-attachments/assets/798cda58-d243-4e73-925f-bbdcb33bfbf5)

* Key features of Unity Catalog include:
  * Define once, secure everywhere: Unity Catalog offers a single place to administer data access policies that apply across all workspaces.
  * Standards-compliant security model: Unity Catalog’s security model is based on standard ANSI SQL and allows administrators to grant permissions in their existing data lake using familiar syntax, at the level of catalogs, schemas (also called databases), tables, and views.
  * Built-in auditing and lineage: Unity Catalog automatically captures user-level audit logs that record access to your data. Unity Catalog also captures lineage data that tracks how data assets are created and used across all languages.
Data discovery: Unity Catalog lets you tag and document data assets, and provides a search interface to help data consumers find data.
  * System tables (Public Preview): Unity Catalog lets you easily access and query your account’s operational data, including audit logs, billable usage, and lineage.

* UC top level arch:
  ![image](https://github.com/user-attachments/assets/6650120d-d260-4caa-8638-d32f69297d86)
* External locations allow Unity Catalog to read and write data on your cloud tenant on behalf of users. External locations are defined as a path to cloud storage, combined with a storage credential that can be used to access that location. You can use external locations to register external tables and external volumes in Unity Catalog. The content of these entities is physically located on a sub-path in an external location that is referenced when a user creates the volume or the table
* A storage credential encapsulates a long-term cloud credential that provides access to cloud storage. It can either be an Azure managed identity (strongly recommended) or a service principal
* Using an Azure managed identity has the following benefits over using a service principal: Managed identities do not require you to maintain credentials or rotate secrets. And if your Azure Databricks workspace is deployed to your own VNet (also known as VNet injection), you can connect to an Azure Data Lake Storage Gen2 account that is protected by a storage firewall.
* For increased data isolation, you can bind storage credentials and external locations to specific workspaces.
* Once you have external locations configured in Unity Catalog, you can create external tables and volumes on directories inside the external locations. You can then use Unity Catalog to manage user and group access to these tables and volumes. This allows you to provide specific users or groups access to specific directories and files in the cloud storage container
* Metastore: The metastore is the top-level container for metadata in Unity Catalog. It registers metadata about data and AI assets and the permissions that govern access to them. You should have one metastore for each region in which you have workspaces. Typically, a metastore is created automatically when you create a Azure Databricks workspace in a region for the first time
* Every workspace attached to a single regional metastore has access to the data managed by the metastore. If you want to share data between metastores, use Delta Sharing.
* Each metastore can be configured with a managed storage location (also called root storage) in your cloud tenant that can be used to store managed tables and managed volumes.
* Data isolation using managed storage: Unity Catalog gives the ability to configure storage locations at the metastore, catalog, or schema level.let’s say your organization has a company compliance policy that requires production data relating to human resources to reside in the container abfss://mycompany-hr-prod@storage-account.dfs.core.windows.net. In Unity Catalog, you can achieve this requirement by setting a location on a catalog level, creating a catalog called, for example hr_prod, and assigning the location abfss://mycompany-hr-prod@storage-account.dfs.core.windows.net/unity-catalog to it. This means that managed tables or volumes created in the hr_prod catalog (for example, using CREATE TABLE hr_prod.default.table …) store their data in abfss://mycompany-hr-prod@storage-account.dfs.core.windows.net/unity-catalog.
* If you choose to create a metastore-level managed location, you must ensure that no users have direct access to it (that is, through the cloud account that contains it). Giving access to this storage location could allow a user to bypass access controls in a Unity Catalog metastore and disrupt auditability.
* The system evaluates the hierarchy of storage locations from schema to catalog to metastore.
* Workspace-catalog binding: By default, catalog owners can make a catalog accessible to users in multiple workspaces attached to the same Unity Catalog metastore.
* For increased data isolation, you can also bind cloud storage access to specific workspaces. See (Optional) Assign a storage credential to specific workspaces and (Optional) Assign an external location to specific workspaces.
* Lakehouse Federation is the query federation platform for Azure Databricks. The term query federation describes a collection of features that enable users and systems to run queries against multiple siloed data sources (externan data sources) without needing to migrate all data to a unified system. Azure Databricks uses Unity Catalog to manage query federation. You use Unity Catalog to configure read-only connections to popular external database systems and create foreign catalogs that mirror external databases.
* Delta Sharing is a secure data sharing platform that lets you share data and AI assets with users outside your organization, whether or not those users use Databricks. Although Delta Sharing is available as an open-source implementation, in Databricks it requires Unity Catalog to take full advantage of extended functionality.

# How do I set up Unity Catalog for my organization?
  * To use Unity Catalog, your Azure Databricks workspace must be enabled for Unity Catalog, which means that the workspace is attached to a Unity Catalog metastore
  * All new workspaces are enabled for Unity Catalog automatically upon creation, but older workspaces might require that an account admin enable Unity Catalog manually
  * If your workspace was not enabled for Unity Catalog automatically, an account admin or metastore admin must manually attach the workspace to a Unity Catalog metastore in the same region. If no Unity Catalog metastore exists in the region, an account admin must create one
  * Whether or not your workspace was enabled for Unity Catalog automatically, the following steps are also required to get started with Unity Catalog: i)Create catalogs and schemas to contain database objects like tables and volumes.
ii)Create managed storage locations to store the managed tables and volumes in these catalogs and schemas. iii) Grant user access to catalogs, schemas, and database objects.
  * Workspaces that are automatically enabled for Unity Catalog provision a workspace catalog with broad privileges granted to all workspace users. This catalog is a convenient starting point for trying out Unity Catalog.
  * To run Unity Catalog workloads, compute resources must comply with certain security requirements.SQL warehouses always comply with Unity Catalog requirements, but some cluster access modes do not (only shared and single user do)

# Migrating an existing workspace to Unity Catalog involves the following:
  * Converting any workspace-local groups to account-level groups. Unity Catalog centralizes identity management at the account level.
  * Migrating tables and views managed in Hive metastore to Unity Catalog.
  * Update queries and jobs to reference the new Unity Catalog tables instead of the old Hive metastore tables.
* Hive metastore table migration to UC: https://learn.microsoft.com/en-us/azure/databricks/data-governance/unity-catalog/migrate
* To access data in Unity Catalog, clusters must be configured with the correct access mode. Unity Catalog is secure by default. If a cluster is not configured with shared or single-user access mode, the cluster can’t access data in Unity Catalog (Other access modes are no isolation shared and custom- these 2 does are not supported.)
* Unity Catalog supports the following table formats:
  * Managed tables must use the delta table format.
  * External tables can use delta, CSV, JSON, avro, parquet, ORC, or text.
 
* UC limitation:
  * Bucketing is not supported for Unity Catalog tables. If you run commands that try to create a bucketed table in Unity Catalog, it will throw an exception.
  * Writing to the same path or Delta Lake table from workspaces in multiple regions can lead to unreliable performance if some clusters access Unity Catalog and others do not.
  * Custom partition schemes created using commands like ALTER TABLE ADD PARTITION are not supported for tables in Unity Catalog. But Unity Catalog can access tables that use directory-style partitioning.
  * Unity Catalog manages partitions differently than Hive. Hive commands that directly manipulate partitions are not supported on tables managed by Unity Catalog.
  * Table history is not migrated when you run CREATE TABLE CLONE. Any tables in the Hive metastore that you clone to Unity Catalog are treated as new tables. You cannot perform Delta Lake time travel or other operations that rely on pre-migration history.
  * Unity Catalog enforces resource quotas on all securable objects. If you expect to exceed these resource limits, contact your Azure Databricks account team.. E.g table/schema=	10000; table/metastore=1000000 etc
 
# UC best practices
* It is necessary to have data isolation boundaries between environments (such as development and production) or between organizational operating units. Isolation standards might vary for your organization, but typically they include the following expectations:
  * Users can only gain access to data based on specified access rules.
  * Data can be managed only by designated people or teams.
  * Data is physically separated in storage.
  * Data can be accessed only in designated environments.
* The need for data isolation can lead to siloed environments that can make both data governance and collaboration unnecessarily difficult. Azure Databricks solves this problem using Unity Catalog
* Centralized governance model: your governance administrators are owners of the metastore and can take ownership of any object and grant and revoke permissions.
* Distributed governance model: the catalog or a set of catalogs is the data domain. The owner of that catalog can create and own all assets and manage governance within that domain.
  ![image](https://github.com/user-attachments/assets/8daf1409-4d38-43ff-beac-2bc8f521416a)
* In Databricks, the workspace is the primary data processing environment, and catalogs are the primary data domain. Unity Catalog lets metastore admins and catalog owners assign, or “bind,” catalogs to specific workspaces. These environment-aware bindings give you the ability to ensure that only certain catalogs are available within a workspace, regardless of the specific privileges on data objects granted to a user.
* Securing access using dynamic view
    * Column level:
          CREATE VIEW < catalog_name >.< schema_name >.< view_name > as SELECT  id,  CASE WHEN is_account_group_member(< group_name >) THEN email ELSE 'REDACTED' END AS email,  country,  product,  total FROM  < catalog_name >.< schema_name >.< table_name >;
  
          GRANT USE CATALOG ON CATALOG < catalog_name > TO < group_name >;
          GRANT USE SCHEMA ON SCHEMA < catalog_name >.< schema_name >.< view_name > TO < group_name >;
          GRANT SELECT   ON < catalog_name >.< schema_name >.< view_name > TO < group_name >;
    * Row level:
          CREATE VIEW < catalog_name >.< schema_name >.< view_name > as SELECT  * FROM   < catalog_name >.< schema_name >.< table_name > WHERE  CASE WHEN is_account_group_member(managers) THEN TRUE ELSE total <= 1000000 END;
* We can also secure access to tables using row filters and columns masks bu using Catalog explorer or by creating UDFs. e.g
    * Filter:
      Create the UDF:
      CREATE FUNCTION us_filter(region STRING)
      RETURN IF(IS_ACCOUNT_GROUP_MEMBER('admin'), true, region='US');

      Apply the UDF on the table
      CREATE TABLE sales (region STRING, id INT);
      ALTER TABLE sales SET ROW FILTER us_filter ON (region);
    
    * Masking:
      Create the UDF:
      CREATE FUNCTION ssn_mask(ssn STRING)
      RETURN CASE WHEN is_member('HumanResourceDept') THEN ssn ELSE '***-**-****' END;

      Apply the UDF on the table:
      --Create the `users` table and apply the column mask in a single step:
        CREATE TABLE users ( name STRING,  ssn STRING MASK ssn_mask);

      --Create the `users` table and apply the column mask after:
        CREATE TABLE users  (name STRING, ssn STRING);
        ALTER TABLE users ALTER COLUMN ssn SET MASK ssn_mask;
* Use dynamic views if you need to apply transformation logic such as filters and masks to read-only tables, and if it is acceptable for users to refer to the dynamic views using different names. If you want to filter data when you share it using Delta Sharing, you must use dynamic views. Use row filters and column masks if you want to filter or compute expressions over specific data but still provide users access to the tables using their original names.

  ## Refer the Databricks Docs for the other admin type of details. The above are the basic about UC**
    
