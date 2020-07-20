
### These notes are based on the following lectures: Spark Core and Internals
- 2015 by Sameer Farooqui, link
- by Daniel Tomes
- deeper understanding


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

Avoiding Lopsided Cached RDD's

### ============================================
#### Aside; Shuffles
link

What is a Shuffle: Shuffles are one of the most memory/network intensive parts of most Spark jobs

Consider an "embarrassingly parallel" process
- reading in a number of text files (as partitions), apply a map, and write out to disk

Bring a shuffle into the picture
- same operation, but introduce a reduceByKey() after map, effectively implementing a result that contains a list of tuples where the first element is a word and the second element is the number of occurances in the text
- in order to count all of the given words which may appear across dataset (with a bunch of partitions), each parition must aggregate all the counts of words within that partition, but then it must also sum across each **other** partition
- Process of moving data from partition to partition in order to aggregate, join, match up, in some way is known as **shuffling**
- aggregation/reduction that takes place before data is moved across partitions known as **map-side shuffle**

Some shuffle inducing operations:
- groupBy/aggregateBy/ReduceByKey
- cogroup
- any join transformation
- distinct

Shuffle Manager:
- 

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

**Spark Local Node:**
- JVM: Executor + Driver
- inside, cached RDD's, Task Slots (Cores, num_cores)
- **oversubscribe** number of cores by factor of 2-3x, if you have 6 core, you can assign 12-18 task slots
- ``` spark-shell --master local[12] ```, local[*], run with as many threads (tasks) as there are logical cores
- some threads, about 20, are "internal threads" and conduct unrelated tasks to map-reduce

**Spark Standalone Mode:**
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
    - SPARK_WORKER_CORES --> Number of cores (threads) a worker JVM can to provie to its executors
    - SPARK_WORKER_MEMORY --> Total memory to allow Spark workers to allocate to its executors
    - SPARK_DAEMON_MEMORY --> Memory to allocate to Spark master JVM and worker daemon themselves 

- standalone mode settings: Apps submitted will run in FIFO mode by default 
    - ```spark.cores.max```: maximum amount of CPU cores to request for the application from across the cluster
    - ```spark.executor.memory```: max memory for each executor
        - err on the side of potentially allowing a single worker JVM to allocate all the clusters memory and cores to a single executor 


**Spark YARN Mode:**
- A Resource Manager (Master Machine), Many Slave Machines in Hadoop cluster each run runs a Node Manager
- Node Managers heartbeat with Resource Manager --> "my node has 50% of bandwidth occupied", other resource info
- a Client Application, submits application to Resource Manager, with instructions and constraints for APp Master
    - RM starts an App Master in a container inside on of the Slave Machines, alongside the existing Node Manager
- App Master (like a Job Scheduler), contacts the RM and asks for containers
    - RM gives the App Master keys and tokens necessary to start Containers on each of the other Slave Machines
    - Containers register with the App Master
    - App Master can make very detailed requests regarding specs of containers it requests
- The RM is out of the loop once app is running--> App Master communicates directly with Client
- can run multiple applications, independently 
- RM contains: 
    1) a Scheduler, where app masters will run and where containers will be scheduled 
    2) an Apps Master (plural); if the app masters crash, the RM can restart the app master on the same or on a new container

Two ways to run Spark in YARN
- **Client Mode**
    - driver  runs on the client itself (a laptop)
    - interactively work with large data set
    - worker machines have containers containing executors; the executors in direct contact with Driver (running on client)
        - the App Master just negotiates with Resource Manager (similar to Spark Master) where Executor JVM's will run (not so important)
    - More important Scheduler inside Driver --> organizing where tasks should be allocated, across different Containers
- **Cluster Mode**
    - want to submit application and walk away, non-interactive
    - Client submits the application and driver. Driver ends up running on the App Master on one of the Worker Machines

**YARN Settings:**
    - num-executors: controls how many executors that will be allocated (not possible in standalone)
    - executor-memory (RAM for each executor)
    - executor-cores (CPU Cores for each executor)
    - Dynamic Allocation; inc/dec number of executors live in application (see details at 2:28)

### ============================================
#### Memory and Peristence 

**Persist an RDD to memory**
- **Persist to Memory:**
    - RDD.cache() --> RDD.persist(MEMORY_ONLY)
    - storing the de-serlized RDD inside the JVM 
    - if certain partitions of RDD do not fit in memory, not cached and dropped. Recomputed on the fly from udnerlying data source or earlier cached RDD. No fraction of a partition can disappear, all or nothing. 
    - Caching a large RDD may result in evacuation of earlier cached RDD's if memory constraints demand
    - be strategic in what you cache, and lost executors will jhave to have cached RDD's re computed at action
- **persist to Memory, serialized:**
    - RDD.persist(MEMORY_ONLY_SER)
    - More space efficient, but consider using fast serialization library (recommend Cryo)
    - slows down overall compute performance because of serialization/de-serialization, but greater cache availability
    - reduces cost of garbage collection 
- **Persist to memory and disk:**
    - RDD.persist(MEMORY_AND_DISK):
    - Basically, try to store RDD partitions, de-serialized, in JVM Memory. If do not fit, oldest paritions in cache moved to disk (LOCAL DIRS directory)
    - In disk, the partitions are stored in serialized
    - can run with MEMORY_AND_DISK_SER to foce serialization on both disk and memory 
- **Persist to disk only:**
    - RDD.persist(DISK_ONLY)
- **Persist to two different JVM's:**
    - RDD.persist(MEMORY_ONLY_2)
    - store RDD partitions as de-serialized in two different JVM's
    - if RDD is extermely costly to create (for example using wide partitions)
    - not recommended
- **Tachyon (now Alluxio):**
    - RDD.persist(OFF_HEAP)
    - link! 
    - off heap storage, faster than disk
    - safe from executor crash 
    - intermediate between pyspark, scala, other API's
    - more here: 
- **Un Persist**:
    - RDD.unpersist()
- **Persist to memory only**:
    - rdd.persist(MEMORY_ONLY)
    - MEMORY_ONLY_SER with fast ser library
    - don't spill to disk if possible, recomputing can be just as quick many times thatn reading from disk 
    - replicated storeage levels (MEMORY_ONLY_2) UNLESS it's vitally imporant 

Notes:
    - Intermediate data is automatically persisted during a **shuffle** opeartion
    - In PySpark, stored objects will always be serialized in memory with Pickle library

**Default memory allocation in exec JVM**
- 60% Cached RDD (spark.storage.memoryFRaction)
    - employed when user calls .pesrist()
    - can be relieved by using Allexio (formely Tachyon)
- 20% Shuffle memory (spark.shuffle.memoryFRaction)
    - employed during shuffle operations. Spark creates intermediate buffers for storing shuffle output data. 
- 20% user programs (remainder)
- if you see exec JVMs with OutOfMemory Error:
    - can be one of these three
    - most likely shuffle memory or user programs

**Determine RDD Memory Consumption**
- look at logs for SparkContext on Driver

**Data Serialization**
- serialization used when 
    - transferring data over network
    - spilling data to disk 
    - caching to memory serialized 
    - broadcasting variables 
- Note choice between Java serialization vs. Kryo
    - Kryo serialization: Faster
        - recommended for production
        - conf.set("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
        - not default because not all data classes are default supported by Kryo, classes must be registered first

**High Churn: Tuning Note**
- when you are caching too many RDD's, any many existing cached RDD's must either be deleted or moved to disk when memory exhaustive (garbage collection is expensive. cost proportional to number, not size of objeccts)
- COst of Garbage colleciton, different options:
    - Parallel GC, CMS GC, G1 GC


### ============================================
#### Application: Jobs --> Stages --> Tasks

When you call an action, collect(), that action triggers a **job**
- the **job** made up of multople stages, that may or may not run in parallel
- **stages** made up of multople **tasks**, which are always run in parallel
- each task is operating on one partition:
    - read a partition from a parent RDD, processing, emitting an new child partition 

Scheduling Process:
- **RDD Object:**
    - transformations build operator DAG
- **DAG Scheduler:**
    - once action called, Spark Driver JVM finds how to carve up the DAG into **stage boundaries**
    - reduce the original lineage graph into stages, and organize stages by tasks. 
    - stages submitted as ready
- **Task Scheduler:**
    - accepts stages from DAG scheduler (as parallelism between stages allows)
    - launches and schedules individual tasks on individual executors 
    - after successfully sending all tasks (and receiving success messages) from a stage, tells DAG Scheduler
- **Executor:**
    - contains task threads and block manager
    - execute tasks, and store & servce blocks

**Lineage**
- recall that an RDD graph has a lineage: parent rdd --> transformation --> child rdd
- dependencies can be either **wide** or **narrow**
- **Wide**, where multiple child partitions may depend on it. **Narrow** where each partition of the parent RDD is used by at most one partition iof the child RDD 
- Narrow:
    - map, filter, **join with inputs co-partitioned** (similarly hash partitioned), union
- Wide **Requires Shuffle**:
    - groupByKey, join with inputs **not** co-partitioned










