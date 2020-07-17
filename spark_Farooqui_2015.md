
### ============================================
### These notes are based on the following lectures:
- 2015 by Sameer Farooqui, link
- 


### ============================================
#### Introduction: Spark core, center of Spark universe:
What is Spark?
- A scheduling, monitoring, and distributing engine for big data

- use memory and disk while processing data
- python, java, scala, and dataframes API's
- around spark core there are higher level libraries 
    - Spark SQL; working with structured data
        - leverages Hive (replacement for Hive)
    - Spark Streaming;
        - analyze real time streaming data; Twitter
    - MLlib
        - leverages Breeze and jblast Scala libraries
    - GraphX
    - BlinkDB
        - approximate queries
    - Tachyon (now Alluxio)
        - data sharing across cluster frameworks, memory based distributed storage system
- Resource managers (manage with ZooKeeper)
    - Local Node
    - Apache Yarn
    - Mesos
        - original spark resource manager
        - Spark is actually a use case app for Mesos
    - Spark Standalone
- File Systems
    - S3, HDFS, local, etc --> spark can read out of almost any file system, relational, nosql database, etc. 
- Flume and Kafka can be buffers for Spark streaming 
 

#### MapReduce: Spark compared to Hadoop
- in Hadoop framework, not just one phase of map-reduce, requires extensive chaining of these cycles
- these cycles linked by io reading and writing to HDFS
    - even small data processes can require extensive io cycles
    - ameliorated by some projects like Apache Oozie
- 10-100 times faster than Hadoop --> Spark
    - keeping data in memory between map-reduce cycles
    - memory access on RAM much faster than reading from disk or network! 

#### RDD Fundamentals
Main abstraction in spark is that of a resilient distributed dataset (RDD)
- represents a read only collection of objects partitioned across a set of machines that can be rebuilt if a partition is lost
- Lineage; how RDD's achieve fault tolerance. If a partition of an RDD is lost, the RDD has enough information about how it was derived from OTHER RDD's to be able to rebuild just that partition
- RDD's are motivated by two types of applications: 1) iterative algorithms and 2) interactive data mining tools

Using interactive shell for spark
- the shell is the driver program
    - driver sends code to worker machines
- an RDD --> contrains a number of partitions; more partitions = more parallelism 
- different partitions from a given RDD spread across different worker machines 
- each partition requires a thread of compute; a transformation against an RDD requires as many tasks, and each task is a thread running inside one of the worker machines 
    - 1000 core machine running against an RDD with only 5 partitions underutilizes parallelism 

RDD can be created by 
1) parallelize a collection 
  - in pyspark:
```
wordsRDD = sc.parallelize(['fish', 'cats', 'dogs'])
```
 - not genearlly used, since it requires the entire dataset fit in memory of driver JVM
2) read data from an external source (S3, HDFS, etc)
- more frequently used
```
linesRDD = sc.textFile("/path/to/file")
```

Base RDD or "Input RDD"
- an operation like filter, will produce a child RDD
- the child RDD will have the SAME number of partitions as parent, even if certain partitions are entirely wiped by filter
- coalesce transformation reduces RDD to derivative with fewer partitions
- collect pulls RDD into the driver node
- read() --> filter() --> coalesce() --> collect(): a common workflow

Transformations in Spark are lazy
- running transformations builds a DAG
- running an action executes that DAG


