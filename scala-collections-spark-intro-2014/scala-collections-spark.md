# From <span class="scalared">Scala</span> Collections
# to Fast Big Data with
# <span class="georgia-peach">Apache Spark</span>

### Evan Chan
### Nov 2014

---

## Who am I?

- Principal Engineer, [Socrata, Inc.](http://www.socrata.com)
- @evanfchan
- [`http://github.com/velvia`](http://github.com/velvia)
- Creator of [Spark Job Server](http://github.com/spark-jobserver/spark-jobserver)

---

## ![](socrata-white-medium.png)

<center>
<h3>We build <span style="color: #4e82ff">software</span> to make <span style="color: #ff5887">data</span> useful to <span style="color: #f7b63d">more people</span>.</h3>
</center>

[data.edmonton.ca](http://data.edmonton.ca) [finances.worldbank.org](http://finances.worldbank.org) [data.cityofchicago.org](http://data.cityofchicago.org) [data.seattle.gov](http://data.seattle.gov) [data.oregon.gov](http://data.oregon.gov) [data.wa.gov](http://data.wa.gov) [www.metrochicagodata.org](http://www.metrochicagodata.org) [data.cityofboston.gov](http://data.cityofboston.gov) [info.samhsa.gov](http://info.samhsa.gov) [explore.data.gov](http://explore.data.gov) [data.cms.gov](http://data.cms.gov) [data.ok.gov](http://data.ok.gov) [data.nola.gov](http://data.nola.gov) [data.illinois.gov](http://data.illinois.gov) [data.colorado.gov](http://data.colorado.gov) [data.austintexas.gov](http://data.austintexas.gov) [data.undp.org](http://data.undp.org) [www.opendatanyc.com](http://www.opendatanyc.com) [data.mo.gov](http://data.mo.gov) [data.nfpa.org](http://data.nfpa.org) [data.raleighnc.gov](http://data.raleighnc.gov) [dati.lombardia.it](http://dati.lombardia.it) [data.montgomerycountymd.gov](http://data.montgomerycountymd.gov) [data.cityofnewyork.us](http://data.cityofnewyork.us) [data.acgov.org](http://data.acgov.org) [data.baltimorecity.gov](http://data.baltimorecity.gov) [data.energystar.gov](http://data.energystar.gov) [data.somervillema.gov](http://data.somervillema.gov) [data.maryland.gov](http://data.maryland.gov) [data.taxpayer.net](http://data.taxpayer.net) [bronx.lehman.cuny.edu](http://bronx.lehman.cuny.edu) [data.hawaii.gov](http://data.hawaii.gov) [data.sfgov.org](http://data.sfgov.org) [data.cityofmadison.com](http://data.cityofmadison.com) [healthmeasures.aspe.hhs.gov](http://healthmeasures.aspe.hhs.gov) [data.weatherfordtx.gov](http://data.weatherfordtx.gov) [www.data.act.gov.au](http://www.data.act.gov.au) [data.wellingtonfl.gov](http://data.wellingtonfl.gov) [data.honolulu.gov](http://data.honolulu.gov) [data.kcmo.org](http://data.kcmo.org) [data2020.abtassociates.com](http://data2020.abtassociates.com)

Note: We lead the open data and open government movements and organize lots of civic hackathons.  We publish government data to make it easy for you to be an informed citizen.   Oh, and our entire backend is in Scala!!

---

<center>
![](scala.jpg)
</center>

<p>
<center>
Why do we love Scala so much?
</center>

&nbsp;
<p>
<center>
Concurrency?   <!-- .element: class="fragment roll-in" -->

Java interop?  <!-- .element: class="fragment roll-in" -->

FP?            <!-- .element: class="fragment roll-in" -->

**Collections...**  <!-- .element: class="fragment roll-in" -->
</center>

Note: I would say functional collections is one of those reasons.  You don't understand how awesome Scala's collections are until compared with other languages.  Immutable collections, very very rich functionality....

---

## Why Functional Collections are So Awesome

- Makes working with data a joy
- Easily go from sequential, to parallel, to distributed Hadoop/Spark computations
- Many other monads are based on collections as well!
    + `Option`
    + `Try`

Note: I know for me personally, I strongly prefer languages that have functional collections.  Can't flatmap it, won't program it.  That's the biggest thing missing from Go! and what makes it not really functional in my book.

---

## map

```scala
scala> List(1, 3, 5, 6, 7).map(_ * 2)
res0: List[Int] = List(2, 6, 10, 12, 14)
```

---

## What is really going on when you do `myList.map`?

```scala
  def map[B, That](f: A => B)(implicit bf: CanBuildFrom[Repr, B, That]): That = {
    def builder = { // extracted to keep method size under 35 bytes, so that it can be JIT-inlined
      val b = bf(repr)
      b.sizeHint(this)
      b
    }
    val b = builder
    for (x <- this) b += f(x)
    b.result
  }
```

- A new collection is created using `CanBuildFrom`
- Elements are added one at a time by evaluating the mapping function

---

## Why does it have to be sequential?

- Well, it doesn't.

```scala
scala> List(1, 3, 5, 6, 7).par.map(_ * 2)
res1: scala.collection.parallel.immutable.ParSeq[Int] = ParVector(2, 6, 10, 12, 14)
```

> Parallel operations are implemented with divide and conquer style algorithms that parallelize well. The basic idea is to split the collection into smaller parts until they are small enough to be operated on sequentially.

---

<center>
![](ants-labor-division.jpg) 
</center>

---

## What else can be easily parallelized?

- `filter`
- `foreach`
- Harder: `groupBy`

---

## Intro to Apache ![](spark.jpg)

- Horizontally scalable, in-memory queries
- `map`, `filter`, `groupBy`, `sort` etc.
- SQL, machine learning, streaming, graph, R
- Huge community and momentum
- REPL for easy interactive data analysis

---

## Doing a distributed map with Spark

```scala
  myIntRdd.map(_ * 2).take(100)
```

What is really going on under the hood?

---

## RDD - Resilient Distributed Dataset

- Core data abstraction in Spark
- Distributed collection of items
- Each partition must fit entirely on one node

![](spark-rdd-concept.png)

---

## RDD Transformations

<center>
    ![](2014-11-Spark-RDD-XForm.png)
</center>

---

## RDD Actions

---

## Eager vs lazy collections

---

## How Laziness is achieved

---

## Spark and Laziness

---

## What's Going on?

---

## Why Laziness is Important for Big Data

---

## More fun - grouping and sorting

---

## TopK in Spark - method 1

---

## TopK in Spark - method 2

---

## ETL, single threaded

---

## Parallel ETL

---

## Parallel ETL in Spark

---

## Caching Data in Spark

---

## Spark, Lineage, and Laziness

What if I lose some data in memory?
