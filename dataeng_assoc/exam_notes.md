# Exam Notes for Assoc. Certification



### Various Databricks SQL constructs

COPY INTO

CREATE TABLE

CREATE TABLE AS SELECT

APPLY CHANGES INTO

GENERATED ALWAYS AS

DROP TABLE

AUTO LOADER

  - Directory Listing
  - File notification

CREATE LIVE TABLE

CREATE STREAMING LIVE TABLE


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
