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

* Metastire: The metastore is the top-level container for metadata in Unity Catalog. It registers metadata about data and AI assets and the permissions that govern access to them. You should have one metastore for each region in which you have workspaces. Typically, a metastore is created automatically when you create a Azure Databricks workspace in a region for the first time
* Data isolation using managed storage: Unity Catalog gives the ability to configure storage locations at the metastore, catalog, or schema level.let’s say your organization has a company compliance policy that requires production data relating to human resources to reside in the container abfss://mycompany-hr-prod@storage-account.dfs.core.windows.net. In Unity Catalog, you can achieve this requirement by setting a location on a catalog level, creating a catalog called, for example hr_prod, and assigning the location abfss://mycompany-hr-prod@storage-account.dfs.core.windows.net/unity-catalog to it. This means that managed tables or volumes created in the hr_prod catalog (for example, using CREATE TABLE hr_prod.default.table …) store their data in abfss://mycompany-hr-prod@storage-account.dfs.core.windows.net/unity-catalog.
* Workspace-catalog binding: By default, catalog owners can make a catalog accessible to users in multiple workspaces attached to the same Unity Catalog metastore.
* For increased data isolation, you can also bind cloud storage access to specific workspaces. See (Optional) Assign a storage credential to specific workspaces and (Optional) Assign an external location to specific workspaces.
* Lakehouse Federation is the query federation platform for Azure Databricks. The term query federation describes a collection of features that enable users and systems to run queries against multiple siloed data sources (externan data sources) without needing to migrate all data to a unified system. Azure Databricks uses Unity Catalog to manage query federation. You use Unity Catalog to configure read-only connections to popular external database systems and create foreign catalogs that mirror external databases.
* Delta Sharing is a secure data sharing platform that lets you share data and AI assets with users outside your organization, whether or not those users use Databricks. Although Delta Sharing is available as an open-source implementation, in Databricks it requires Unity Catalog to take full advantage of extended functionality.

# How do I set up Unity Catalog for my organization?
  * To use Unity Catalog, your Azure Databricks workspace must be enabled for Unity Catalog, which means that the workspace is attached to a Unity Catalog metastore
  * All new workspaces are enabled for Unity Catalog automatically upon creation, but older workspaces might require that an account admin enable Unity Catalog manually
  * Whether or not your workspace was enabled for Unity Catalog automatically, the following steps are also required to get started with Unity Catalog: i)Create catalogs and schemas to contain database objects like tables and volumes.
ii)Create managed storage locations to store the managed tables and volumes in these catalogs and schemas. iii) Grant user access to catalogs, schemas, and database objects.
  * Workspaces that are automatically enabled for Unity Catalog provision a workspace catalog with broad privileges granted to all workspace users. This catalog is a convenient starting point for trying out Unity Catalog.

