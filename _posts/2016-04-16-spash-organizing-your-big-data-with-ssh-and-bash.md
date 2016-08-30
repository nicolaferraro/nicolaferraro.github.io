---
title:  "Spash - Organizing your Big Data with SSH and Bash"
date:   2016-04-16 17:29:32 +0200
tags: [Apache Hadoop, Apache Spark, Bash, HDFS, Spash]
---
Spash is a command line tool for Big Data platforms that simulates a real Unix environment, 
providing most of the commands of a typical Bash shell on top of **YARN, HDFS and Apache Spark**.

Spash uses the HDFS APIs to execute simple file operations and Apache Spark to perform parallel computations on big datasets. 
With Spash, managing a Big Data cluster becomes *as natural as writing bash commands*.

Spash is an open source project from an idea of [Nicola Barbieri](http://nicola-barbieri.tumblr.com/). 
The code is here: [https://github.com/nerdammer/spash](https://github.com/nerdammer/spash).

## Architecture
The Spash daemon runs on an edge node of a Big Data cluster and listens for incoming SSH connections on port `2222`. 
Clients can connect using the OS native terminal application, or [Putty](http://www.putty.org/) on Windows.

The Spash daemon will emulate a Unix OS and leverage the power of Spark to perform efficient computations on distributed data.

![Spash Architecture](/images/spash.png)

## The World Before Spash
For those who don’t remember the classic way of doing simple operations on HDFS, here’s a reminder:

```bash
bash$ hdfs dfs -ls /
#
##
### [wait for the JVM to load and execute]
...
##########################################
bash$ hdfs dfs -copyFromLocal myFile /
#
##
### [wait for the JVM to load and execute]
...
##########################################
bash$
```

## Spash Makes It Easier
Just run the Spash daemon on a node of your Big Data cluster, 
you can connect to it using `ssh user@hdfshost -p 2222` (the password is `user`) and then run all your favourite 
bash commands to manipulate data.

```bash
user@spash:/$ echo "Maybe I can provide a real life example..."
Maybe I can provide a real life example...
 
user@spash:/$ echo "My content" > myFile
 
user@spash:/$ ls
myFile
 
user@spash:/$ cat myFile
My content
 
user@spash:/$ echo text2 > myFile2
 
user@spash:/$ ls -l
drwxr-xr-x 4 user supergroup 0 30 mar 18:34 myFile 
drwxr-xr-x 4 user supergroup 0 30 mar 18:35 myFile2
 
user@spash:/$ cat myFile myFile2 > myFile3
 
user@spash:/$ cat myFile3
My content
text2
 
user@spash:/$ cat myFile3 | grep -v 2
My content
 
user@spash:/$ exit
```

And this is just the tip of the iceberg. 
From your host, you can use scp (`scp myfile -P 2222 user@hdfshost:/`) to copy a file into hdfs. You can even use FileZilla or WinSCP to browse HDFS.

Spash works on Cloudera CDH 5 and many other platforms.

Spash is a *proof of concept* and needs contributions. 
Look at how to contribute and get involved on the project home page: [https://github.com/nerdammer/spash](https://github.com/nerdammer/spash).