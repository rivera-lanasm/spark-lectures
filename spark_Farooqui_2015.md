
### ============================================
### These notes are based on the following lectures:
- 2015 by Sameer Farooqui, link
- by Daniel Tomes


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
 
### ============================================
#### MapReduce: Spark compared to Hadoop
- in Hadoop framework, not just one phase of map-reduce, requires extensive chaining of these cycles
- these cycles linked by io reading and writing to HDFS
    - even small data processes can require extensive io cycles
    - ameliorated by some projects like Apache Oozie
- 10-100 times faster than Hadoop --> Spark
    - keeping data in memory between map-reduce cycles
    - memory access on RAM much faster than reading from disk or network! 

### ============================================
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

Using Apache Cassandra
- Strategic Caching RDD (like a view); a lazy operation
- 2 fast 2 furious
- Does not have to cache 100% of RDD, "graceful degradation" 
- saveToCassandra() --> save RDD's prior to collect() like action 

RDD's are typed
- HadoopRDD, filteredRDD, MappedRDD, PairRDD, ShuffledRDD, PythonRDD, etc. 
- should keep in mind both the names and types of RDD in processd 

Lifecycle of spark program:
- create som input (base) RDD's from external data or parallelize a collection in driver 
- lazily transform RDD's to define new RDD's using transformations (map(), filter(), etc.)
- cache() intermediate RDD's that wil need to be reused
- LAunch actions such as count() and collect() to run DAG, which is then optimized and executed by Spark

Spark Transformations
- looking into Scala source code, comments on transformations often much more detailed than docs
- row-wise and partition-wise transformations
    - RDD with 50 partitions, want to act on each partition (containing 1000 items each) and push items inside into a remote data source
    - want to do a partition level, NOT element wise operation --> want to open db connection 50 times, not 50*1000 times

RDD Interface: all spark RDD types are made by subclassing these interfaces (can be described in these terms)
- set of partitions (splits)
- list of dependencies on parent RDD's
- function to compute a partition given parents 
- optional --> preferred locations
- optional --> partitioner: partitioning info of k/v RDD's

### ============================================
#### Resource Managers

Ways to run Spark:
- Local 
- Standalone Scheduler
- YARN
- Mesos

Note:
- Local runs on one JVM/machine
- LOcal and Standalone Scheduler have static partitioning
  - cannot dynamically inc/dec number of executors **look into this**
- YARN and Mesos have dynamic partitioning
  - can start cluster with 10 executors, and grow shrink as is necessary 

Recall in traditional **Hadoop MapReduce**:
    - Spark does not replace HDFS
    - Hadoop: HDFS, Map Reduce, YARN
- JobTracker JVM --> Task Trackers in each executor
- Hadoop achieves parallelism by spawning more process ID's in each new JVM created
    - JobTracker --> brain of JVM scheduler
    - in spark, you get the same by spawning more threads inside each executor 
    - One executor JVM in each machine, and within each many slots (process ID's) exist where tasks run
    - in Hadoop, **slots inside executor are specified to either Map or Reduce** --> during Map phase, Reduce slots left unused (CPU Consumption underutitlized)
    - Heartbeat between task tracker and job tracker, which causes slowdown due to communication between executors and JobTracker
    - in spark, the re-use of slots contributes to speed, latency of assigning new slot very low

### ============================================
#### Running Spark

Spark Local Node:
- JVM: Executor + Driver
- inside, cached RDD's, Task Slots (Cores, num_cores)
- **oversubscribe** number of cores by factor of 2-3x, if you have 6 core, you can assign 12-18 task slots
- ``` spark-shell --master local[12] ```, local[*], run with as many threads (tasks) as there are logical cores
- some threads, about 20, are "internal threads" and conduct unrelated tasks to map-reduce

Spark Standalone Mode:
- ```spark-submit --name "secondapp" --master spark://host1:port1 myApp.jar```
- spark-env.sh --> important setting: SPARK_LOCAL_DIRS
    - used for when an 
        - 1) RDD is persisted with memory and disk and one partition must be spilled to local disk, then can go to one of the n SSD's defined by SPARK_LOCAL_DIRS
        - 2) for intermediate shuffle data; when a map-side operation spills data to local disk, and then a reduce-side operation pulls data off local disk (or over network), those map-spill files end up in SPARK_LOCAL_DIRS
        - should use SSD if you can --> Multiple SSD's will help because Spark can parallelize io writes this way
- starting machines: 
    - with an init scripts, multiple instances spin up, with each containing a worker JVM (which just spawns an executor) and one is chosen to contain spark master JVM as well, and each worker JVM registers with matser JVM 
    - master and worker JVM's are not large
    - **spark-submit** command: a driver starts on ONE machine, and tells spark master to provision executor JVM's.
        - spark master is a scheduler, where to launch executive JVM's 
        - tasks run inside executor JVM's
        - hardware configuration can be different across machines
        - can add more Master JVM's and can make Master JVM's highly available with ZooKeeper
    
    - Can run multiple applications
        - multiple sets of Driver <--> Executor Groups, the executors of all groups contained together in same machines

    - SPARK_WORKER_INSTANCES --> number of worker instance run in each machine
    - SPARK_WORKER_INSTANCES --> Number of cores a worker to provie to its executors
    - SPARK_WORKER_MEMORY --> Total memory to allow Spark workers to allocate to its executors
    - SPARK_DAEMON_MEMORY --> Memory to allocate to Spark master JVM and worker daemon themselves 


