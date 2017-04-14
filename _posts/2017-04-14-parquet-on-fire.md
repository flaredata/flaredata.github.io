---
layout: post
title: "Parquet on Fire"
date: 2017-04-14
---

As we discussed in our [first]({% post_url 2016-09-30-q6-spark-vs-c-on-a-laptop %}) post, running the TPC-H benchmark from disk yields a suboptimal result. To combat this, we gave two options for speeding up this query: converting our data to Parquet, or preloading the data. In that post, we chose the latter option at the time and **[spoiler alert]** found great speedups doing so. Can we see similar speedups using Parquet?

In this blog post, we take a closer look at the Parquet data format and its performance on relational workloads, and we show that, again, there is still room for further acceleration.

## Apache Parquet

[Apache Parquet](https://parquet.apache.org/) is, to quote their website, a "columnar storage format available to any project in the Hadoop ecosystem." This format boasts both [significant speedups](https://blog.cloudera.com/blog/2016/12/achieving-a-300-speedup-in-etl-with-spark/) and [significant compression gains](https://blog.cloudera.com/blog/2016/04/benchmarking-apache-parquet-the-allstate-experience/) compared with a traditional CSV file. For our purposes, we focus mainly on these speedups, and accept the additional compression gains as a cherry on top.

Much of Parquet's speedup over CSV in regards to relational workloads come simply from the columnar representation; only necessary columns need to be handled, removing extraneous data manipulation. So, let's examine this representation in more detail!

Parquet files are encoded using a concept of row groups, columns, and pages; a file contains N row groups, each of which has M columns (where each column corresponds to a column in the overall schema). For clarity, we'll call these M columns, "column chunks." Each column chunk has a corresponding page header containing metadata ("page header metadata") regarding the values to follow. These values, then, are stored in pages -- large chunks of pure data.

It's worth noting at this point that Parquet uses the striping and assembly algorithms proposed by Google in [their Dremel paper,](https://research.google.com/pubs/pub36632.html), but we'll skip these details for simplicity.

At the end of each Parquet file is a block of metadata which includes the file's schema, the total number of rows, and the locations within the file where each column chunk can be found. Spark uses this metadata to construct a set of column iterators, providing the aforementioned direct access to individual columns.

Now, the big question...is it fast?

<div>
<img width="60%" src="https://media.tenor.co/images/a8441f0f55fa49d137894f93e0371bdc/tenor.gif">
</div>



## Parquet Performance
We'll consider two cases:

1. Reading an entire file of data (worst-case scenario for Parquet)
2. Reading a subset of columns


All experiments were run on a single NUMA machine with 4 sockets, 12 Xeon E5-4657L cores per socket, and 256GB RAM per socket (1 TB total). We used Ubuntu 14.04.1 LTS, and Spark 2.0, with Scala 2.11. We will focus on the standard TPC-H benchmark, with all results in milliseconds.

**Reading a File**
It is expected that using Parquet will result in no significant gains over a traditional CSV reader. However, we find that even in this scenario, **Spark performs better when loading data from Parquet than from CSV.** We believe that this is due to effort on Apache's park to optimize Parquet reading in Spark, as well as design efforts from the Parquet team with Spark-like systems in mind.

Let's look at loading every table in TPC-H on scale factor 10.

<style type="text/css">
    table {
        background: white;
        color: black;
        padding: 15px;
    }
    th {
        border-bottom: 1px solid black;
        padding-right: 5px;
        text-align: right;
    }
    td {
        padding-right: 5px;
        text-align: right;
    }
</style>

<p></p>

<p></p>

|   Table   | # Tuples   | Spark (CSV)   | Spark(Parquet)   |
|:---------|:-------:|:-------:|:-------:|
| CUSTOMER | 1500000  | 11664  | 9730  |
| LINEITEM | 59986052 | 471207  | 257898  |
| NATION | 25 | 106  | 110  |
| ORDERS | 15000000 | 85985  | 54124  |
| PART | 2000000 | 11154  | 7601  |
| PARTSUPP | 8000000 | 28748  | 17731  |
| REGION | 5 | 102  | 90  |
| SUPPLIER | 100000 | 616  | 522  |

<p></p>

As shown here, we see that only one table is loaded faster in CSV: NATION. It appears that this is due to the inherent overhead built into Parquet due to the column chunk/paging structure -- NATION, being only 25 lines long, is too small for Parquet to "catch back up."

**Reading a Subset**
This is where we really expect Parquet to shine. And, indeed, we are not disappointed. Let's take a look at the results of TPC-H query 6 (on scale factor 1, this time), in which we look at 4 columns of table LINEITEM (which has 16 columns). First, we'll look at CSV:

    scala> val q = spark.sql(tpchq6)
    q: org.apache.spark.sql.DataFrame = [revenue: double]

    scala> time(q.show)
    +--------------------+                                                          
    |             revenue|
    +--------------------+
    |2.4609358141849962E8|
    +--------------------+

    took: 12278 ms

And now, Parquet:

    scala> val q = spark.sql(tpchq6)
    q: org.apache.spark.sql.DataFrame = [revenue: double]

    scala> time(q.show)
    +--------------------+                                                          
    |             revenue|
    +--------------------+
    |2.4609358141849962E8|
    +--------------------+

    took: 535 ms

Almost a **23x** speedup! However, can we do better?

## Flare CSV
As mentioned in our first post, using Flare's CSV reader yields far better results than Spark's default implementation. So, how does Flare's CSV reader stack up to Spark's Parquet reader?

    scala> val q1 = flare(q)
    scala> time(q1.show)
    +--------------------+    
    |             revenue|
    +--------------------+
    |2.4609358141849962E8|
    +--------------------+

    took: 568 ms

While this is still an impressive **22x** speedup over Spark's default CSV reader, it falls just shy of Spark's Parquet reader. With this in mind, how does Flare's Parquet reader stack up?

## Flare Parquet
By applying the same techniques to Flare's Parquet reader as we did with Flare's CSV reader, do we get the same speedups? Let's look at our results from query 6:

    scala> val q1 = flare(q)
    scala> time(q1.show)
    +--------------------+    
    |             revenue|
    +--------------------+
    |2.4609358141849962E8|
    +--------------------+

    took: 99 ms

This translates to a **124x** speedup over Spark's CSV reader, an additional **5x** speedup over Spark's Parquet reader! In fact, across all TPC-H queries (for scale factor 1), we see speedups of up to **720x** over Spark's CSV readers, with the minimum being a still sizeable **35x** speedup.

<div>
<img src="{{ site.baseurl }}/img/tpch-streaming-speedup-sf1.jpg" width="100%" />
</div>

For more concrete numbers, check out the recent published Flare paper [here](https://arxiv.org/abs/1703.08219)!

How does Flare Parquet stack up against other column store formats and other database engines? Check back soon to find out!
