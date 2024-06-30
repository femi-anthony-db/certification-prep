# Exam Notes for Assoc. Certification



## Various Databricks SQL constructs

### COPY INTO
 
**Purpose**

Loads data from a file location into a Delta table.

**Additional Notes**

Retriable and idempotent operation—files in the source location that have already been loaded are skipped.

CREATE TABLE

CREATE TABLE AS SELECT

APPLY CHANGES INTO

GENERATED ALWAYS AS

DROP SCHEMA IF EXISTS schema_name CASCADE

DROP TABLE

### REFRESH TABLE table_name

**Purpose** 
Forces Spark to refresh the availability of external files and any changes.

**Additional Notes**

When spark queries an external table it caches the files associated with it, so that way if the table is queried again it can use the cached files so it does not have to retrieve them again from cloud object storage, but the drawback here is that if new files are available Spark does not know until the Refresh command is ran.


AUTO LOADER

  - Directory Listing
  - File notification

CREATE LIVE TABLE

CREATE STREAMING LIVE TABLE

Note that `IF EXISTS` **ALWAYS precedes the object name** for `CREATE TABLE, CREATE SCHEMA, DROP TABLE, DROP SCHEMA` statements.

Note that `CASCADE` only applies when dropping schemas/databases NOT when dropping tables.

### Unity Catalog Privileges

 - DELETE, UPDATE are NOT a UC privileges
 - Correct privileges: SELECT, MODIFY, CREATE TABLE, EXECUTE

 ### Databricks SQL dashboards

 SQL is used to create dashboards

 ### UNION, UNION ALL operators


UNION and UNION ALL are set operators,

UNION combines the output from both queries but also eliminates the duplicates.

UNION ALL combines the output from both queries.

### Databricks Repos

Databricks Repos supports the following operations:

* Create a new branch
* Switch to a different branch
* Commit and push changes to the remote Git repository
* Pull changes from the remote Git repository
* Merge branches
* Rebase a branch on another branch
* Resolve merge conflicts
* Git reset

It does **NOT** support:

* Delete a branch
* Create and approve pull requests
* Trigger Databricks CICD pipeline


### Auto Loader vs COPY INTO?


#### Auto Loader

Auto Loader incrementally and efficiently processes new data files as they arrive in cloud storage without any additional setup. Auto Loader provides a new Structured Streaming source called cloudFiles. Given an input directory path on the cloud file storage, the cloudFiles source automatically processes new files as they arrive, with the option of also processing existing files in that directory.

When to use Auto Loader instead of the COPY INTO?

You want to load data from a file location that contains files in the order of millions or higher. Auto Loader can discover files more efficiently than the COPY INTO SQL command and can split file processing into multiple batches.

You do not plan to load subsets of previously uploaded files. With Auto Loader, it can be more difficult to reprocess subsets of files. However, you can use the COPY INTO SQL command to reload subsets of files while an Auto Loader stream is simultaneously running.


Auto loader file notification will automatically set up a notification service and queue service that subscribe to file events from the input directory in cloud object storage like Azure blob storage or S3. File notification mode is more performant and scalable for large input directories or a high volume of files.

[AUTO LOADER File Detection Modes](https://docs.databricks.com/en/ingestion/auto-loader/file-detection-modes.html)


COPY INTO only support directory listing NOT file notification modes:

[How does COPY INTO  know when a new file is added](https://chatgpt.com/c/c3913dfd-53a3-4703-9c9c-dca78246993a)


https://docs.databricks.com/en/ingestion/auto-loader/file-detection-modes.html

Here are some additional notes on when to use COPY INTO vs Auto Loader


When to use COPY INTO

https://docs.databricks.com/delta/delta-ingest.html#copy-into-sql-command

When to use Auto Loader

https://docs.databricks.com/delta/delta-ingest.html#auto-loader

###  Structured streaming and end-to-end fault tolerance

How does structured streaming achieves end to end fault tolerance:

First, Structured Streaming uses checkpointing and write-ahead logs to record the offset range of data being processed during each trigger interval.

Next, the streaming sinks are designed to be _idempotent_—that is, multiple writes of the same data (as identified by the offset) do not result in duplicates being written to the sink.

Taken together, replayable data sources and idempotent sinks allow Structured Streaming to ensure end-to-end, exactly-once semantics under any failure condition.

https://spark.apache.org/docs/3.0.0-preview/structured-streaming-programming-guide.html#overview


_Finally, the system ensures end-to-end exactly-once fault-tolerance guarantees through checkpointing and Write-Ahead Logs._


### SQL Endpoint(SQL Warehouse) Overview:

A SQL Warehouse should have at least one cluster

A cluster comprises one driver node and one or many worker nodes

No of worker nodes in a cluster is determined by the size of the cluster (2X -Small ->1 worker, X-Small ->2 workers.... up to 4X-Large -> 128 workers) this is called Scale up

A single cluster irrespective of cluster size(2X-Smal.. to ...4XLarge) can only run 10 queries at any given time if a user submits 20 queries all at once to a warehouse with 3X-Large cluster size and cluster scaling (min 1, max1) while 10 queries will start running the remaining 10 queries wait in a queue for these 10 to finish.

Increasing the Warehouse cluster size can improve the performance of a query, for example, if a query runs for 1 minute in a 2X-Small warehouse size it may run in 30 Seconds if we change the warehouse size to X-Small. this is due to 2X-Small having 1 worker node and X-Small having 2 worker nodes so the query has more tasks and runs faster (note: this is an ideal case example, the scalability of a query performance depends on many factors, it can not always be linear)

A warehouse can have more than one cluster this is called Scale out. If a warehouse is configured with X-Small cluster size with cluster scaling(Min1, Max 2) Databricks spins up an additional cluster if it detects queries are waiting in the queue, If a warehouse is configured to run 2 clusters(Min1, Max 2), and let's say a user submits 20 queries, 10 queriers will start running and holds the remaining in the queue and databricks will automatically start the second cluster and starts redirecting the 10 queries waiting in the queue to the second cluster.

A single query will not span more than one cluster, once a query is submitted to a cluster it will remain in that cluster until the query execution finishes irrespective of how many clusters are available to scale.

If the queries are running sequentially then scale up(Size of the cluster from 2X-Small to 4X-Large) if the queries are running concurrently or with more users then scale out(add more clusters).

SQL endpoint(SQL Warehouse) scales horizontally(scale-out) and vertical (scale-up), you have to understand when to use what.

Scale-out -> to add more clusters for a SQL endpoint, change max number of clusters

If you are trying to improve the throughput, being able to run as many queries as possible then having an additional cluster(s) will improve the performance.


### Difference between Delta Live Tables and Streaming Live Tables 

#### Delta Live Tables
Live tables are **materialized views** for the lakehouse.

A live table is:

 * Defined by a SQL query
 * Created and kept up to date by a pipeline

 

In DLT pipelines, we use the CREATE LIVE TABLE syntax to create a table with SQL. 
To query another live table, prepend the LIVE. keyword to the table name.


	CREATE LIVE TABLE aggregated_sales
	AS SELECT store_id, sum(total)
	FROM LIVE.cleaned_sales
    GROUP BY store_id



Dependencies owned by other producers are just read from the catalog or spark data source
as normal;

   	CREATE LIVE TABLE events
   	AS SELECT ... FROM prod.raw_data

LIVE dependencies, from the **same pipelin** are read from the LIVE schema.

    CREATE LIVE TABLE report
    AS SELECT ... FROM LIVE.events

DLT detects LIVE dependencies and executes all operations in correct order.

DLT handles parallelism and captures the lineage of the data.

#### Streaming Live Tables

A streaming live table is based on Spark Structured Streaming 
and is stateful:

 * Ensures exactly-once processing of input rows
 * Inputs are only read once.

SLTs compute results over append-only streams such as Kafka, Kinesis or Auto Loader.

SLTs allow one to reduce costs and latency by avoiding reprocessing of old data


Easily ingest files from cloud storage as they are uploaded:


     CREATE STREAMING  LIVE TABLE report
     AS SELECT sum(profit)
     FROM cloud_files(prod.sales)
     

* _cloud_files_ keeps track of which files have been read to avoid duplication and wasted work. Uses AUTO LOADER
* Supports both listing and notifications for arbitrary scale
* Configurable schema inference and schema evolution

Can allso read from live streaming sources 

SQL_STREAM function

    CREATE STREAMING LIVE TABLE mystream
    AS SELECT * FROM STREAM(my_table)

 `my_table` must be an **append-only** source

 * `STREAM(my_table)` reads a stream of new records instead of a snapshot
 * Streaming tables must be an append-only table
 * Any append-only delta table can be read as a stream (from live schema, catalog/path)



https://employee-academy.databricks.com/learn/course/1266/play/14544/introduction-to-delta-live-tables

### Differences between Development DLT pipelines and Production DLT pipelines ?

#### Development

 * Resuses a long-running cluster, running for fast iteration
 * No retries on errors enabling faster debugging. Runs once only.

#### Production

 * Cuts costs by turning off clusters as soon as they are done.(uses job cluster)
 * Escalating retries, incl. cluster restarts, ensure reliability in the face of transient issues.



### DLT Pipelinea and Expectations

Expectations are true/false expressions used to validate each row during processing

```CONSTRAINT valid_timestamp
EXPECT (timestamp > '2012-01-01')
ON VIOLATION DROP


@dlt.expect_or_drop(

"valid_timestamp",
col("timestamp") > '2021-01-01')
)

```

DLT offers flexible policies on how to handle records that violate expectations:

* Track number of bad records
* Drop bad records
* Abort processing for a single bad record.

**Retain invalid records**

```
CONSTRAINT valid_timestamp EXPECT (timestamp > '2012-01-01')

```

**Drop invalid records**

```
CONSTRAINT valid_current_page EXPECT (current_page_id IS NOT NULL and current_page_title IS NOT NULL) 
ON VIOLATION DROP ROW
```

**Fail on invalid records**

```
CONSTRAINT valid_count EXPECT (count > 0) ON VIOLATION FAIL UPDATE
```



### Connection to SQL databases


Databricks runtime currently supports connecting to a few flavors of SQL Database including SQL Server, My SQL, SQL Lite and Snowflake using JDBC.

The syntax is below

```
CREATE TABLE <jdbcTable>
USING org.apache.spark.sql.jdbc or JDBC
OPTIONS (
    url = "jdbc:<databaseServerType>://<jdbcHostname>:<jdbcPort>",
    dbtable " = <jdbcDatabase>.atable",
    user = "<jdbcUsername>",
    password = "<jdbcPassword>"
)
```

Note the following:

* The USING clause always ends in .jdbc or .JDBC, the database type is NEVER SPECIFIED here.
* The url option within OPTIONS is where the the database type is specified.

Examples:

```
CREATE TABLE users_jdbc
USING org.apache.spark.sql.jdbc
OPTIONS (
    url = "jdbc:sqlite:/sqmple_db",
    dbtable = "users"
)
```


### Constraints in Databricks

Databricks supports standard SQL constraint management clauses. Constraints fall into two categories:

* Enforced contraints ensure that the quality and integrity of data added to a table is automatically verified.

* Informational primary key and foreign key constraints encode relationships between fields in tables and are **NOT enforced**.

All constraints on Databricks require Delta Lake.

#### Enforced constraints on Databricks

When a constraint is violated, the transaction fails with an error. Two types of constraints are supported:

* `NOT NULL`: indicates that values in specific columns cannot be null.

* `CHECK`: indicates that a specified boolean expression must be true for each input row.


**NOT NULL** usage:

```
ALTER TABLE people10m ALTER COLUMN middleName DROP NOT NULL;
ALTER TABLE people10m ALTER COLUMN ssn SET NOT NULL;
```

**CHECK** usage
```
ALTER TABLE people10m ADD CONSTRAINT dateWithinRange CHECK (birthDate > '1900-01-01');
ALTER TABLE people10m DROP CONSTRAINT dateWithinRange;
```

Note : Databricks does **NOT ENFORCE** primary or foreign key constraints.


### Difference between FLATTEN and EXPLODE in Databricks SQL

https://gemini.google.com/app/9e215f16dbda1ffe


FLATTEN and EXPLODE are both used for manipulating nested data structures in Databricks SQL, but they target different scenarios:

FLATTEN: Used for arrays of arrays. It takes a single array containing multiple inner arrays and transforms it into a single, flat array containing all the elements from the inner arrays.

EXPLODE: Used for arrays or maps. It takes an array (or map) and expands it into separate rows. Each element in the array (or each key-value pair in the map) becomes a new row in the resulting DataFrame.

Here's a table summarizing the key differences:

|Feature|FLATTEN|EXPLODE|
|-------|-------|-------|
|Input Data Type|Array of Arrays|	Array or Map|
|Output Structure|	Single, Flat Array|	Separate Rows for each element|
Use Case|Flatten nested arrays|Expand arrays or maps into rows|


#### Example:

Imagine a DataFrame with a column "colors" that holds arrays of color preferences for each user.

FLATTEN: If "colors" holds arrays like `[["red", "green"], ["blue"]]``, FLATTEN would create a single array with all colors: `["red", "green", "blue"]``.

EXPLODE: EXPLODE would create a separate row for each color preference within each user's array. So, for the example data, you'd get two rows: one for "red" and "green", and another for "blue".
   
   ```
   red
   green
   blue
   ```


#### In short:

Use FLATTEN to combine nested arrays into a single level.

Use EXPLODE to break down arrays or maps into separate rows


https://chatgpt.com/c/c3d7c51e-c20c-4a76-86f0-9e20d4707fd9

__________________________

### Different write modes in Structured Streaming

https://chatgpt.com/c/c3d7c51e-c20c-4a76-86f0-9e20d4707fd9

In Spark Structured Streaming, there are three primary output modes for writing the results of a streaming query:

**Append Mode**: Only new rows that are added to the result table since the last trigger are written to the sink. This mode is suitable when the incoming data is being appended and not updated.

**Complete Mode**: The entire result table is written to the sink every time there is an update. This mode is useful for aggregations where you want to output the complete updated result.

**Update Mode**: Only the rows in the result table that have changed since the last trigger are written to the sink. This mode is useful for operations where only the updated results need to be written, making it more efficient than the complete mode.

These modes help to control the flow of data from the streaming query to the output sink, allowing for flexibility based on the nature of the streaming data and the requirements of the application.



**Append mode (default)** - This is the default mode, where only the new rows added to the Result Table since the last trigger will be outputted to the sink. This is supported for only those queries where rows added to the Result Table is never going to change. Hence, this mode guarantees that each row will be output only once (assuming fault-tolerant sink). For example, queries with only select, where, map, flatMap, filter, join, etc. will support Append mode.

**Complete mode** - The whole Result Table will be outputted to the sink after every trigger. This is supported for aggregation queries.

**Update mode** - (Available since Spark 2.1.1) Only the rows in the Result Table that were updated since the last trigger will be outputted to the sink. 

__________________________

### Auto Loader options


When reading the data `cloudfiles.schemalocation` is used to store the inferred schema of the incoming data.

When writing a stream to recover from failures `checkpointlocation` is used to store the offset of the byte that was most recently processed.


Not that the options are `cloudfiles.schemalocation` and `checkpointlocation`. There is no `cloudfiles` prefix before `checkpointlocation`

**Code Example**:

```
spark.readStream.format("cloudFiles") \
    .option("cloudFiles.format", "json")  # Replace with the format of your data files
    .option("cloudFiles.schemaLocation", schema_location) \
    .load(input_path) \
    .writeStream \
    .format("parquet") \
    .option("checkpointLocation", checkpoint_location) \
    .start(output_path)
```


__________________________


### DLT Pipeline Modes


**Triggered pipelines** update each table with whatever data is currently available and then stop the cluster running the pipeline. Delta Live Tables automatically analyzes the dependencies between your tables and starts by computing those that read from external sources. Tables within the pipeline are updated after their dependent data sources have been updated.

**Continuous pipelines** update tables continuously as input data changes. Once an update is started, it continues to run until manually stopped. Continuous pipelines require an always-running cluster but ensure that downstream consumers have the most up-to-date data.

https://docs.microsoft.com/en-us/azure/databricks/data-engineering/delta-live-tables/delta-live-tables-concepts#--continuous-and-triggered-pipelines


_______________________


### Databricks Widgets

https://docs.databricks.com/en/notebooks/widgets.html

### Run a Databricks notebook from another notebook

https://docs.databricks.com/en/notebooks/notebook-workflows.html



The `%run` command allows you to include another notebook within a notebook. You can use `%run` to modularize your code, for example by putting supporting functions in a separate notebook. You can also use it to concatenate notebooks that implement the steps in an analysis. When you use `%run`, the called notebook is immediately executed and the functions and variables defined in it become available in the calling notebook.

The `dbutils.notebook` API is a complement to `%run` because it lets you pass parameters to and return values from a notebook. This allows you to build complex workflows and pipelines with dependencies. For example, you can get a list of files in a directory and pass the names to another notebook, which is not possible with %run. You can also create if-then-else workflows based on return values or call other notebooks using relative paths.


