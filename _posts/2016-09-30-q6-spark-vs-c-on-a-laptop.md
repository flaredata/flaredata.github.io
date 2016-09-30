---
layout: post
title:  "Spark is Fast, but how Fast?"
date: 2016-09-30
---

This blog post is the first in a series to introduce **Flare**, a drop-in accelerator for Apache Spark that delivers (sometimes massive) speedups by compiling queries to native code.

## Query Compilation in Spark 2.0

Performance has been a key focus of the Spark team for quite some time. And truly [impressive progress](https://databricks.com/blog/2015/04/28/project-tungsten-bringing-spark-closer-to-bare-metal.html) has been made in 2015 and 2016. Spark 2.0 sports a query engine that even [generates JVM bytecode](https://databricks.com/blog/2016/05/23/apache-spark-as-a-compiler-joining-a-billion-rows-per-second-on-a-laptop.html) at runtime, modeled after best-of-breed query compilers such as TU Munich's HyPer system. **So what is there really to accelerate?**

In this blog post, we take a closer look at Spark's performance on relational workloads, and we show that there is still room for further acceleration.

## Laptop vs Cluster

Frank McSherry, Michael Isard and Derek G. Murray have eloquently argued in their 2015 [HotOS paper](https://www.usenix.org/system/files/conference/hotos15/hotos15-paper-mcsherry.pdf) and [blog post](http://www.frankmcsherry.org/graph/scalability/cost/2015/01/15/COST.html) that big data systems such as Spark may scale well, but this is often just because there is a lot of internal overhead.

They went on to show that a straightforward native implementation of [PageRank](http://en.wikipedia.org/wiki/PageRank) running on a single laptop can outperform a Spark cluster with 128 cores, using the then-current version. 

Inspired by this setup, we are interested in gauging the inherent overheads of Spark SQL for classic relational workloads.

## The Benchmark

We launch a Spark shell configured to use a single worker thread:

    ./bin/spark-shell --master local[1]


As our benchmark, we pick the simplest query from the industry-standard [TPC-H benchmark](), Query 6:

    scala> val tpchq6 = """
      select
        sum(l_extendedprice*l_discount) as revenue
      from
        lineitem
      where
        l_shipdate >= to_date('1994-01-01')
        and l_shipdate < to_date('1995-01-01')
        and l_discount between 0.05 and 0.07
        and l_quantity < 24
     """

We define the schema of table lineitem, provide the source file, and finally register it as a temporary table for Spark SQL (steps not shown). 

For our experiments, we use scale factor 2 (SF2) of the TPC-H data set, which means that table lineitem is stored in a CSV file of about 1.4 GB.

Following the setup by McSherry et al., we run our tests on a fairly standard laptop: MacBook Pro Retina 2012, 2.6 GHz Intel Core i7, 16 GB 1600 MHz DDR3, 500 GB SSD.

<!--
We define the schema of table `lineitem`:

    import org.apache.spark.sql.types._
    import org.apache.spark.sql.functions._

    val schema_lineitem = StructType(Seq(
      StructField("l_orderkey", IntegerType, nullable = false),
      StructField("l_partkey", IntegerType, nullable = false),
      StructField("l_suppkey", IntegerType, nullable = false),
      StructField("l_linenumber", IntegerType, nullable = false),
      StructField("l_quantity", DoubleType, nullable = false),
      StructField("l_extendedprice", DoubleType, nullable = false),
      StructField("l_discount", DoubleType, nullable = false),
      StructField("l_tax", DoubleType, nullable = false),
      StructField("l_returnflag", StringType, nullable = false),
      StructField("l_linestatus", StringType, nullable = false),
      StructField("l_shipdate", DateType, nullable = false),
      StructField("l_commitdate", DateType, nullable = false),
      StructField("l_receiptdate", DateType, nullable = false),
      StructField("l_shipinstruct", StringType, nullable = false),
      StructField("l_shipmode", StringType, nullable = false),
      StructField("l_comment", StringType, nullable = false)))

And finally load the table:

    val folder = "~/tpch/sf2/"
    val file_lineitem = folder + "lineitem.tbl"

    val lineitem = (spark.read
      .option("delimiter", "|")
      .option("header", "false") // use first line of all files as header
      .option("inferschema", "false") // automatically infer data types
      .schema(schema_lineitem)
      .csv(file_lineitem))

    lineitem.createOrReplaceTempView("lineitem")


We define a small timer function:

    def time[A](f: => A): A = {
      val startTime = System.nanoTime()
        try f finally {
          val endTime = System.nanoTime()
          println("took: " + ((endTime - startTime).toDouble / 1000000) + " ms")
        }
    }

-->


Now let us run our query, Q6, straight from the CSV file as input:

    scala> val q = spark.sql(tpchq6)
    q: org.apache.spark.sql.DataFrame = [revenue: double]

    scala> time(q.show)
    +--------------------+                                                          
    |             revenue|
    +--------------------+
    |2.4609358141849962E8|
    +--------------------+

    took: 24400.555 ms

<!--

    scala> time(q.show)
    +--------------------+                                                          
    |             revenue|
    +--------------------+
    |2.4609358141849962E8|
    +--------------------+

    22616.708

    scala> time(q.show)
    +--------------------+                                                          
    |             revenue|
    +--------------------+
    |2.4609358141849962E8|
    +--------------------+

    21946.58
-->


Clearly this result of 24 seconds is not the best we can do. We could convert our data to Parquet for increased performance, or we can just preload the data so that subsequent runs are purely in-memory. Since we are interested in the computational part and not so much in data loading right now, we opt to preload:

    scala> time(lineitem.persist.count())
    took: 118062.005 ms

    res4: Long = 11997996

We note in passing that preloading is quite slow (almost 2min), which may be due to a variety of factors.

Now we can execute our query in-memory, and we get a much better result:

    scala> time(q.show)
    +--------------------+    
    |             revenue|
    +--------------------+
    |2.4609358141849962E8|
    +--------------------+

    took: 1418.234 ms

Running the query a few more times yields further speedups, but timings stagnate at around 940ms:

    scala> time(q.show)
    +--------------------+    
    |             revenue|
    +--------------------+
    |2.4609358141849962E8|
    +--------------------+

    took: 942.494 ms

The key question now is: how good is this result? 


## Hand-Written C

Since Query 6 is very simple, it is perfectly feasible to write a program in C that does exactly the same thing: map the input file into memory using the mmap system call, load the data into an in-memory columnar representation, and then execute the main query loop which looks like this:

    ...
    for (i = 0; i < size; i++) {
      double l_quantity = l_quantity_col[i];
      double l_extendedprice = l_extendedprice_col[i];
      double l_discount = l_discount_col[i];
      long l_shipdate = l_shipdate_col[i];
      if (l_shipdate >= 19940101L &&
          l_shipdate < 19950101L &&
          l_discount >= 0.05 &&
          l_discount <= 0.07 &&
          l_quantity < 24.0) {
        revenue += l_extendedprice * l_discount;
      }
    }
    ...

If we compile this C program with

    gcc -O3 Q6.c

and run it, it will take 2.8s in total (including data loading), and **just 45ms** for the actual query computation.

So compared to Spark 2.0, the C program runs **20x** faster!


## Flare: Set your Data on Fire!

Is there any good reason *why* the C code needs to be faster than Spark? We believe not, and in fact, running the same Query 6 accelerated with Flare

    scala> val q1 = flare(q)
    scala> time(q1.show)
    +--------------------+    
    |             revenue|
    +--------------------+
    |2.4609358141849962E8|
    +--------------------+

    took: 44.327 ms


yields **exactly the same performance** as the hand-written C code!


Excited? Stay tuned for the next post!




