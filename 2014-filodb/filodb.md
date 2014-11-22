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

## Use Cases

- I have many tables of various sizes, including ones too big for traditional RDBMS 
- I want easy sharding, replication, and HA, including cross-datacenter replication
- I want to version my changes and query on versions.  
    + Or, I want to preview new changes without making them public
- I want an efficient schema designed for OLAP queries
- I want to easily add columns to my data
- I want something designed to work with at-least-once streaming systems

---

## Distributed

Apache Cassandra.

---

## Versioned

- User controls what version to write to (or just say latest)
- queries will merge different layers / versions

---

## Columnar

- Columns are stored together
- Retrieve select columns and minimize I/O for OLAP queries
- Add a new column without having to copy the whole table

---

## It's like Parquet on Cassandra

- You can actually write to, update, replace data elements easily - works well with at-least-once data pipelines
- Parquet won't let you add, delete, replace columns easily, or give you versioning
- All the advantages of Cassandra over HDFS: simplicity, HA, cross-datacenter replication
- Scales much better for large numbers of versions or datasets
- OTOH, with versioning, achieving data locality is much harder

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
    batch_size text,
    properties map<text, text>,
    PRIMARY KEY (dataset_name, version, column_name)
);
```

Combination of version and column_name uniquely identifies a column.

--

## Versions table

Easily see all the relevant columns for a given version.

--

## Shards table

This is for tracking all the shards for a given version.

```sql
CREATE TABLE shards (
    dataset text,
    version int,
    shard int,
    column_count int,
    row_count int
    PRIMARY KEY ((dataset, version), shard)
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

The above places all columns for a subshard for a given version on the same physical row, but data for a column is still grouped together, which is similar to Parquet.  An alternative would be to have each version/column combo on separate physical rows.  Tradeoff probably depends on the exact dataset.

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

## Compute a new column from existing column

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
