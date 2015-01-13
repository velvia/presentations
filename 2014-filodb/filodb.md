# Introduction to <span class="golden">FiloDB</span>

### Evan Chan

---

## What is <span class="golden">FiloDB</span>?

#### A Distributed, versioned, columnar database for tabular datasets
#### Based on Apache Cassandra and Apache Spark

---

## What's in the name?

<center>
![Filo dough](Filo.jpg)
</center>

Rich sweet layers of distributed, versioned database goodness

---

## Distributed

Apache Cassandra.  Scale out with no SPOF.  Cross-datacenter replication.

---

## Versioned

Incrementally add a column or a few rows as a new version.  Easily control what versions to query.  Roll back changes inexpensively.

Generate incremental views on new versions**

---

## Columnar

- Values from the same column are stored together
- Retrieve select columns and minimize I/O for OLAP queries
- Add a new column without having to copy the whole table
- Vectorization and lazy/zero serialization for extreme efficiency

---

## Tight Spark Integration

* Read tables into Spark SQL, process, join, write back out as new tables or versions
* Use 1.2 Spark SQL data source API to selectively read only necessary columns
* Should be easy to cache columns in Tachyon using the Table support feature
* Implement custom `FiloColumnarRelation` that can efficiently scan ByteBuffers read from Cassandra.
    - Like `spark.sql.columnar.InMemoryRelation` but no need to recompress from source!  Should be much faster
    - Would be really interesting to compare with Parquet

---

## Use Cases

- I want really fast Spark SQL queries on Cassandra
- I love Parquet, but HDFS is a pain to operate/setup
- I want more flexibility than HDFS/Parquet (easily add columns, rows)
- I want at-least-once/exactly once storage of streaming data
- I want to version my changes and query on versions.  
    + Or, I want to preview new changes without making them public
- I want an awesome HA and replication story
- I have many tables of various sizes, including ones too big for traditional RDBMS 

---

## What FiloDB is Optimized For

- Bulk appends and updates
- OLAP / Data warehousing queries (full table scans)
- Generating views

---

## What FiloDB is Not Optimized For

- Pinpoint reads and writes
- OLTP

---

## Think of <span class="golden">FiloDB</span> as
## Git + Parquet
## meets Cassandra and Spark

---

## It's like Parquet on Cassandra

- You can actually write to, update, replace data elements easily - works well with at-least-once data pipelines
- Parquet won't let you add, delete, replace columns easily, or give you versioning
- All the advantages of Cassandra over HDFS: simplicity, HA, cross-datacenter replication
- Scales much better for large numbers of versions or datasets
- OTOH, with versioning, achieving data locality is much harder
- For now, no nested structures (possible in future)

---

## Wait, but I thought Cassandra was columnar?

- Cassandra/CQL groups values from each logical row together.  See [this explanation](http://www.slideshare.net/DataStax/understanding-how-cql3-maps-to-cassandras-internal-data-structure).
    + Reading a subset of columns still incurs high I/O cost
- FiloDB stores values from the same column together on disk, minimizing I/O for OLAP queries
- Initial studies show columnar layout is much more compact and 10-100x more efficient on reads
- FiloDB is designed for a virtually unlimited number of tables.  Cassandra will OOM with lots of CFs.

---

## The New Scalable Data Platform

1.  Store your data in a scalable data store (Cassandra)
2.  Use a flexible distributed computation layer (Spark)
3.  Visualize and profit!

---

## FiloDB Concepts

- **Dataset**: a table with a schema
- **Version**: each "diff" or incremental set of changes (appends / updates / deletes / new column)
- **Shard**: contains one set of rows for all the columns in a version.  Divides and distributes the dataset.

---

## Assumptions and Tradeoffs

- Only one writer per shard.  Central shard assignment.
    + Keep local state per writer, such as row-id
    + Coordination required for error recovery, writer shard assignment
- Each version is the unit of atomic change.  Changes within a version may not be atomic.
- Versions can be used for change isolation and propagation
    + No need for explicit transaction log.  Just read out data from a version

---

## Kafka Inspired

- Explicitly control sharding for parallelism
- Each shard reader controls their own pointer.  Server is simple and relatively stateless
- Use of zero-copy, zero-deserialisation techniques for fast queries

TODO: insert a Mermaid diagram here.

---

## Cassandra Schema

This will probably change a lot.

--

## Datasets Table

```sql
CREATE TABLE datasets (
    name text,
    properties map<text, text>,
    PRIMARY KEY (name)
);
```

Dataset name must be globally unique.

--

## Columns Table

```sql
CREATE TABLE columns (
    dataset text,
    version int,
    column_name text,
    type text,
    deleted boolean,
    batch_size int,
    properties map<text, text>,
    PRIMARY KEY (dataset_name, version, column_name)
);
```

Combination of version and column_name uniquely identifies a column.

--

## Shards table

This is for tracking all the shards for a given dataset.

```sql
CREATE TABLE shards (
    dataset text,
    num_shards int STATIC,
    shard int,
    versions list<int>,
    PRIMARY KEY (dataset, shard)
);
```

To find out latest shard number, just select shard order desc limit 1;

--

## Data Table

```sql
CREATE TABLE data (
    dataset text,
    version int,
    shard int,
    column_name text,
    row_id int,
    data blob
    PRIMARY KEY ((dataset, version, shard), column_name, row_id)
);
```

- All the columns for a given shard are colocated together
- Works well for mostly-append data with few updates; data with many columns
- Works poorly when there are tons of updates on the same rows

--

## Data Table - Alternative Scheme

```sql
    PRIMARY KEY ((dataset, column_name, shard), row_id, version)
);
```

- All the versions are colocated, making merging trivial
- Works well for frequently updated data with not too many columns read at once
- Would work poorly for data with many columns;

---

## Example data: mostly appends

| Version | Shard |    |    |     |     |
| :------ | :---- | -- | -- | --- | --- |
|  1      |  1    | Col1_R1 | Col1_R2 | Col2_R1 | Col2_R2 |
|  1      |  2    | Col1_R1 | Col1_R2 | Col2_R1 | Col2_R2 |
|  2      |  2    | Col1_R3 | Col1_R4 | Col2_R3 | Col2_R4 |
|  3      |  3    | Col1_R1 | Col1_R2 | Col2_R1 | Col2_R2 |
<!-- .element: class="fullwidth" -->

Once the versions and shards are known, reading becomes pretty easy.

---

## Example Data: mostly updates

| Version | Shard |    |    |     |     |
| :------ | :---- | -- | -- | --- | --- |
|  1      |  1    | Col1_R1 | Col1_R2 | Col2_R1 | Col2_R2 |
|  2      |  1    |         | Col1_R2' |    |  |
|  3      |  1    |         | Col1_R2'' |    |  |
|  4      |  1    |         | Col1_R2''' |    |  |

Reading Col1_R1 is easy.
Reading Col1_R2 is not.  We have to read all four versions to collapse into the latest version.

---

## Versioning

- Query version n:  really accumulate all the state from version 0 to version n
- What if you could select a range of versions (n1, n2) to query?
    + Might have incomplete data.  It might contain only newly added columns for example, or newly added rows.
    + Over time, a range of old versions might get compacted, and rewritten.

---

## Version Control

Writers must provide their own external synchronization mechanism to make sure they are all writing the same version.

---

## Versioning and Cassandra

* In the future, we could potentially use stored procedures (Cassandra 3.0) for
    - efficient reading of relevant versions (server side skipping of data)
    - help with compaction of versions
* HBase would be easier actually due to the built in versioning API

---

## Layered Architecture

Have a small core and layer functionality on top.  Keep the core extremely simple and reliable.

| Layer         |    Description    |
| :------------ | :---------------- |
| Core          | Basic I/O.  Version merging logic. No coordination of shards or versions.  No parsing of data. |
| Bulk Ingester | Coordinates parallel/distributed ingest of bulk and multiple datasets.  Coordinates sharding. |
| Vectorizer    | Groups multiple row values together for efficient I/O  |
| Spark         | All queries beyond basic data export; dataset joins    |
| Application   | Decides on versions, reads/writes data. |
<!-- .element: class="fullwidth" -->

---

## Sharding

Because one Cassandra physical row might not be enough for larger datasets.

* Single writer - increment shard # when one fills up one physical row
* Multiple writers - have a singleton (Akka cluster?) provision shards.  New one whenever any writer fills up one row.  Shards from multiple writers will be interleaved.
* Custom - for example, a geo Z-curve based sharding.  The shard numbers might not be incremental at all, but based on the higher order bits of the z-curve.

---

## Primary keys

- Needed in some cases to look up a row by a user-defined key
- Recommend: optimize for bulk ingest/read, store primary key as just another column
- Add an inverted index mapping primary key to `(version, shard, row_id)`
    + Consider using Lucene integrations like StratioBD, Stargate (would need some customizations to work with this schema)

---

## Deep Dive - Ingestion

--

## Ingestion Goals

1. Don't lose any data!  Idempotent at-least-once
2. Backpressure.  Ingest only when ready.
3. Support distributed bulk ingest
4. Efficient ingest and efficient retries
5. Should work for streaming data, including error recovery

--

## Ingestion API

- User divides dataset into independent parts or streams, sequences input rows
- FiloDB should take care of details like sharding, row IDs
- Regular acks of incoming stream

--

## Typical Ingest Message Flow

TODO: add websequencediagram of typical ingest workflow

--

## Don't Lose Any Data

- Increasing sequence numbers Kafka-style for each row/chunk of ingress stream
- Ack latest committed sequence number
    + Works for any data layout
    + Scalable, works when grouping rows into columnar layout
- Ingester logs state changes with every chunk
- On error:
    + Rewind to last committed sequence number (may rely on client for replay)
    + Ingester uses logged events to reconstruct prev state

---

## Detailed Use Cases / WorkFlow

--

## Initial Ingress

- (Core API) create a new dataset
- (Core API) define columns for initial version
- (Bulk Ingester) start ingesting rows of data
    + New shard ID assignment
    + Translate rows of data into individual columns of data
    + Use lower level Core API to write out columns
    + Know when a new shard is needed.  Initial version - fixed # of rows per shard?

--

## Append new rows

- (Bulk Ingester) restore state of current shards / look up latest shard ID
- (Core API) create a new version if desired to write new rows into
- (Bulk Ingester) ingest more rows of data into existing or new shard

Hm, what about state of how many rows have been written to current shard?

--

## Replace a Row

--

## Compute a new column from existing column

- Find all the shards for the desired version range
- For each shard, merge all versions together, read into Spark
- Compute new column
- Bulk ingester write new column in

--

## Export all rows

- There is no concept of an offset.
- Paging can be done using a token which translates to a `(version, shard, row_id)` or similar.

--

## Deleting a column

--

## DDL Operations?

* Change column type - This breaks down as follows:
    - (Core API) Create new version with same column name but different type
    - (Spark) Use a Spark job to read from v1/columnA, do type conversion, and write v2/columnA
