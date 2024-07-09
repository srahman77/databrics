**Data Governance**
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
* UC metastore: Top level logical container in UC. It has the metadata about the objects that the metastore manages and the ACL that govern access to those objects.
* UC Metastore has a control plane that has the metadata about the the objects/ACLs. It also has Cloud storage tied to a region and the data related to the objects managed by the metastore is stored in the cloud storage .
* The MD of both managed and external tables are managed by the metastore in the control plane. The data of the managed table is stored in the cloud storage associated with the metastore mentioned in the above point.
* UC metastire has 5 componenet:
    1. Catalog: A UC metastore can 1000 catalog( need to contact databricks team if more needed). Catalog has tables,views ,volumes,Models and functions
    2. 
  
