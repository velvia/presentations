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
- All the advantages of Cassandra over HDFS
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
    num_shards int,
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

## Shards table

This is for tracking all the subshards.

```sql
CREATE TABLE shards (
    dataset text,
    shard int,
    subshards list<int>,
    PRIMARY KEY (dataset, shard)
);
```

--

## Data Table

```sql
CREATE TABLE data (
    dataset text,
    shard int,
    subshard int,
    version int,
    column_name text,
    row_id int,
    data blob
    PRIMARY KEY ((dataset, shard, subshard, version), column_name, row_id)
);
```

The above places all columns for a subshard for a given version on the same physical row, but data for a column is still grouped together, which is similar to Parquet.  An alternative would be to have each version/column combo on separate physical rows.  Tradeoff probably depends on the exact dataset.

---

## Sharding

* **shard** - Managed by users, allows for parallel / independent writers, or manual sharding like for Geo datasets
* **subshard** - Managed by FiloDB, is for partitioning data within a single shard
    - Although .... maybe it could be used by users too, if they really know what they're doing

---

## Version Control

Writers must provide their own external synchronization mechanism to make sure they are all writing the same version.

---

## Common Operations - DDL

* Change column type - create a new version with the same column name, do Spark job to do type conversion

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
