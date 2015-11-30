# Breakthrough OLAP
# Performance with
# <span class="cassred">Cassandra</span> and Spark

### Evan Chan
### December 2015

---

## Who am I?

<center>
![](home.jpg)
</center>

- Distinguished Engineer, [Tuplejump](http://www.tuplejump.com)
- @evanfchan
- [`http://velvia.github.io`](http://velvia.github.io)
- User and contributor to Spark since 0.9, Cassandra since 0.6
- Co-creator and maintainer of [Spark Job Server](http://github.com/spark-jobserver/spark-jobserver)

---

## About Tuplejump

[Tuplejump](http://tuplejump.com) is a big data technology leader providing solutions and development partnership.

* [FiloDB](http://github.com/tuplejump/FiloDB) - subject of today's talk
* [Calliope](http://tuplejump.github.io/calliope/) - the first Spark-Cassandra integration
* [Stargate](http://tuplejump.github.io/stargate/) - an open source Lucene indexer for Cassandra
* [SnackFS](https://github.com/tuplejump/snackfs) - open source HDFS for Cassandra

---

## Tuplejump as your Development Partner

![](tj2.png)

---

### Big data is yesterday.
# FAST DATA
## is now.

--

<center>
![](Fast-Data-FSI-Whiteboard.png)
</center>

--

## Fast Data + Big Data

- **1-30 seconds**: Reactive processing of streaming data as it comes in to derive instant insights.
- **Minutes to Days/Months**: Combine with recent or historical data for deeper insights, trends, ML.
- Not enough just to have stream processing or batch processing.

--

## Example: Video analytics

- Typical collection and analysis of consumer events
- 3 billion new events every day
- Video publishers want both instant streaming analytics plus reports over months, dashboards
- Pre-aggregation only enables simple dashboard UIs
- What if one wants to offer more advanced analysis, or a generic data query API?
    + Eg, top countries filtered by device type, OS, browser

NOTE: Too many possible combinations to pre-aggregate

--

## Requirements

- Scalable - rules out PostGreSQL, etc.
- Easy to update and ingest new data
    + Not traditional OLAP cubes - that's not what I'm talking about
- Very fast for analytical queries - OLAP not OLTP
- Extremely flexible queries
- Preferably open source

---

## Parquet

- Widely used, lots of support (Spark, Impala, etc.)
- Problem: Parquet is read-optimized, not easy to use for writes
    + Cannot support idempotent writes
    + Optimized for writing very large chunks, not small updates
    + Not suitable for time series, IoT, etc.
    + Often needs multiple passes of jobs for compaction of small files, deduplication, etc.

&nbsp;
<p>
People really want a database-like abstraction, not a file format!

---

## SMACK

- Scala/Spark Streaming
- Mesos
- Akka
- Cassandra
- Kafka

--

## Spark Streaming

Stream processing + SQL + Machine Learning + ETL...

<center>
![](spark-streaming.png)
</center>

"What's New In Spark Streaming" - Tathagada Das, Strata NY 2015

--

## <span class="cassred">Cassandra</span>

<center>
![](cassandra.jpg)
</center>

- Horizontally scalable
- Very flexible data modelling (lists, sets, custom data types)
- Easy to operate
- Perfect for ingestion of real time / machine data
- Best of breed storage technology, huge community
- **BUT: Simple queries only**
- **OLTP-oriented**

--

## Spark provides the missing fast, deep analytics piece of <span class="cassred">Cassandra</span>!

### ...tying together fast event ingestion and rich deep analytics!

---

## How to make Spark and <span class="cassred">Cassandra</span>
## Go Fast

---

## Spark on <span class="cassred">Cassandra</span>: No Caching

--

## Not Very Fast, but Real-Time Updates

- Spark does no caching by default - you will always be reading from C*!
- Pros:
  + No need to fit all data in memory
  + Always get the latest data
- Cons:
  + Pretty slow for ad hoc analytical queries - using regular CQL tables

--

## How to go Faster?

- Read less data
  + COMPACT STORAGE, store your data as Avro/Protobuf/etc.
  + BUT... need lots of extra code, application + Spark side
- Filter using 2ndary indices
  + However, 2i isn't always fast, must scan whole cluster

---

## Spark as Cassandra's Cache

![](2014-07-spark-cass-cache.jpg)

--

## Caching a SQL Table from Cassandra

DataFrames support in Cassandra Connector 1.4.0 (and 1.3.0):

<p>
```scala
val sqlContext = new org.apache.spark.sql.SQLContext(sc)

val df = sqlContext.read
                   .format("org.apache.spark.sql.cassandra")
                   .option("table", "gdelt")
                   .option("keyspace", "test").load()
df.registerTempTable("gdelt")
sqlContext.cacheTable("gdelt")
sqlContext.sql("SELECT count(monthyear) FROM gdelt").show()
```

<p>&nbsp;<p>

NOTE: go over it real fast, 30 sec max, just highlight the cacheTable line

--

## How Spark SQL's Table Caching Works

![](spark-sql-caching-details.jpg)

--

## Spark Cached Tables can be Really Fast

GDELT dataset, 4 million rows, 60 columns, localhost

| Method   |  secs       |
| :------- | ----------: |
| Uncached |   317       |
| Cached   |    0.38     |

<p>&nbsp;<p>
Almost a 1000x speedup!
<p>

On an 8-node EC2 c3.XL cluster, 117 million rows, can run common queries 1-2 seconds against cached dataset.

--

## Problems with Cached Tables

- Still have to read the data from Cassandra first, which is slow
- Amount of RAM: your entire data + extra for conversion to cached table
- Cached tables only live in Spark executors - by default
    + tied to single context - not HA
    + once any executor dies, must re-read data from C*
- Caching takes time: convert from RDD[Row] to compressed columnar format
- Cannot easily combine new RDD[Row] with cached tables (and keep speed)

--

## Problems with Cached Tables

NOTE: Cached tables don't work with 2i indices

NOTE: `rdd.cache()` is NOT the same as SQLContext's `cacheTable`!

---

## Faster Queries Through Columnar Storage

### Wait, I thought Cassandra was columnar?

--

## How Cassandra stores your CQL Tables

Suppose you had this CQL table:

```sql
CREATE TABLE (
  department text,
  empId text,
  first text,
  last text,
  age int,
  PRIMARY KEY (department, empId)
);
```

--

## Cassandra CQL vs Columnar Layout

Cassandra stores CQL tables row-major, each row spans multiple cells:

| PartitionKey | 01:first | 01:last | 01:age | 02:first | 02:last | 02:age |
| :----------- | :------- | :------ | -----: | :------- | :------ | -----: |
| Sales        | Bob      | Jones   | 34     | Susan    | O'Connor | 40    |
| Engineering  | Dilbert  | P       | ?      | Dogbert  | Dog     |  1     |

&nbsp;<p>
Columnar layouts are column-major:

| PartitionKey | first  |  last  |  age  |
| :----------- | ------ | ------ | ----- |
| Sales        | Bob, Susan | Jones, O'Connor | 34, 40 |
| Engineering  | Dilbert, Dogbert | P, Dog    | ?, 1   |

--

## Columnar Format solves I/O

How much data can I query interactively?  More than you think!

<center>
![](columnar_minimize_io.png)
</center>

--

## Columnar Storage Performance Study

<center>
http://github.com/velvia/cassandra-gdelt
</center>
&nbsp;

| Scenario       | Ingest   | Read all columns | Read one column |
| :------------- | -------: | ---------------: | --------------: |
| Narrow table   | 1927 sec | 505 sec          | 504 sec         |
| Wide table     | 3897 sec | 365 sec          | 351 sec         |
| COMPACT STORAGE | 79 sec  |  82 sec          | 82 sec          |
| Columnar       |  93 sec  |   8.6 sec        | 0.23 sec        |

&nbsp;<p>
On reads, using a columnar format is up to **2190x** faster, while ingestion is 20-40x faster.

---

## So, why isn't everybody doing this?

- No columnar storage format designed to work with NoSQL stores
- Efficient conversion to/from columnar format a hard problem
- Most infrastructure is still row oriented
    + Spark SQL/DataFrames based on `RDD[Row]`
    + Spark Catalyst is a row-oriented query parser

NOTE: Simply put, it's a lot of work!

---

> All hard work leads to profit, but mere talk leads to poverty.<br>
> - Proverbs 14:23

--

![](lets_get_this_party_started.gif)

---

## Introducing <span class="golden">FiloDB</span>

<center>
Distributed. Versioned. Columnar. Built for Streaming.
</center>

<p>&nbsp;<p>
<center>
[github.com/tuplejump/FiloDB](http://github.com/tuplejump/FiloDB)
</center>

---

## <span class="golden">FiloDB</span> - What?

--

## Distributed.  Versioned.  Columnar.

* Apache Cassandra as the rock-solid storage engine.  Scale out with no SPOF.
* Separate new rows or columns as versions and query them separately
  - Roll back changes anytime, inexpensively
* Efficient columnar layout = space efficient + fast queries

--

## Spark SQL Queries!

```sql
CREATE TEMPORARY TABLE gdelt
USING filodb.spark
OPTIONS (dataset "gdelt");

SELECT Actor1Name, Actor2Name, AvgTone FROM gdelt ORDER BY AvgTone DESC LIMIT 15;
```

- Read to and write from Spark Dataframes
- Append/merge to FiloDB table from Spark Streaming
- Use Tableau or any other JDBC tool

--

## Multiple ways to Accelerate Queries

* Columnar projection - read fewer columns, saves I/O
* Partition key filtering - read less data
* Sort key / PK filtering - read from subset of keys
  - Possible because FiloDB keeps data sorted
* Versioning - write to multiple versions, read from the one you choose

--

## What's in the name?

<center>
![Filo dough](Filo.jpg)
</center>

Rich sweet layers of distributed, versioned database goodness

--

## 100% Reactive

Built completely on the Typesafe Platform:

- Scala 2.10 and SBT
- Spark (including custom data source)
- Akka Actors for rational scale-out concurrency
- Futures for I/O
- Phantom Cassandra client for reactive, type-safe C* I/O
- Typesafe Config

---

## <span class="golden">FiloDB</span> - Why?

--

## Analytical Query Performance

### Orders of magnitude Faster Queries for Spark on Cassandra 2.x
### Parquet Performance with Cassandra Flexibility
### (with much better performance ceiling)

<center>
(Stick around for the demo)
</center>

NOTE: 200x is just based on columnar storage + projection pushdown - no filtering on sort or partition keys, and no caching done yet.

--

## Economical Big Data Storage

Comparison of GDELT 1979-1984 stored in different formats

| Storage type  | MB  |
|---------------|-----|
| Regular C* CQL Table LZ4    |  1900   |
| Raw CSV file                |  1100   |
| FiloDB + C* LZ4             |   285 |
| C* COMPACT STORAGE LZ4      |   260 |
| FiloDB + C* Deflate         |   170 |
| Parquet + GZIP              |   122 |

FiloDB within 40% of Parquet/HDFS today, with room to improve

--

## <span class="golden">FiloDB</span> = Streaming + Columnar

### Fast Streaming Data + Big Data, All in One!

NOTE: Combining streaming input and columnar/analytical storage is an extremely hard problem, that we are solving!

--

## Designed for Streaming

- New rows appended via Spark Streaming or Kafka
- Writes are *idempotent* - easy **exactly once** ingestion
- Converted to columnar chunks on ingest and stored in C*
- FiloDB keeps your data sorted as it is being ingested

--

## Fast Event/Time-Series Ad-Hoc Analytics

| Entity  | Time1 | Time2 |
| ------- | ----- | ----- |
| US-0123 | d1    | d2    |
| NZ-9495 | d1    | d2    |

&nbsp;<p>
Model your time series with FiloDB similarly to Cassandra:

- **Sort key**: Timestamp, similar to clustering key
- **Partition Key**: Event/machine entity

FiloDB keeps data sorted while stored in efficient columnar storage.

--

## Simplify your Lambda Architecture...

<center>
![Lamba Architecture](lambda-architecture-2-800.jpg)
</center>

(https://www.mapr.com/developercentral/lambda-architecture)

--

## With Spark, Cassandra, and FiloDB

![](simple-architecture.mermaid.png)
<!-- .element: class="mermaid" -->

- Write key-value lookups to Cassandra regularly
- Write raw data / events to FiloDB for ad-hoc analysis / ML
- Far smaller stack to maintain for your analytics

---

## No Cassandra? Keep it All In Memory

![](streaming-in-memory-filodb.mermaid.png)
<!-- .element: class="mermaid" -->

- Unlike RDDs and DataFrames, FiloDB can ingest new data, and still be fast
- Unlike RDDs, FiloDB can filter in multiple ways, no need for entire table scan

---

## Spark Streaming -> FiloDB

```scala
    val ratingsStream = KafkaUtils.createDirectStream[String, String, StringDecoder, StringDecoder](ssc, kafkaParams, topics)
    ratingsStream.foreachRDD {
      (message: RDD[(String, String)], batchTime: Time) => {
        val df = message.map(_._2.split(",")).map(rating => Rating(rating(0).trim.toInt, rating(1).trim.toInt, rating(2).trim.toInt)).
          toDF("fromuserid", "touserid", "rating")
      
        // add the batch time to the DataFrame
        val dfWithBatchTime = df.withColumn("batch_time", org.apache.spark.sql.functions.lit(batchTime.milliseconds))
      
        // save the DataFrame to FiloDB
        dfWithBatchTime.write.format("filodb.spark")
          .option("dataset", "ratings")
          .save()
      }
    }
```
One-line change to write to FiloDB vs Cassandra

---

## Use Case: Smart Cities Transportation

--

## Streaming Data Sources

* City buses - regular telemetry (position, velocity, timestamp)
* Street sweepers - regular telemetry
* Transactions from metro (rail/MRT/subway), buses, smart cards

<center>
![Boston street sweeper](boston-street-sweeper.jpg)
</center>

--

## Streaming Data Sources (II)

* "311" information - reports of potholes, broken light fixtures, things needing repair
* "911" information - new emergencies and incidents

--

## Citizens Want to Know

* Where and for how long can I park my car?
* Are transportation options affected by 311 and 911 events?
* How long will it take the next bus to get here?
* Where is the closest bus to where I am?

--

## Cities Want to Know...

* How can I maximize parking revenue?
  - More granular updates to parking spots that don't need sweeping
* How does traffic affect waiting times in public transit, and revenue?
* How do events affect bus scheduling?

--

## Example: Car Parking vs Street Sweepers

Static dataset: Car Parking spaces/blocks

* Partition key: Geohash
* Sort key: (Parking space geo center, Parking space UUID)

Streaming dataset: Street sweeper telemetry

* Partition key: Geohash of current sweeper location
* Fields: direction, velocity

Streaming dataset: Parking records

* Partition key: Geohash of parking space
* Fields: (Space UUID, Datetime start parking, Datetime end, violations, tow, etc.)

--

## Example Architecture

![](smart-cities.mermaid.png)
<!-- .element: class="mermaid" -->

--

## Example Analyses

Spark Streaming - Alerting Car Owners

* Join newest sweeper telemetry with FiloDB parking spaces and parking records dataset
  - All local join due to same partitioning strategy
  - Send alert to owners N hours before sweeper comes nearby

Spark - Analyzing Parking Revenue

* Scan all recent parking records (verioned by month for example) for a given district, analyze unavailability of parking spaces due to sweepers

---

## FiloDB - How?

--

## FiloDB Architecture

<center>
![](http://velvia.github.io/images/filodb_architecture.png)
</center>

--

## FiloDB Cassandra Schema

```sql
CREATE TABLE filodb.gdelt_chunks (
    partition text,
    version int,
    columnname text,
    segmentid blob,
    chunkid int,
    data blob,
    PRIMARY KEY ((partition, version), columnname, segmentid, chunkid)
) WITH CLUSTERING ORDER BY (columnname ASC, segmentid ASC, chunkid ASC)
```

--

## Ingestion and Storage?

Current version:

* Each dataset is stored using 2 regular Cassandra tables
* Ingestion using Spark (Dataframes or SQL)

Future version?

* Automatic ingestion of your existing C* data using custom secondary index

---

## Towards Extreme Query Performance

--

## The filo project

[http://github.com/velvia/filo](http://github.com/velvia/filo) is a binary data vector library designed for extreme read performance with minimal deserialization costs.

- Designed for NoSQL, not a file format
- random or linear access
- on or off heap
- missing value support
- Scala only, but cross-platform support possible

--

## What is the ceiling?

This Scala loop can read integers from a binary Filo blob at a rate of **2 billion integers** per second - single threaded:

```scala
  def sumAllInts(): Int = {
    var total = 0
    for { i <- 0 until numValues optimized } {
      total += sc(i)
    }
    total
  }
```

--

## Vectorization of Spark Queries

The [Tungsten](https://databricks.com/blog/2015/04/28/project-tungsten-bringing-spark-closer-to-bare-metal.html) project.

Process many elements from the same column at once, keep data in L1/L2 cache.

Coming in Spark 1.4 through 1.6

--

## Hot Column Caching in Tachyon

- Has a "table" feature, originally designed for Shark
- Keep hot columnar chunks in shared off-heap memory for fast access

---

## FiloDB - Roadmap

* Support for many more data types and sort and partition keys - please give us your input!
* Non-Spark ingestion API.  Your input is again needed.
* In-memory caching for significant query speedup
* Projections.  Often-repeated queries can be sped up significantly with projections.
* Use of GPU and SIMD instructions to speed up queries

--

## You can help!

- Try it out!
- Send me your use cases for fast big data analysis on Spark and Cassandra
    + Especially IoT, Event, Time-Series
    + What is your data model?
- Email if you want to contribute

---

## Thanks...

<center>
to the entire OSS community, but in particular:
</center>

- Lee Mighdoll, Nest/Google
- Rohit Rai and Satya B., Tuplejump
- My colleagues at Socrata

<p>&nbsp;</p>
> If you want to go fast, go alone.  If you want to go far, go together.<br>
  -- African proverb

---

# DEMO TIME

### GDELT: Regular C* Tables vs FiloDB

---

# Extra Slides

---

## The scenarios

- [Global Database of Events, Language, and Tone](http://gdeltproject.org) dataset
    + 1979 to now
- 60 columns, 250 million+ rows, 250GB+
- Let's compare Cassandra I/O only, no caching or Spark

1. Narrow table - CQL table with one row per partition key
2. Wide table - wide rows with 10,000 logical rows per partition key
3. Columnar layout - 1000 rows per columnar chunk, wide rows, with dictionary compression

First 4 million rows, localhost, SSD, C* 2.0.9, LZ4 compression.  Compaction performed before read benchmarks.

---

## Columnar Format solves Caching

- Use the same format on disk, in cache, in memory scan
    + Caching works a lot better when the cached object is the same!!
- No data format dissonance means bringing in new bits of data and combining with existing cached data is seamless

---

## Connecting Spark to Cassandra

- Datastax's [Spark Cassandra Connector](https://github.com/datastax/spark-cassandra-connector)
- Tuplejump [Calliope](http://tuplejump.github.io/calliope/)

<p>&nbsp;
<center>
Get started in one line with `spark-shell`!
</center>

```bash
bin/spark-shell \
  --packages com.datastax.spark:spark-cassandra-connector_2.10:1.4.0-M3 \
  --conf spark.cassandra.connection.host=127.0.0.1
```

---

## What about C* Secondary Indexing?

Spark-Cassandra Connector and Calliope can both reduce I/O by using Cassandra secondary indices.  Does this work with caching?

No, not really, because only the filtered rows would be cached.  Subsequent queries against this limited cached table would not give you expected results.

NOTE: the DataFrames support in connector 1.3.0-M1 doesn't seem to support predicate pushdown.

