---
title:  "Logging to a NoSQL DB from Spark"
modified: 2016-08-31 12:00:00 +0200
last_modified_at: 2016-08-31 12:00:00 +0200
tags: [Apache Log4j, Apache Phoenix, Apache Spark, Docker, Scala]
categories: [Dev]
header:
    image: post-phoenix-logo.png
    teaser: post-phoenix-logo.png
---
Logging effectively is often a hard task in standard applications. But when the application runs in a distributed environment, for instance, a Spark job in a big YARN cluster, 
it becomes ten times harder.

Jobs are split into thousands of tasks that run inside multiple worker machines, 
so the classic **console logging is not a good option**, because the logs get written to the standard output of 
several remote machines, making it impossible to find useful information.

One of the best options available in all modern data platforms is **logging to a NoSQL database**. 
Many data platforms support HBase and Phoenix as NoSQL layer, so why don’t using a Phoenix table to store the logs?

First of all, the table must be created inside Phoenix, and it must be **optimized for efficiently writing the log data**.
For example:

```sql
CREATE TABLE LOG 
(
  LOG_DATE  TIMESTAMP  NOT NULL,
  LOG_ID    BIGINT     NOT NULL,
  LOGGER    VARCHAR(150),
  LEVEL     VARCHAR(10),
  MESSAGE   VARCHAR(8192)
  CONSTRAINT LOG_PK PRIMARY KEY (LOG_DATE, LOG_ID)
) SALT_BUCKETS = 50;
 
CREATE SEQUENCE SEQ_LOG_ID MINVALUE 1;
```

The *LOG* table is defined as a **salted table** with *50* buckets 
(the number can be increased/decreased, depending on the size of the cluster). 
Bucketing is needed to spread the data across multiple region servers, 
in order to balance the load across all the machines. 
The *LOG* table has a timestamp as first column of the primary key (that is a monotonically increasing field) so, 
if salting buckets were not in place, only one machine at a time would be used to store the logs. 
The *LOG_ID* column is part of the primary key. It is useful to prevent collisions among log messages.

Once the table is defined, we need to configure *log4j* to store log messages inside it. 
Phoenix is compliant with the JDBC API. 
This allows using the `JDBCAppender`. 
Unfortunately, the standard `JDBCAppender` is not perfect for being used with Phoenix, 
just because **it does not commit the transactions**. 
Of course, **I’m not saying that Phoenix supports transactions** (not now). 
Phoenix requires that you commit the `UPSERT` statements, 
otherwise they will remain stuck in the driver’s cache. 
So, we need to write a custom extension of the `JDBCAppender`:

```java
package it.nerdammer.log4j;
 
import org.apache.log4j.jdbc.JDBCAppender;
 
import java.sql.Connection;
import java.sql.SQLException;
 
public class PhoenixAppender extends JDBCAppender {
 
    protected Connection getConnection() throws SQLException {
        Connection connection = super.getConnection();
        connection.setAutoCommit(true);
 
        return connection;
    }
 
}
```

The class above just sets the `autocommit=true` property on every connection created by the `JDBCAppender`.

Now that we have an appender compatible with Phoenix, the next step is configuring it in the *log4j.properties* file:

```bash
# Root logger option
log4j.rootLogger=INFO, stdout
 
log4j.logger.com.enterprise=INFO, phoenix
log4j.additivity.com.enterprise=true
 
# Direct log messages to stdout
log4j.appender.stdout=org.apache.log4j.ConsoleAppender
log4j.appender.stdout.Target=System.out
log4j.appender.stdout.layout=org.apache.log4j.PatternLayout
log4j.appender.stdout.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss} %-5p %c{1}:%L - %m%n
 
 
log4j.appender.phoenix=it.nerdammer.log4j.PhoenixAppender
log4j.appender.phoenix.URL=jdbc:phoenix:phoenixhost
log4j.appender.phoenix.user=anyuser
log4j.appender.phoenix.password=anypassword
log4j.appender.phoenix.driver=org.apache.phoenix.jdbc.PhoenixDriver
log4j.appender.phoenix.sql=UPSERT INTO LOG (LOG_DATE, LOG_ID, LOGGER, LEVEL, MESSAGE) VALUES ('%d', NEXT VALUE FOR SEQ_LOG_ID, '%C', '%p', '%m')
log4j.appender.phoenix.layout=org.apache.log4j.PatternLayout
```

An appender named `phoenix` has been created. 
It has been associated with the logger named “com.enterprise”. 
You can change it to the base package of your application, but **note: 
you cannot associate the phoenix appender to the root logger**. The reason is that **Phoenix itself uses log4j 
while initializing the connection to the database**. 
If you allow the logger `org.apache.phoenix` to append logs to the Phoenix table, 
you will get soon a **stackoverflow** error. One way of breaking the loop it is limiting the usage of the “phoenix” 
appender to just few packages of your application (in the example `com.enterprise` and all its sub-packages).

In order to test locally, you can just run one docker container with everything preinstalled, 
for example, **my dockmob container for Phoenix** [https://hub.docker.com/r/dockmob/phoenix/](https://hub.docker.com/r/dockmob/phoenix/), 
using the following script:

```bash
#!/bin/bash
# 
# Before executing the script, add to /etc/hosts the following entry
# <docker-machine-ip> phoenixhost
#
# Where <docker-machine-ip> is the IP assigned to the docker machine on OSX (usually 192.168.99.100), 
# or 127.0.0.1 on Linux
 
MYPHOENIX_ID=$(docker run -d -p 2181:2181 -p 60000:60000 -p 60010:60010 -p 60020:60020 -p 60030:60030 -h phoenixhost dockmob/phoenix:4.5.2-1.0.1 -t pseudodistributed)
docker exec -it $MYPHOENIX_ID /usr/lib/phoenix/bin/sqlline.py localhost
```

The last command starts a **SQL console** on the Phoenix instance. 
You need to paste the previous SQL commands to **create the LOG table**.

**Ensure you have put all the required libraries for writing to Phoenix in the Spark application classpath**. 
Here is a simple test for verifying if it works:

```java
package com.enterprise.test
 
import java.sql.{DriverManager}
 
import it.nerdammer.log4j.PhoenixAppender
import org.apache.log4j.Logger
import org.apache.spark.{SparkConf, SparkContext}
import org.scalatest.{BeforeAndAfterAll, FlatSpec, Matchers}
 
class SparkJobTest extends FlatSpec with Matchers with BeforeAndAfterAll {
 
  lazy val sc: SparkContext = {
    val conf = new SparkConf()
      .setAppName("Console Test")
      .setMaster("local")
 
    new SparkContext(conf)
  }
 
  override protected def beforeAll(): Unit = sc
 
  override protected def afterAll(): Unit = sc.stop()
 
 
  "the console logger " should "work" in {
 
    val databaseURL = Logger.getLogger("com.enterprise").getAppender("phoenix").asInstanceOf[PhoenixAppender].getURL
    val conn = DriverManager.getConnection(databaseURL, "any", "any")
    conn.setAutoCommit(true)
    val pstm = conn.prepareStatement("delete from log")
    pstm.execute()
    pstm.close()
 
    sc.parallelize(1 to 10)
      .map(e => {
        Logger.getLogger(classOf[SparkJobTest]).info("This is a log message")
        e
      })
      .count();
 
    val pstm2 = conn.prepareStatement("select count(*) from log")
    val rs = pstm2.executeQuery()
    rs.next()
    val count = rs.getInt(1)
    rs.close()
    pstm.close()
    conn.close()
 
    assert(count == 10)
  }
}
```

Now the logs appear if you query the LOG table using the SQL console. 
You can find the full code here: [https://github.com/nerdammer/spark-additions/tree/master/log4j-phoenix](https://github.com/nerdammer/spark-additions/tree/master/log4j-phoenix).