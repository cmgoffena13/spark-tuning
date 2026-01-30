# Spark Tuning

## Rule of Thumbs & Magic Numbers
- Number of Partitions: 3x the number of cores in cluster
- Number of Cores per Executor: 4-8
- Allocated Memory of Executor for Processing: 85% of Total Memory
- Shuffle Partition Size (MB): 200mb

## Spark Architecture Granularities
### Cluster
- Number of Total CPU Cores
- Number of Total Partitions (in each stage)

### Executor
- Allocated Memory for Processing

### CPU Core / Partition
- Partition Size (MB)

## Walkthrough
A partition in Spark is a chunk of data. A partition is processed by 1 CPU Core. It is important to note that if data is compressed, any sizes are of the compressed data. They are only uncompressed during executor processing.

### Read Phase
When Spark reads in data, it automatically chunks out the data into read partitions, doing its best to have each partition size be the default (128MB). This means the number of partitions read is variable through roughly this formula:  
 `Total Size of Data Read (Compressed) / 128mb` -> Total Read Partitions

### Shuffle Phase
A shuffle occurs during wide transformations, `group by`, `order by`, etc. When a shuffle occurs, data is transferred across executors to group the data in shuffle partitions for the wide transformation to occur. The partition size is variable through roughly this formula:  
`Total Size of Data Read (Compressed) / 200 (Default Total Shuffle Partitions)` -> Shuffle Partition Size (MB)  

Because the default for total shuffle partitions is 200, with no automatic scaling, it is highly likely that this config will need adjusted. Especially if the total number of cpu cores in the cluster is greater than 200 (because this would mean the cluster is not utilizing the maximum parallelism).

### Write Phase
The end of a spark job normally ends up writing data. The partition size and number of partitions for this phase are determined by the previous phase (Shuffle or Read). There is an interesting dilemma here because the ideal write size of partitions is 128-200mb, but the shuffle phase can change the partition size and number of partitions. There is the `spark.databricks.delta.optimizeWrite.enabled` = true config that will trigger *another* shuffle to align the written partition sizes for optimal reads downstream.

## optimizeWrite vs Optimize
Given the complexity involved in shuffle operations, having `optimizeWrite` be true on a Spark job may underutilize the cluster, but optimizing the files in a table to be 128-200mb size improves read operations for all downstream jobs. It is an option to skip optimized writes and have a smaller cluster use the `optimize` command on a table to group smaller files together as a background job.

## AQE
Adaptive Query Execution (AQE) is turned on by default. It optimizes joins based upon runtime statistics, but we are mostly concerned with its ability to merge small shuffle partitions together. It targets ~64mb size partitions if there are small partitions detected. This means if we over-allocate the number of partitions and the size of partitions becomes small, AQE will merge smaller partitions together to help prevent the "small file problem". The reason why the default is ~64mb is because it is optimized for write parallelism rather that optimized file size for writes. This means faster writes.

## Small File Problem
The main problem with having too many files that are small is an I/O and metadata scanning problem. Which is why Spark wants to aggregate data together into bigger partitions to be more performant. A simple example is inserting 1000 records into a database one by one is slower than sending all 1000 records over in a batch statement. It's an I/O problem. The small file problem is a performance killer.

## Large File Problem
Large files become large partitions and end up with executor memory exhaustion when multiple cores process them simultaneously Each executor is assigned a specific amount of memory. The tricky part is that each executor normally has multiple CPU Cores and Spark processes 1 Partition per CPU Core. So you have to make sure that the total number of partitions that are being processed on an executor does not exceed the total executor memory. When this occurs data spills to disk or, worse-case, you'll get an Out of Memory (OOM) error and the job will fail. The large file problem is a performance killer.

## Parallelism
Since a cluster has so many CPU Cores, it is possible for not all CPU Cores to be processing partitions if the number of partitions is not able to be allocated properly. The sweet spot to optimize parallelism is to have 3x the number of cores be the total partitions.

## Shuffle Spill
If you see a shuffle spill in the median quartile of the spark job, the partitions are too big. Due to the default read partition size of 128mb, this will most likely only happen when a shuffle is present in the job. The total number of partitions needs *increased* by increasing `spark.sql.shuffle.partitions` (See the formula below).

## Knobs
### Spark Conf
- spark.sql.files.maxPartitionBytes (default: 128mb)
    - Don't need to change this too often unless the output of your job increases data size, like exploding out an array.
    - The other reason to change this input is for parallelism (Total Partitions = 3x Number of CPU Cores in Cluster)
        - Decrease to 32-64mb for streams with small files to increase parallelism
- spark.sql.shuffle.partitions (default: 200)
    - Formula: `Total Size of Data Read (Compressed) / 200mb (Magic Number) = Total Shuffle Partitions`

### AQE
- spark.sql.adaptive.advisoryPartitionSizeInBytes (default: 64mb)
    - Increase to 128mb if you're already using optimized writes

## Run-Length Encoding
What we need to know is we can sort partition data with the `sortWithinPartition` spark command. If we designate the lowest cardinality columns to be sorted (and the largest sized columns, such as strings vs smaller sized columns like boolean), Run-Length Encoding kicks in and the data can be compressed *even smaller* be grouping the values together in columnar format. This benefits downstream jobs because they have to read less data. **Note: any shuffle that happens after the sort command completely unsorts the data**

## References
### Videos
- [How to Automate Performance Tuning for Apache Spark by Jean Yves Stephan](https://www.youtube.com/watch?v=ph_2xwVjCGs)
- [Apache Spark Core - Practical Optimization by Daniel Tomes](https://www.youtube.com/watch?v=_ArCesElWp8)