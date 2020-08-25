---
title:  "Spark-HBase-Connector 1.0.2 is out"
modified: 2016-08-31 12:00:00 +0200
last_modified_at: 2016-08-31 12:00:00 +0200
tags: [Apache HBase, Apache Spark, Scala]
categories: [News]
header:
    image: images/post-logo-spark.png
    teaser: images/post-logo-spark.png
---
The Spark-HBase-Connector project started as a 3-days programming marathon I made last year. At home, with the flu. 
Now it is becoming one of the most popular drivers to read/write data to Apache HBase from Spark.

The project is hosted on github: [https://github.com/nerdammer/spark-hbase-connector](https://github.com/nerdammer/spark-hbase-connector).
The library is also available on Maven. You can find more information on the github page.

Probably, its future will be limited by the advent of Apache Phoenix but, for now, 
Phoenix is running slowly and the connector’s popularity is increasing.

Some small improvements have been applied to the code base for version 1.0.2. Now Spark-HBase-Connector supports Spark 1.6.0+ and HBase 1.0.1.1+. 
Support for tables with up to 22 columns has been added.

Thanks for the contributions arrived so far. Pull requests are always appreciated.