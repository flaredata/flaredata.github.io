---
layout: post
title: "TensorFlare or: How I Learned to Stop Worrying and Love ML"
date: 2017-05-10
---

In our [last]({% post_url 2017-04-14-parquet-on-fire %}) post, we discussed running queries in Apache Spark and Flare using the Parquet data format. In the [posts before that](https://flaredata.github.io#blog), we've looked at various speedups which Flare brings over the current iteration of Spark. In all of these posts, we've viewed Flare as a closed system, designed to operate solely as a data loading/processing tool. However, what sort of speedups could we see by chaining a system like Spark or Flare with an existing machine learning framework (in our case, TensorFlow)?

In this blog post, we examine the possibility of using a "big data" processing engine like Flare in a streamed pipeline with TensorFlow to see what sort of speed gains we can achieve.

## TensorFlow

[TensorFlow](https://www.tensorflow.org/) is, to quote their website, "an open source software library for numerical computation using data flow graphs." Essentially, this boils down to being an extremely efficient machine learning (ML) framework. TensorFlow represents computations as graphs, with nodes representing operations and edges between those nodes representing data. All data in TensorFlow is stored in multidimensional arrays (tensors), which allows TensorFlow to be extremely flexible in where it can be deployed and run.

TensorFlow is extremely efficient (concrete numbers available [here](https://www.tensorflow.org/performance/benchmarks) and [here](https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/45166.pdf)), in part due to being written in C++. However, APIs are available in Python, Scala, and Java, with Python being the primary target (as evidenced by the [lack of API stability promises for all other languages](https://www.tensorflow.org/api_docs/)).

## PySpark: The Mullet of Data Processing

In keeping with TensorFlow's target usage, we elected to use Spark's Python API, [PySpark](http://spark.apache.org/docs/2.1.0/api/python/pyspark.html). Due to Python's expressive nature and [documented wide usage](https://www.tiobe.com/tiobe-index/), PySpark is a natural extension to lower the barrier of entry for data science professionals by abstracting things even further than Scala. This results in our proverbial mullet of data processing: a system which allows for users to have nothing but business up front, but a party of performance in the back.

It should be noted that PySpark may contain some inherent latency compared to Spark's native Scala API due to the necessity of using [Py4J](https://www.py4j.org/), but we accept this small performance loss in exchange for using TensorFlow's main API.

## PyFlare: Mullets Without Regret

It's a widely held belief that abstraction must be accompanied by performance loss. However, at the core of Flare is a system aimed specifically at providing "abstraction without regret." That is, users should be given a high-level interface **without** the associated runtime penalty.

To this end, we hooked the PySpark frontend into our Flare backend, replacing the original SparkSQL API. Thus, we achieve the expressiveness gains offered by PySpark, while retaining the performance gains brough by Flare (documented in our previous blog posts). As such, we get our mullet, without the regret! [Disclaimer: We do not advocate getting an actual mullet. While PyFlare may alleviate any regret using PySpark, there are some wounds that even we can't heal.]

## Performance

In order to compare performance between Spark, PySpark, PyFlare, and Flare, we run TPC-H Q6 with scale factor 1. All results described below were obtained using a System76 laptop with an Intel i7-6700HQ and 16GB DDR3 RAM running Ubuntu 16.04 LTS, Spark 2.0, Scala 2.11, and Python 2.7.12.

For brevity, we simply show the results of running these queries with their associated runtimes. We note, however, that Spark and PySpark have almost equivalent runtimes, with Flare and PyFlare both having approximately an 10x speedup.

### Spark

	scala> val q = spark.sql(tpchq6)
    q: org.apache.spark.sql.DataFrame = [revenue: double]

    scala> time(q.show)
    +--------------------+
    |             revenue|
    +--------------------+
    |1.2314107822829978E8|
    +--------------------+

    took: 11355 ms

### PySpark

    >>> df = spark.sql(tpchq6)
    >>> time(df.show)

	+--------------------+
    |             revenue|
    +--------------------+
    |1.2314107822829978E8|
    +--------------------+

    took: 11257 ms

### PyFlare

	>>> df = flare(tpchq6)
	>>> time(df.show)

	+--------------------+                                      
	|             revenue|
	+--------------------+
	|1.2314107822829978E8|
	+--------------------+

	took: 1217 ms

### Flare

	scala> val q1 = flare(q)
    scala> time(q1.show)
    +--------------------+    
    |             revenue|
    +--------------------+
    |2.4609358141849962E8|
    +--------------------+

    took: 1161 ms

## TensorFlare

When looking at how to build an optimized pipeline like TensorFlare, we discovered that TensorFlow does not have a built-in mechanism for sharing tensors across sources. As such, all data querying must be followed by an extremely expensive big data copy operation. TensorFlare, however, avoids these restrictions by sharing the same intermediate representation across sources. This yields huge performance gains, without sacrificing any expressivity.

To show the benefits of having such a system, we built a small experiment utilizing TensorFlow's capabilities. We first trained a model to run on US crime statistics using the method described in [TensorFlow's MNIST tutorial](https://www.tensorflow.org/get_started/mnist/beginners). We then used this model in both PySpark and TensorFlare to see what kind of results we get. In both, we run the following query:

	SELECT
		cluster,
		sum(case when classification = 0 then 1 else 0 end) as class1, 
		sum(case when classification = 1 then 1 else 0 end) as class2, 
		sum(case when classification = 2 then 1 else 0 end) as class3, 
		sum(case when classification = 3 then 1 else 0 end) as class4
	FROM
		(
			SELECT 
				cluster,
				classifier(murder, assault, population, rape) as classification
			FROM
				data
		)
	GROUP BY cluster
	ORDER BY cluster

The classifier method mentioned above is a UDF to Spark -- a blackbox which the optimizer is unable to reason about. In our situation, this classifier is our ML model.

This query allows us to build an accuracy matrix, in which 100% accuracy is represented by every index in the matrix other than the diagonal running from [0, 0] to [N - 1, N - 1] is 0. In our tests, we work on a very small sample size (200 data points) simply for fast testing. As such, we are actually able to obtain 100% accuracy, though this is not expected in larger results.

First, we'll run this in PySpark to get a baseline:

	>>> df = spark.sql(crimeQuery)
	>>> time(df.show)
	+-------+------+------+------+------+
	|cluster|class1|class2|class3|class4|
	+-------+------+------+------+------+
	|      0|    43|     0|     0|     0|
	|      1|     0|    49|     0|     0|
	|      2|     0|     0|    54|     0|
	|      3|     0|     0|     0|    54|
	+-------+------+------+------+------+

	took: 4542 ms

Approximately 4.5 seconds! Not bad...maybe? As we've asked in other posts, how good is this result really?

Running the same query in TensorFlare yields the following:

	scala> val df = flare(crimeQuery)
	scala> time(df.show)
	+-------+------+------+------+------+
	|cluster|class1|class2|class3|class4|
	+-------+------+------+------+------+
	|      0|    43|     0|     0|     0|
	|      1|     0|    49|     0|     0|
	|      2|     0|     0|    54|     0|
	|      3|     0|     0|     0|    54|
	+-------+------+------+------+------+

	took: 1351 ms

1.3 seconds, over a 3x speedup! As mentioned above, this is over a much smaller dataset than one would originally expect. Should we increase our dataset to 2000 data points, we find that PySpark takes almost a minute to run, whereas TensorFlare remains nearly constant!

TensorFlare is still very much in its infancy, so these numbers will be updated as further developments are made.

Interested in fast, easy machine learning at scale? Stay tuned for further posts!