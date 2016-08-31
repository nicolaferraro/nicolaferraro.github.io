---
title:  "Exception Handling in Apache Spark"
date:   2016-02-18 17:29:32 +0200
tags: [Apache Spark, Scala]
header:
    image: post-logo-spark.png
    teaser: post-logo-spark.png
---
Apache Spark is a fantastic framework for writing highly scalable applications. 
Data and execution code are spread from the driver to tons of worker machines for parallel processing. 
But debugging this kind of applications is often a really hard task.

Exceptions need to be treated carefully, because a simple runtime exception caused by dirty source data can easily 
lead to the **termination of the whole process**. Let’s see an example.

```scala
// source contains dirty data
val transformed = source
  .map(e => myCustomFunction(e))
  // do some other actions
 
transformed.saveAsTextFile("xxx")
```

The code above is quite common in a Spark application. 
Data gets transformed in order to be joined and matched with other data and the transformation algorithms 
are often provided by the application coder into a *map* function. 
It is clear that, when you need to transform a RDD into another, the *map* function is the best option, 
as it changes every element of the RDD, without changing its size. 
But an exception thrown by the *myCustomFunction* transformation algorithm causes the **job to terminate** with error.

In the real world, a RDD is composed of millions or billions of simple records coming from different sources. 
The probability of having wrong/dirty data in such RDDs is really high. In these cases, instead of letting 
the process terminate, it is **more desirable to continue processing** the other data and analyze, at the end 
of the process, what has been left behind, and **then decide if it is worth spending some time to find the 
root causes of the problem**.

How should the code above change to support this behaviour? A first trial:

```scala
val transformed = source
  .flatMap(e => Try{myCustomFunction(e)}.toOption)
  // other actions
```

Here the function *myCustomFunction* is executed within a Scala *Try* block, then converted into an *Option*. 
The code is put in the context of a *flatMap*, so the result is that all the elements that can be converted 
using the custom function will be present in the resulting *RDD*. Elements whose transformation function throws 
an exception will be automatically discarded. Pretty good, **but we have lost information about the exceptions**. 
Can we do better?

**Why don’t we collect all exceptions, alongside the input data that caused them?** 
If the exception are (as the word suggests) not the default case, they could all be collected by the driver 
and then printed out to the console for debugging. What I mean is explained by the following code excerpt:

```scala
// define an accumulable collection for exceptions
val accumulable = sc.accumulableCollection(mutable.HashSet[(Any, Throwable)]())
 
val transformed = source.flatMap(e => {
  val fe = Try{myCustomFunction(e)}
  val trial = fe match {
    case Failure(t) =>
      // push to an accumulable collection 
      // both the input data and the throwable
      accumulable += (e, t)
      fe
    case t: Try[U] => t
  }
  trial.toOption
})
 
// call at least one action on 'transformed' (eg. count)
transformed.count
```

Probably it is more verbose than a simple *map* call. 
I will simplify it at the end. Now that you have collected all the exceptions, you can print them as follows:

```scala
// at the end of the process, print the exceptions
accumulable.value.foreach{case (i, e) => {
  println(s"--- Exception on input: ($i)")
  // using org.apache.commons.lang3.exception.ExceptionUtils
  println(ExceptionUtils.getStackTrace(e))
}}
```

So far, so good. Now you can **generalize the behaviour and put it in a library**. 
Or... you'd better use mine: [https://github.com/nerdammer/spark-additions](https://github.com/nerdammer/spark-additions).

The **Spark Additions** way is a lot **easier**:

```scala
// import all implicit conversions
import it.nerdammer.spark.additions._
 
// ...
val transformed = source
  .tryMap(e => myCustomFunction(e))
 
// call at least one action on 'transformed' (eg. count)
transformed.count
```

The *tryMap* method does everything for you.

What you need to write is the code that gets the exceptions on the driver and prints them. Very easy:

```scala
// sc is the SparkContext: now with a new method
sc.accumulatedExceptions()
  .foreach{case (i, e) => {
    println(s"--- Exception on input: ($i)")
    println(ExceptionUtils.getStackTrace(e))
  }}
```

More usage examples and tests [here (BasicTryFunctionsIT)](https://github.com/nerdammer/spark-additions/blob/master/core/src/test/scala/com/enterprise/integration/tests/tryfunctions/BasicTryFunctionsIT.scala). 
Look also at the [package implementing the Try-Functions](https://github.com/nerdammer/spark-additions/tree/master/core/src/main/scala/it/nerdammer/spark/additions/tryfunctions/) (there is also a **tryFlatMap function**).

Contributions are always appreciated.