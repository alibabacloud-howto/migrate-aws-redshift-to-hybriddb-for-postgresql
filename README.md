# Migrating from Amazon Redshift to HybridDB for PostgreSQL
Describe overall process of migration from AWS Redshift to Alibaba Cloud HybridDB for PostgreSQL


## Introduction
In this document, it will describe how to migrate data from AWS Redshift to HybridDB for PostgreSQL.

## Data migration architecture and overall procedure

In general, this migration process includes the following process:
1. Alibaba Cloud related product selection and environment construction, mainly HybridDB for PostgreSQL and OSS;
2. The customer exports the data to be migrated from a AWS Redshift database to S3 at a certain point in time;
3. Configure and start the OSSImport to move the csv data file;
4. Build various objects of the HybridDB for PostgreSQL database, including Schema, Tables, Views and Functions;
5. Import data from the OSS external table into the data tables.

- **Batch import workflow:** Redshift -> S3 -> OSSImport on ECS -> OSS -> HybridDB for PostgreSQL
- **Incremental import workflow:** Application -> S3 -> Lamda -> OSS -> (optional)MNS/MQ -> Function Compute ->HybridDB for PostgreSQL

## Preparation on AWS

### Customer provides access to S3's credentials
Contains the following information
- AccessKeyID and AccessKey Secret.
- Endpoint of S3, such as: **s3.ap-southeast-2.amazonaws.com**
- Bucket name, such as: **alibaba-hybrid-export**

### Exported data format convention
- Export data to a CSV format file
- The size of the export file is up to 50M
- the order of the column values in the file is the same as the column order of the table creation statement
- The number of files is preferably the same as the number of segments in HybridDB for PostgreSQL or a multiple of the number of segments.

### Recommended Redshift UNLOAD command option
After many practices, we offer the Redshift UNLOAD option that is best for HybridDB for PostgreSQL imports. Here is an example:
```
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
```
DELIMITER AS ','
ADDQUOTES
ESCAPE
NULL AS 'NULL'
MAXFILESIZE 50 mb
```

### Get the DDL statement of the object in the Redshift database
Export all DDL statements from AWS Redshift, including but not limited to schema, table, function, view.

## Preparation on Alibaba Cloud

### Prepare Alibaba Cloud RAM sub-account
- RAM account ID
- RAM account Password
- RAM account Access Key ID
- RAM account Access Key Secret

### Prepare OSS Bucket
Create an Alibaba Cloud OSS Bucket in the same area as the AWS S3 Bucket, such as Region Sydney(ap-southeast-2)

After the OSS bucket is created, the Internet Access Endpoint addresses and VPC Network Access from ECS (Internal Network) of the bucket are obtained from OSS Console.


### Prepare OSSImport
- Create ECS in the same area of the bucket, with a network bandwidth of 100Mbps,here we use Windows x64 as the operating system for ECS.
- Download and install the OSSImport tool on the created ECS
The latest version of the OSSImport tool can be found and obtained at https://www.alibabacloud.com/help/doc-detail/56990.htm
- After unzip the OSSImport package, you can see following folders and files (skip)


## Migrating data files from S3 to OSS using OSSImport
### Configuring OSSImport
1. In this practice, we use OSSImport in a stand-alone deployment mode.
2. Edit **conf/local_job.cfg** file,here is an example, only the parameter configuration that must be modified in this practice is provided here.

**Note** The detailed configuration instructions for OSSImport can be found at https://www.alibabacloud.com/help/doc-detail/56990.htm

- srcType=s3
- srcAccessKey="your AWS Access Key ID"
- srcSecretKey="your AWS Access Key Secret"
- srcDomain=s3.ap-southeast-2.amazonaws.com
- srcBucket=alibaba-hybrid-export
- srcBucket=
- destAccessKey="your Alibaba Cloud Access Key ID"
- destSecretKey="your Alibaba Cloud Access Key Secret"
- destDomain=http://oss-ap-southeast-2-internal.aliyuncs.com
- destBucket=alibaba-hybrid-export-1
- destPrefix=
- isSkipExistFile=true

### Starting the OSSImport Migration Task
In the OSSImport stand-alone deployment mode, you can start the migration task by executing **import.bat**.

### Monitoring task status
During the data migration process, you can observe the output of the command execution window. On the other hand, you can observe the occupancy of the network bandwidth through the resource manager.

In this example, since the ECS and the OSS Bucket are deployed in the same region, the network speed between data uploading from the ECS to the OSS Bucket (intranet Endpoint) is not limited. However, since data is downloaded from S3 to OSS via the Internet, the speed of ECS->OSS is virtually the same as the speed of S3->ECS, and the upload speed is limited by the download speed.

### Failed task retry (optional)
Subtasks may fail due to network or other reasons. Failure Retry only retry the failed Task and will not retry the successful Task. Execute **console.bat retry** in cmd.exe under Windows

### Check the files migrated to the OSS Bucket (optional)
Files can be checked through the OSS Console. We also recommend using the **OSS Brower** client tool to view and manipulate files in the bucket.

OSS Brower can be found here: https://www.alibabacloud.com/help/doc-detail/61872.htm

## Data scrubbing with csv files (optional)
**Note** Here is only an example. It does not mean that you must complete this operation. You can perform certain data cleaning operations on the csv file according to your business needs.
- Replace **"NULL"** in the csv files with blank
- Replace **\\,** in the csv files with ,

For this data scrubbing operation, we recommend doing it locally. Therefore, the data file that needs to be scrubbed is first downloaded to the ECS through the OSS Brower tool, and then the data scrubbing operation is performed; then, the data scrubbed files are uploaded to another newly created bucket to be distinguished from the original CSV files. When downloading the original csv files and uploading the cleaned files, we recommend that OSS Brower use OSS's intranet Endpoint for communication, so that intranet traffic charges will be reduced.

## DDL conversion from Redshift to HybridDB for PostgreSQL
In this chapter, we will describe the necessary preparations before creating the HybridDB for PostgreSQL database object, mainly converting the DDL statements in the Redshift syntax format to the HybridDB for PostgreSQL syntax format. At the same time, we will briefly describe the conversion rules.

### Prepare CREATE SCHEMA
This is a sample that conforms to the PostgreSQL syntax format, then you can save it to **create schema.sql**

```
CREATE SCHEMA schema1
  AUTHORIZATION xxxpoc;
GRANT ALL ON SCHEMA schema1 TO xxxpoc;
GRANT ALL ON SCHEMA schema1 TO public;
COMMENT ON SCHEMA model IS 'for xxx migration  poc test';

CREATE SCHEMA oss_external_table
  AUTHORIZATION xxxpoc;
```

### Prepare CREATE FUNCTION
Since Redshift provides some SQL functions, the corresponding functions are not provided in HybridDB for a while, so you need to customize these functions or rewrite them.

- CONVERT_TIMEZONE(a,b,c) [CONVERT_TIMEZONE](https://docs.amazonaws.cn/en_us/redshift/latest/dg/CONVERT_TIMEZONE.html)
Replace with following code:
```
timezone(b, timezone(a,c))
```
- GETDATE() [GETDATE](https://docs.aws.amazon.com/redshift/latest/dg/r_GETDATE.html)
Replace with following code:
```
current_timestamp(0):timestamp
```
- Replace and optimize user defined function
```
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
Replace with folliowing SQL statement, it will improve the performance:
```
to_char(a - interval '4 hour', 'yyyy-mm-dd')
```

- Other Redshift standard SQL Functions

In your practice, please query the standard SQL function library of PostgreSQL. If you do not provide SQL functions compatible with Redshift, you need to consider implementing it yourself, or find best practices from the Internet. Here are a few commonly used functions for your reference. .

[Functions and Operators in PostgreSQL8.2](https://www.postgresql.org/docs/8.2/functions.html)
```
- ISNULL() https://docs.microsoft.com/en-us/sql/t-sql/functions/isnull-transact-sql?view=sql-server-2017
- DATEADD() https://docs.aws.amazon.com/redshift/latest/dg/r_DATEADD_function.html
- DATEDIFF() https://docs.aws.amazon.com/redshift/latest/dg/r_DATEDIFF_function.html
- REGEXP_COUNT() https://docs.aws.amazon.com/redshift/latest/dg/REGEXP_COUNT.html
- LEFT() https://docs.aws.amazon.com/redshift/latest/dg/r_LEFT.html
- RIGHT() https://docs.aws.amazon.com/redshift/latest/dg/r_LEFT.html
```


### Prepare CREATE TABLE
1. Change Compression Encoding [Redshift Compression Encodeing](https://docs.aws.amazon.com/redshift/latest/dg/c_Compression_encodings.html)
HybridDB for PostgreSQL does not support following ENCODE encoding options in AWS Redshift :
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

ENCODE XXX should be removed and replaced with following option in CREATE TABLE statement:
```
with (COMPRESSTYPE={ZLIB|QUICKLZ|RLE_TYPE|NONE})
```
Complete CREATE TABLE definition here:
https://www.alibabacloud.com/help/zh/faq-detail/59195.html



2. Change Distribution Style, Reference: [Distribution Styles](https://docs.aws.amazon.com/redshift/latest/dg/c_choosing_dist_sort.html)
- DISTSTYLE EVEN： Replace with **distributed randomly**
- DISTKEY： Replace with **distributed by (colname1,...)**
- ALL： Not support，removed

3. Change SORT Key, Reference: [Sort keys](https://docs.aws.amazon.com/redshift/latest/dg/t_Sorting_data.html)

Replace with following options:
```
with(APPENDONLY=true,ORIENTATION=column)
sortkey (volume);
```
Remove COMPOUND or INTERLEAVED key words in **\[ COMPOUND | INTERLEAVED \] SORTKEY (column_name \[, ...\] ) \]**

4. Example
Following CREATE TABLE statement is from Redshift：
```
CREATE TABLE schema1.table1
(
	filed1 VARCHAR(100) ENCODE lzo,
	filed2 INTEGER DISTKEY,
	filed3 INTEGER,
	filed4 BIGINT ENCODE lzo,
	filed5 INTEGER,
)
INTERLEAVED SORTKEY
(
	filed1,
	filed2
);

```
After the conversion, the CREATE TABLE statement that conforms to the HybridDB for PostgreSQL syntax is as follows:
```
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

Another example of ENCODE and SORTKEY, the original DDL is as follows:
```
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

After the conversion, the CREATE TABLE statement that conforms to the HybridDB for PostgreSQL syntax is as follows:
```
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
Same as CREATE TABLE. If there is a SQL statement that needs to be modified, please modify it according to the standard that conforms to HybridDB for PostgreSQL syntax.

### Prepare CREATE EXTERNAL TABLE
HybridDB for PostgreSQL supports parallel import from OSS or export to OSS through external tables (which is called the gpossext function).

It can also compress OSS external table files in gzip format to reduce the storage space and the costs.
The gpossext function can read or write text/csv files or text/csv files in gzip format.

You can find detail description from [Parallel import from OSS or export to OSS](https://www.alibabacloud.com/help/doc-detail/35457.htm)

### Prepare INSERT INTO
Based on the above work, this step becomes simple and easy to understand. Simply insert the data from the OSS external table into the normal data table.

The format is **INSERT INTO \<TABLE NAME\> SELECT * FROM \<OSS EXTERNAL TABLE NAME\>; **

Example:
```
INSERT INTO schema1.table1 SELECT * FROM oss_external_table.table1;
```

### Prepare VACUUM SORT
This step is performed after importing OSS external table data into the HybridDB database. For VACCUM, refer to [VACUUM] (https://www.postgresql.org/docs/8.2/sql-vacuum.html)

## Prepare HybridDB for PostgreSQL instance
### Create and Config HybridDB for PostgreSQL instance

From Alibaba Cloud Document Center, you can find detail topics there:
- [Create an instance](https://www.alibabacloud.com/help/doc-detail/50200.htm?spm=a2c63.p38356.b99.10.62a23c622heIqM)
- [Set up a whitelist](https://www.alibabacloud.com/help/doc-detail/50207.htm?spm=a2c63.p38356.b99.13.44cc50b9EMx6kF)
- [Set up an account](https://www.alibabacloud.com/help/doc-detail/50206.htm?spm=a2c63.p38356.b99.14.6cd26ca6znuAoi)

### Create Database Objects

Please follow this document to connect instance [Connect to a HybridDB for PostgreSQL database](https://www.alibabacloud.com/help/doc-detail/35428.htm?spm=a2c63.p38356.b99.16.1f9429ebRNi9K2)

In my practice, I choose **psql** and **pgAdmin III 1.6.3**

Then you can perform DDL files completed in the Chapter 6.

1. create schema
2. create functions
3. create tables
4. create views
5. create oss external tables

After completing the creation of the OSS external table, a table of HybridDB for PostgreSQL is associated with a set of CSV files, and the SELECT statement can be used to view the data as follows:
```
SELECT * FROM oss_external_table.table1;
```

## Import OSS external table data into the data table

1. Execute insert.sql to complete the data import process.

The csv file saved in the OSS Bucket will be imported into the corresponding table in the HybridDB for PostgreSQL database.

After importing completed, you can use some familar SELECT statments to verify the data.

2. Execute vaccum.sql to make the table ordered again.




