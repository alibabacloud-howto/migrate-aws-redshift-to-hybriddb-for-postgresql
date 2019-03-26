# Migrating from Amazon Redshift to HybridDB for PostgreSQL

## Summary
1. [Introduction](#introduction)
2. [Data migration architecture and overall procedure](#data-migration-architecture-and-overall-procedure)
3. [Preparation on AWS](#preparation-on-aws)
    1. [Customer provides access to S3's credentials](#customer-provides-access-to-s3s-credentials)
    2. [Exported data format convention](#exported-data-format-convention)
    3. [Recommended Redshift UNLOAD command option](#recommended-redshift-unload-command-option)
    4. [Get the DDL statement of the object in the Redshift database](#get-the-ddl-statement-of-the-object-in-the-redshift-database)
4. [Preparation on Alibaba Cloud](#preparation-on-alibaba-cloud)
    1. [Prepare Alibaba Cloud RAM sub-account](#prepare-alibaba-cloud-ram-sub-account)
    2. [Prepare OSS Bucket](#prepare-oss-bucket)
    3. [Prepare OSSImport](#prepare-ossimport)
5. [Migrating data files from S3 to OSS using OSSImport](#migrating-data-files-from-s3-to-oss-using-ossimport)
    1. [Configuring OSSImport](#configuring-ossimport)
    2. [Starting the OSSImport Migration Task](#starting-the-ossimport-migration-task)
    3. [Monitoring task status](#monitoring-task-status)
    4. [Failed task retry (optional)](#failed-task-retry-optional)
    5. [Check the files migrated to the OSS Bucket (optional)](#check-the-files-migrated-to-the-oss-bucket-optional)
6. [Data scrubbing with csv files (optional)](#data-scrubbing-with-csv-files-optional)
7. [DDL conversion from Redshift to HybridDB for PostgreSQL](#ddl-conversion-from-redshift-to-hybriddb-for-postgresql)
    1. [Prepare CREATE SCHEMA](#prepare-create-schema)
    2. [Prepare CREATE FUNCTION](#prepare-create-function)
    3. [Prepare CREATE TABLE](#prepare-create-table)
    4. [Prepare CREATE VIEW](#prepare-create-view)
    5. [Prepare CREATE EXTERNAL TABLE](#prepare-create-external-table)
    6. [Prepare INSERT INTO](#prepare-insert-into)
    7. [Prepare VACUUM SORT](#prepare-vacuum-sort)
8. [Prepare HybridDB for PostgreSQL instance](#prepare-hybriddb-for-postgresql-instance)
    1. [Create and Config HybridDB for PostgreSQL instance](#create-and-config-hybriddb-for-postgresql-instance)
    2. [Create Database Objects](#create-database-objects)
    3. [Import OSS external table data into the data table](#import-oss-external-table-data-into-the-data-table)
9. [Support](#support)

## Introduction
In this document, you will learn how to migrate data from [AWS Redshift](https://aws.amazon.com/redshift/) to
[HybridDB for PostgreSQL](https://www.alibabacloud.com/product/hybriddb-postgresql).

## Data migration architecture and overall procedure
This migration process generally includes the following steps:
1. Select Alibaba Cloud related products and build the environment, with mainly
   [HybridDB for PostgreSQL](https://www.alibabacloud.com/product/hybriddb-postgresql) and
   [OSS](https://www.alibabacloud.com/product/oss);
2. Export the data to be migrated from a [AWS Redshift](https://aws.amazon.com/redshift/) database to
   [S3](https://aws.amazon.com/s3) at a certain point in time;
3. Configure and start [OSSImport](https://github.com/aliyun/ossimport) to transfer the generated CSV data file;
4. Build various objects for the [HybridDB for PostgreSQL](https://www.alibabacloud.com/product/hybriddb-postgresql)
   database, including [schemas](https://www.postgresql.org/docs/current/ddl-schemas.html),
   [tables](https://www.postgresql.org/docs/current/ddl-basics.html),
   [views](https://www.postgresql.org/docs/current/tutorial-views.html) and
   [functions](https://www.postgresql.org/docs/current/sql-createfunction.html);
5. Import data from the OSS external table into the data tables.

- **Batch import workflow:** Redshift → S3 → OSSImport on ECS → OSS → HybridDB for PostgreSQL
- **Incremental import workflow:** Application → S3 → Lambda → OSS → MNS/MQ (optional) → Function Compute → HybridDB for PostgreSQL

## Preparation on AWS
### Customer provides access to S3's credentials
Contains the following information
- AccessKeyID and AccessKey Secret.
- Endpoint of S3, such as: **s3.ap-southeast-2.amazonaws.com**
- Bucket name, such as: **alibaba-hybrid-export**

### Exported data format convention
- Export data to a CSV format file.
- The size of the export file is up to 50M.
- The order of the column values in the file is the same as the column order of the table creation statement.
- The number of files is preferably the same as the number of segments in HybridDB for PostgreSQL or a multiple of
  the number of segments.

### Recommended Redshift UNLOAD command option
After many practices, we believe that the Redshift UNLOAD option is best for HybridDB for PostgreSQL imports. Here is
an example:
```sql
unload ('select * from test')
to 's3://xxx-poc/test_export_'
access_key_id '<Your access key id>'
secret_access_key '<Your access key secret>'
DELIMITER AS ','
ADDQUOTES
ESCAPE
NULL AS 'NULL'
MAXFILESIZE 50 mb ;
```

In this UNLOAD command example, the following options are recommended:
```sql
DELIMITER AS ','
ADDQUOTES
ESCAPE
NULL AS 'NULL'
MAXFILESIZE 50 mb
```

### Get the DDL statement of the object in the Redshift database
Export all DDL statements from AWS Redshift, including but not limited to schema, table, function and view.

## Preparation on Alibaba Cloud
### Prepare Alibaba Cloud RAM sub-account
- RAM account ID
- RAM account Password
- RAM account Access Key ID
- RAM account Access Key Secret

### Prepare OSS Bucket
Create an Alibaba Cloud OSS Bucket in the same [region](https://www.alibabacloud.com/help/doc-detail/40654.htm)
as the AWS S3 Bucket, such as Sydney (ap-southeast-2).

After the OSS bucket is created, the Internet Access Endpoint addresses and VPC Network Access from ECS
(Internal Network) of the bucket are obtained from the [OSS Console](https://oss.console.aliyun.com).

### Prepare OSSImport
- Create an ECS instance in the same region as the bucket, with a network bandwidth of 100Mbps, here we use Windows x64
  as the operating system.
- Download and install the OSSImport tool on the created ECS. The latest version of the OSSImport tool can be found and
  obtained at https://www.alibabacloud.com/help/doc-detail/56990.htm
- Unzip the OSSImport package.

## Migrating data files from S3 to OSS using OSSImport
### Configuring OSSImport
1. In this practice, we use OSSImport in a stand-alone deployment mode.
2. Edit the `conf/local_job.cfg` file. In the following an example, only the parameters that must be modified in this
   practice are displayed:

    - srcType=s3
    - srcAccessKey="your AWS Access Key ID"
    - srcSecretKey="your AWS Access Key Secret"
    - srcDomain=s3.ap-southeast-2.amazonaws.com
    - srcBucket=alibaba-hybrid-export
    - destAccessKey="your Alibaba Cloud Access Key ID"
    - destSecretKey="your Alibaba Cloud Access Key Secret"
    - destDomain=http://oss-ap-southeast-2-internal.aliyuncs.com
    - destBucket=alibaba-hybrid-export-1
    - destPrefix=
    - isSkipExistFile=true

> **Note:** The detailed configuration instructions for OSSImport can be found at
> https://www.alibabacloud.com/help/doc-detail/56990.htm

### Starting the OSSImport Migration Task
In the OSSImport stand-alone deployment mode, you can start the migration task by executing `import.bat`.

### Monitoring task status
During the data migration process, you can observe the output of the command execution window. In addition, you
can observe the usage of the network bandwidth through the resource manager.

In this example, since the ECS and the OSS Bucket are deployed in the same region, the network speed between data
uploading from the ECS to the OSS Bucket (intranet Endpoint) is not limited. However, since data is downloaded from
S3 to OSS via internet, the speed of ECS→OSS is virtually the same as the speed of S3→ECS, and the upload speed
is limited by the download speed.

### Failed task retry (optional)
Sub-tasks may fail due to network or other reasons. Failure Retry only retry the failed Task and will not retry the
successful Task. Execute `console.bat retry` in a terminal in Windows.

### Check the files migrated to the OSS Bucket (optional)
Files can be checked through the OSS Console. We also recommend using the **Ossbrowser** client tool to view and
manipulate files in the bucket.

Ossbrowser can be found here: https://www.alibabacloud.com/help/doc-detail/61872.htm

## Data scrubbing with csv files (optional)
> **Note:** This is only an example. It does not mean that you must complete this operation. You can perform certain
> data cleaning operations on the CSV file according to your business needs.

- Replace `NULL` in the CSV files with a blank character.
- Replace `\,` in the csv files with `,`.

For this data scrubbing operation, we recommend doing it locally. Therefore, the data file that needs to be scrubbed
is first downloaded to the ECS through the Ossbrowser tool, and then the data scrubbing operation is performed; then,
the data scrubbed files are uploaded to another newly created bucket to be distinguished from the original CSV files.
When downloading the original CSV files and uploading the cleaned files, we recommend that Ossbrowser use OSS's
intranet Endpoint for communication, so that intranet traffic charges will be reduced.

## DDL conversion from Redshift to HybridDB for PostgreSQL
In this chapter, we will describe the necessary preparations before creating the HybridDB for PostgreSQL database
objects, mainly converting DDL statements in the Redshift syntax format into the HybridDB for PostgreSQL syntax
format. At the same time, we will briefly describe the conversion rules.

### Prepare CREATE SCHEMA
This is a sample that conforms to the PostgreSQL syntax format, you can save it to `create schema.sql`:
```sql
CREATE SCHEMA schema1
  AUTHORIZATION xxxpoc;
GRANT ALL ON SCHEMA schema1 TO xxxpoc;
GRANT ALL ON SCHEMA schema1 TO public;
COMMENT ON SCHEMA model IS 'for xxx migration  poc test';

CREATE SCHEMA oss_external_table
  AUTHORIZATION xxxpoc;
```

### Prepare CREATE FUNCTION
Since Redshift provides some SQL functions that are currently not supported by HybridDB, you need to replace these
functions or rewrite them:

- `CONVERT_TIMEZONE(a,b,c)` [CONVERT_TIMEZONE](https://docs.amazonaws.cn/en_us/redshift/latest/dg/CONVERT_TIMEZONE.html)

    Replace with the following code:
    ```sql
    timezone(b, timezone(a,c))
    ```
    
- `GETDATE()` [GETDATE](https://docs.aws.amazon.com/redshift/latest/dg/r_GETDATE.html)

    Replace with the following code:
    ```sql
    current_timestamp(0):timestamp
    ```
    
- Replace and optimize user defined functions.

    For example, let's consider the following RedShift function:
    ```sql
    CREATE OR REPLACE FUNCTION public.f_jdate(dt timestamp without time zone)
    RETURNS character varying AS
    '      from datetime import timedelta, datetime
           if dt.hour < 4:
                  d = timedelta(days=-1)
                  dt = dt + d
           return str(dt.date())'
    LANGUAGE plpythonu IMMUTABLE;
    COMMIT;
    ```
    
    We can replace a call to the function above by the following code snippet:
    ```sql
    to_char(a - interval '4 hour', 'yyyy-mm-dd')
    ```

- Other Redshift standard SQL Functions

    If possible, it is better to use the standard SQL function library of PostgreSQL. When it is not possible,
    you need to consider implementing it yourself. Here are a few commonly used functions for your reference:

    [Functions and Operators in PostgreSQL8.2](https://www.postgresql.org/docs/8.2/functions.html)

    - `ISNULL()` https://docs.microsoft.com/en-us/sql/t-sql/functions/isnull-transact-sql?view=sql-server-2017
    - `DATEADD()` https://docs.aws.amazon.com/redshift/latest/dg/r_DATEADD_function.html
    - `DATEDIFF()` https://docs.aws.amazon.com/redshift/latest/dg/r_DATEDIFF_function.html
    - `REGEXP_COUNT()` https://docs.aws.amazon.com/redshift/latest/dg/REGEXP_COUNT.html
    - `LEFT()` https://docs.aws.amazon.com/redshift/latest/dg/r_LEFT.html
    - `RIGHT()` https://docs.aws.amazon.com/redshift/latest/dg/r_LEFT.html

### Prepare CREATE TABLE
1. Change [Redshift Compression Encoding](https://docs.aws.amazon.com/redshift/latest/dg/c_Compression_encodings.html)

    HybridDB for PostgreSQL does not support the following `ENCODE` encoding options from AWS Redshift:
    - BYTEDICT
    - DELTA
    - DELTA32K
    - LZO
    - MOSTLY8
    - MOSTLY16
    - MOSTLY32
    - RAW (no compression)
    - RUNLENGTH
    - TEXT255
    - TEXT32K
    - ZSTD

    `ENCODE XXX` should be removed and replaced with following option in `CREATE TABLE` statement:
    ```sql
    with (COMPRESSTYPE={ZLIB|QUICKLZ|RLE_TYPE|NONE})
    ```
    Complete `CREATE TABLE` definition: https://www.alibabacloud.com/help/doc-detail/59195.htm

2. Change [Distribution Styles](https://docs.aws.amazon.com/redshift/latest/dg/c_choosing_dist_sort.html)

    - `DISTSTYLE EVEN`: Replace with `distributed randomly`
    - `DISTKEY`: Replace with `distributed by (colname1, ...)`
    - `ALL`: Not supported, please remove it

3. Change [Sort keys](https://docs.aws.amazon.com/redshift/latest/dg/t_Sorting_data.html)

    Replace with following options:
    ```sql
    with(APPENDONLY=true,ORIENTATION=column)
    sortkey (volume);
    ```
    Remove `COMPOUND` or `INTERLEAVED` key words in `[ COMPOUND | INTERLEAVED ] SORTKEY (column_name [, ...] ) ]`

4. Example

    The following `CREATE TABLE` statement is from Redshift:
    ```sql
    CREATE TABLE schema1.table1
    (
    	filed1 VARCHAR(100) ENCODE lzo,
    	filed2 INTEGER DISTKEY,
    	filed3 INTEGER,
    	filed4 BIGINT ENCODE lzo,
    	filed5 INTEGER
    )
    INTERLEAVED SORTKEY
    (
    	filed1,
    	filed2
    );
    ```
    
    After conversion nto the HybridDB for PostgreSQL syntax, the `CREATE TABLE` statement is as follows:
    ```sql
    CREATE TABLE schema1.table1
    (
    	filed1 VARCHAR(100) ,
    	filed3 INTEGER,
    	filed5 INTEGER
    )
    WITH(APPENDONLY=true,ORIENTATION=column,COMPRESSTYPE=zlib)
    DISTRIBUTED BY (filed2)
    SORTKEY
    (
    	filed1,
    	filed2
    )
    ```

    Another example of `ENCODE` and `SORTKEY`, the original DDL is as follows:
    ```sql
    CREATE TABLE schema2.table2
    (
    	filed1 VARCHAR(50) ENCODE lzo,
    	filed2 VARCHAR(50) ENCODE lzo,
    	filed3 VARCHAR(20) ENCODE lzo,
    )
    DISTSTYLE EVEN
    INTERLEAVED SORTKEY
    (
    	filed1
    );
    ```

    After the conversion, the `CREATE TABLE` statement is as follows:
    ```sql
    CREATE TABLE schema2.table2
    (
    	filed1 VARCHAR(50),
        filed2 VARCHAR(50),
    	filed3 VARCHAR(20),
    )
    WITH(APPENDONLY=true,ORIENTATION=column,COMPRESSTYPE=zlib)
    DISTRIBUTED randomly
    SORTKEY
    (
    	filed1
    );
    ```

### Prepare CREATE VIEW
Same as `CREATE TABLE`. If there is a SQL statement that needs to be modified, please modify it according to the
standard that conforms to the HybridDB for PostgreSQL syntax.

### Prepare CREATE EXTERNAL TABLE
HybridDB for PostgreSQL supports parallel import from OSS or export to OSS through external tables (which is called
the `gpossext` function).

It can also compress OSS external table files in gzip format to reduce the storage space and cost.
The `gpossext` function can read or write text/csv files, gzipped or not.

You can find detail description from [this document](https://www.alibabacloud.com/help/doc-detail/35457.htm)

### Prepare INSERT INTO
Based on the work above, this step becomes simple and easy to understand. Simply insert the data from the OSS
external table into the normal data table.

The format is `INSERT INTO <TABLE NAME> SELECT * FROM <OSS EXTERNAL TABLE NAME>;`

Example:
```sql
INSERT INTO schema1.table1 SELECT * FROM oss_external_table.table1;
```

### Prepare VACUUM SORT
This step is performed after importing OSS external table data into the HybridDB database. For VACUUM, refer to
[this document](https://www.postgresql.org/docs/8.2/sql-vacuum.html)

## Prepare HybridDB for PostgreSQL instance
### Create and Config HybridDB for PostgreSQL instance
You can find the following topics on the Alibaba Cloud Document Center:
- [Create an instance](https://www.alibabacloud.com/help/doc-detail/50200.htm)
- [Set up a whitelist](https://www.alibabacloud.com/help/doc-detail/50207.htm)
- [Set up an account](https://www.alibabacloud.com/help/doc-detail/50206.htm)

### Create Database Objects
Please follow this document to connect instance 
[Connect to a HybridDB for PostgreSQL database](https://www.alibabacloud.com/help/doc-detail/35428.htm)

In my practice, I choose **psql** and **pgAdmin III 1.6.3**

Then you can perform DDL files completed in the Chapter 6.

1. create schema
2. create functions
3. create tables
4. create views
5. create oss external tables

After completing the creation of the OSS external table, a table of HybridDB for PostgreSQL is associated with a set
of CSV files, and the `SELECT` statement can be used to view the data as follows:
```sql
SELECT * FROM oss_external_table.table1;
```

## Import OSS external table data into the data table
1. Execute `insert.sql` to complete the data import process.

    The csv file saved in the OSS Bucket will be imported into the corresponding table in the HybridDB for PostgreSQL database.

    After importing completed, you can use some familiar `SELECT` statements to verify the data.

2. Execute `vaccum.sql` to make the table ordered again.

## Support
Don't hesitate to [contact us](mailto:projectdelivery@alibabacloud.com) if you have questions or remarks.