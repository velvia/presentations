# Efficient Streaming Vector Processing in
# <span class="scalared">Scala</span> at Socrata

### Evan Chan
### Mar 2015

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

Note: We are the open data and open government company. We organize lots of civic hackathons.  We publish government data to make it easy for you to be an informed citizen.   Oh, and our entire backend is in Scala!!
True story, last year I was sitting in your seat, had never heard of Socrata, a speaker came up and talked about how they were changing the world, and I said, I'm in!!

---

## Why Socrata cares about Geo

TODO: Include screenshots of current product, map usage, new UX

---

## The geo-region-coding problem

- What customers want: fast interactive choropleths
- Multitenant
- Fast
- Low latency

--

## Region Datasets

- Comes in as Shapefiles
- Static, new versions = new dataset
- Biggest ones contain 100,000's of polygons
- Many municipalities have their own GIS departments -> custom shapefiles

---

## Why Streaming?

Streaming almost always implies in-memory .... latency.

---

## Why Scala?

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

Note: You don't understand how awesome Scala's collections are until compared with other languages.  Immutable collections, very very rich functionality....

---

## Our Architecture

- Overall diagram: include multiple Geospaces
- Diagram showing Geospace connections during ingress, reading

---

## Memory, Speed and Latency

Sources of Latency:

1. GC Pauses
2. Region cache eviction and reloading

---

## Memory Pressure

- Regularly monitoring memory usage
    + Memory usage in Java/Scala: tricky to measure, % no good?
- (separate slide) Why do we need active eviction?
- Eviction: bad bad bad (latency goes out the window)

---

## More Efficiency?  But How?

- Use bigger machines!
- Sharding
- Less coordinates
    + Simplification
    + Partitioning
- More efficient coordinates
- Off-heap memory

---

## Use bigger machines!

Pros:
- Less memory pressure => better latency
- EASY!

Cons:
- Doesn't really scale
- Doesn't fundamentally solve any issues

---

## Sharding!

- Add a diagram of sharding by region dataset

- Easy to implement, scales (kind of)
- "Hot spot" of large region datasets
    + Huge variation of region dataset sizes - 1000's to millions of coordinates

---

## Partitioning

TODO: insert visual diagram of partitioning shapes/map into regions

---

## Partitioning

- Most of our point datasets are heavily localized
- Helps both loading regions into memory and reduce memory pressure
- Much more fine grained and even distribution/sharding

---

## Efficient Geometries

Insert quote about trading off speed for memory efficiency and latency.

Example: compression
