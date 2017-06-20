# Hardware

If you want to get serious about Elasticsearch, you’ll have to learn about hardware. It might be an unpopular opinion in 2017, but don’t run Elasticsearch in the cloud. It has nothing to do with latency or losing your AWS spot instances because Netflix has just released a new show, it has to do with picking up the right hardware for your needs. Cloud providers such as AWS provides vCPU but there’s no way you know what you’re going to get.

![](https://cdn-images-1.medium.com/max/1600/1*-0zz282vZ9s-WfKr7syzOg.png)

Because they have trouble with Java garbage collection, the first thing people ask advice about is memory management. What you should actually care about is, in no particular order:

* CPU
* Memory
* Network
* Storage

#### CPU {#38fd}

Running complex filtered queries, intensive indexing, percolation and queries against non latin charsets have a strong impact on the CPU, so picking up the right one is critical.

Dive into the CPU specs to understand how they behave with Java. It’s been more than a decade since I last read Intel specs — I even used to have them as physical books — but it prove itself critical to pick up the right hardware.

For example, Xeon E5 v4 provides 60% better performances than the v3 version when running Java. Xeon D works well when you want to scale your cluster horizontally as soon as heavy indexing is split evenly amongst the cluster nodes. Prepare to get into trouble with nodes popping out of the cluster like popcorn otherwise. So picking up the right CPU and horizontal design is critical.

Speaking of CPU, Elasticsearch divides the CPU use into[thread pools](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-threadpool.html)of various types:

* _generic: _for standard operations such as discovery
* _index: _for indexing
* _get: _for get operations, obviously
* _bulk: _for bulk operations such as bulk indexing
* _percolate: _for percolation
* _… _these are the most important ones you'll have to deal with, RTFM for everything else

Each pool runs a number of threads, which can be configured, and has a queue, which can be configured too. The default number of threads is defined using a system variablecalled_allocated\_cpu_, which is never greater than 32, even though you have 48 core and the system variable\_available\_cpu\_shows 48.

I wouldn't recommend changing the thread pool size unless you really know what you do, the defaults settings are quite sensible. You might want to adapt the queue size though, as filling the queue size means the operations will be rejected. You can get a glance of your thread pool state using the thread\_pool API.

```
GET /_cat/thread_pool/search?v&h=host,name,active,rejected,completed
```

A special trick when your cluster is CPU bound and you can support a replica of your data set on every node: run your cluster behind a Haproxy and bypass the http nodes to hit the data node. If you have heterogeneous nodes, give a greater weight to the nodes with the highest number of cores.

#### Memory {#1cbc}

Elasticsearch runs on Java, and Java is a garbage collected language. Which means you'll run into memory management problems.

Memory is divided in 2 parts: what you allocate to the Java heap space, and everything else. Elasticsearch does not rely on Java heap only. For example, every thread created within the thread pool allocates 256Kb of off-heap memory.

> The basic thing to understand with heap allocation is: the more memory you give it, the more time Java spends garbage collecting.

Elasticsearch comes with[Concurrent Mark Sweep](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/cms.html)as a default garbage collector. CMS runs multiple concurrent threads to scan the heap for objects that can be recycled. The main problem with CMS is how it might enter “stop the world” mode in which the JVM becomes unresponsive until it is restarted. The main reason for stop the world is when the application has changed the state of the heap while CMS was running concurrently, forcing it to restart from scratch until it has all the objects marked for deletion. Let's put it this way: CMS sucks donkey balls when the heap is over 4GB, which is almost always the case with Elasticsearch.

Java 8 brings a brand new garbage collector called Garbage First, or G1, designed for heaps greater than 4GB. G1 uses background threads to divide the heap into 1 to 32MB regions, then scan the regions that contain the most garbage objects first. Elasticsearch and Lucene does not recommend using G1GC for many reasons, one of them being[a nasty bug on 32bits JVM](https://bugs.openjdk.java.net/browse/JDK-8038348)that might lead to data corruption. From an operational point of view, switching to G1GC was miraculous, leading to no more stop the world and only a few memory management issues.

That said, choosing the right amount of memory to fill in the heap is the most touchy part of designing an Elasticsearch cluster. Whatever you pick up, never allocate more than 31GB to the heap.

Elasticsearch provides plenty of metrics to understand how the workload wights on the memory.

```
GET /_nodes/stats
```

Elasticsearch uses multiple buffers to perform in memory operations, as well as caches to store the queries results with a system of LRU when the cache becomes full. When the results are mostly large datasets and the queries are not repeated often, disabling the caches might be a good idea.

The caches you need to monitore are:

* the [_query cache_](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-cache.html): defined per node, with a default of 10% of the heap.
* the [_shard request cache_](https://www.elastic.co/guide/en/elasticsearch/reference/current/shard-request-cache.html): used to compute the result of queries ran on multiple shards.
* the [_fielddata cache_](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-fielddata.html): limited to 30% of the total heap.

Since version 5, Elasticsearch buffers were simplified, and there are only 2 buffers to monitor:

* the[_indexing buffer_](https://www.elastic.co/guide/en/elasticsearch/reference/current/indexing-buffer.html)_:_it is used to buffer data during the indexing process.
* the _buffer\_pools_
  .

Elasticsearch is not idiotproof and won't tell you if you allocate more than 100% of the heap to the multiple buffers and caches, unless you try to fill them all at the same time. Then you'll get an out of memory error.

I said earlier that too much memory might lead to management issues. Actually, the more memory the better when you play outside of the heap. The off heap memory is used to manage threads and for the filesystem to cache the data.

Elasticsearch[file system storage](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules-store.html)has an important impact on the cluster performances. After trying both ElasticSearch default\_fs and mmapfs, I’ve picked up niofs for file system storage.

> The NIO FS type stores the shard index on the file system \(maps to Lucene NIOFSDirectory\) using NIO. It allows multiple threads to read from the same file concurrently.

Niofs lets the kernel manage the file system cache instead of relying on the broken, out of memory error generator mmapfs.

You might also want to commit the exact amount of memory you want to allocate to the heap at startup. This prevents the node from swapping when trying to allocate the memory it needs because no more memory is available.

```
boostrap.memory_lock (previously bootstrap.mlockall)
```

#### Network {#a6f5}

Let's put it this way: you never have too much bandwidth. 1GB is good, 10GB is better, and a low latency is even better. Elasticsearch performs lots of network consuming operations, from transferring data during queries to reallocating shards, so networking matters.

The multicast discovery plugin was removed from Elasticsearch 5, so discovery is done either using unicast or a cloud plugin, but you won't run Elasticsearch in the cloud, will you?

If your hosting provider allows it, activate the Jumbo frames on your network interfaces. Jumbo frames might reduces the network latency by about 15% which is noticeable when transferring large amount of data.

```
ifconfig eth0 mtu 9000
```

Elasticsearch comes with some interesting network related settings, which are low by default and won't go over 2Gb/s, notably the recovery transfer which is limited to 40mb/s

```
indices.recovery.max_bytes_per_sec: 2g
```

Raise this value only if your storage can handle it while serving queries, indexing, and performing administrative tasks such as merges.

#### Storage {#a2b2}

After memory, storage is often the bottleneck of an Elasticsearch cluster. Unless you have a good reason to do it, don't use spinning disks. Large SSD drives are now affordable with a decent life expectancy, and you'll need fast IOs. Also, prefer local storage to anything else to reduce the reads and writes latency. Consider your data nodes as disposable when possible.

A question that comes often about storage design is:**should I go with RAID0, RAID1\(0\) or JBOD?**

**RAID0**offers the best cost / speed / storage space ratio. It fits perfectly in large clusters where losing a node is not a problem. RAID0 offers the maximum storage space on a single file system, which is convenient when managing large shards. Without enough available storage on a single node, operations like index optimisation won't be possible. RAID0 also offers the maximum number of axes, without the RAID1\(0\) replication overhead, useful for intensive indexing.

On the other hand, losing a single disk means losing a whole data node, so choosing RAID0 implies to have enough spare data nodes to store the whole dataset in case of crash.

**JBOD**offers the best cost / storage / security ratio. Each disk is affected to a mountpoint, and the mountpoints are listed in Elasticsearch configuration. JBOD is a good choice when you can't afford to lose a whole data node, but losing a whole disk is OK, but provides less read and write performances. Running large shards on JBOD can also be a problem to perform administrative tasks like index optimisation.

Depending on how many data nodes you can afford to lose, running many hosts with software RAID0 is the best speed / storage space / cost setup. Otherwise, use JBOD to reduce the I/Os involved by RAID 1 and RAID10 replication.

**RAID1\(0\)**is the option for people who run Elasticsearch on a single node. It provides the maximum security for the data availability as losing a disk is possible, but at the cost of space and performances. RAID1\(0\) implies to use half of the storage for the RAID replication, and the replication overhead is something to take into account.

Elasticsearch comes with 2 storage related throttling protection. The first one limits the bandwidth of the storage you can use, and is as low as 10mb/s. You can change it in the nodes settings:

```
indices.store.throttle.max_bytes_per_sec: 2g
```

The second one prevents too many merges from happening, which slows down your indexing process. If you run bulk indexing or don't care about search speed, you can disable merge throttling entirely.

```
indices.store.throttle.type: "none"
```



