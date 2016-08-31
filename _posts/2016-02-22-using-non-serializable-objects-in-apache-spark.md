---
title:  "Using Non-Serializable Objects in Apache Spark"
modified: 2016-08-31 12:00:00 +0200
last_modified_at: 2016-08-31 12:00:00 +0200
tags: [Apache Spark, Scala]
categories: [Dev]
header:
    image: post-logo-spark.png
    teaser: post-logo-spark.png
---
Anyone who starts writing applications for Apache Spark encounters immediately an infamous exception...

```java
org.apache.spark.SparkException: Task not serializable
	at org.apache.spark.util.ClosureCleaner$.ensureSerializable(ClosureCleaner.scala:304)
	at org.apache.spark.util.ClosureCleaner$.org$apache$spark$util$ClosureCleaner$$clean(ClosureCleaner.scala:294)
	at org.apache.spark.util.ClosureCleaner$.clean(ClosureCleaner.scala:122)
	at org.apache.spark.SparkContext.clean(SparkContext.scala:2055)
	at org.apache.spark.rdd.RDD$$anonfun$map$1.apply(RDD.scala:324)
	at org.apache.spark.rdd.RDD$$anonfun$map$1.apply(RDD.scala:323)
	at org.apache.spark.rdd.RDDOperationScope$.withScope(RDDOperationScope.scala:150)
	at org.apache.spark.rdd.RDDOperationScope$.withScope(RDDOperationScope.scala:111)
	at org.apache.spark.rdd.RDD.withScope(RDD.scala:316)
	at org.apache.spark.rdd.RDD.map(RDD.scala:323)
```

This happens whenever Spark tries to transmit the scheduled tasks to remote machines. Tasks are just pieces of application code that are sent from the driver to the workers.

Given the frequency of that exception, **one may think that any piece of code that is executed by a worker node must be serializable. Fortunately, that is not true**.

The classpath of the driver and worker nodes are controlled by the user that is launching the application. This is the case when the application is launched in a standalone Spark cluster as well when an external cluster manager, like YARN, is used. If the deployment solution is an uber-jar, the jar will be shared among all machines. When the application is composed of multiple separate jars, the –jars option (supported in YARN mode) can be used to include them all in the classpaths of the worker JVMs.

In this context, it is quite easy to use non-serializable objects in tasks. 
The only requirement is that they have **just a serializable initialization code**.

Let's see an example. Suppose you want to connect from the remote worker machines to a JDBC data source (hope that connections are issued towards a NoSQL – a.k.a. NowSequel – database). Connection factories are often not serializable.

```scala
// A pooled: org.apache.commons.dbcp2.BasicDataSource
val ds = new BasicDataSource() // Not serializable
ds.setDriverClassName("org.driver.Classname")
ds.setUrl("...")
// set username, options
 
sc.parallelize(1 to 10)
  .map(i => {
    val conn = ds.getConnection
    // do something
    conn.close()
  })
  .count
 
// does it work?
```

Of course, **it will not work**.

```java
org.apache.spark.SparkException: Task not serializable
...
...
Caused by: java.io.NotSerializableException: org.apache.commons.dbcp2.BasicDataSource
```

It throws the infamous “Task not serializable” exception. But you can just wrap it in an object to make it available at the worker side.

```scala
// Holds a reference to the datasource factory
object Holder {
 
  def ds() = {
    val ds = new BasicDataSource()
    ds.setDriverClassName("org.driver.Classname")
    ds.setUrl("...")
    // set username, options
    ds
  }
 
}
 
object SparkApplication {
 
  def main(args: Array[String]) {
    // init Spark Context sc
    sc.parallelize(1 to 10)
        .map(i => {
          val conn = Holder.ds.getConnection
          // do something
          conn.close()
        })
        .count
  }
}
```

**Why does it work?** 
Scala functions declared inside objects are equivalent to static Java methods. In order to call a static method, you don’t need to serialize the class, you need the declaring class to be reachable by the classloader (and it is the case, as the jar archives can be shared among driver and workers).

Creating external objects can be tedious. I’ve included some utilities in the [https://github.com/nerdammer/spark-additions](https://github.com/nerdammer/spark-additions) library.

For example, to share the datasource one might write:

```scala
// Declare it as a shared variable
val dsv = SharedVariable {
  val ds = new BasicDataSource() // not serializable
  ds.setDriverClassName("org.driver.Classname")
  ds.setUrl("...")
  // set username, options
  ds
}
 
sc.parallelize(1 to 10)
  .map(i => {
    val ds = dsv.get
    // do something
  })
  .count
```

Any call to *dsv.get* that is executed at the worker side will create a new instance of the datasource. 
Alternatively the library allows you to define a *SharedSingleton*. 
There will be at most one instance of the *SharedSingleton* in each JVM of the Spark cluster.

```scala
// An object that acts as a singleton in each JVM of the cluster
val connectionPool = SharedSingleton {
  // initialization code
}
 
sc.parallelize(1 to 10)
  .map(i => {
    val cp = connectionPool.get
    // use the local-JVM connection pool
  })
  .count
```

Both *SharedVariable* and *SharedSingleton* objects can be used to share non-serializable objects that need to be used in task closures. 
There are other solutions that work in these circumstances, but boilerplate is reduced when using shared variables. 
More info [here](https://github.com/nerdammer/spark-additions/tree/master/core/src/main/scala/it/nerdammer/spark/additions/variables).