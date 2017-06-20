# Designing your indices

Designing indices is the worst part of the cluster, because if you screw up, all you have left is to scratch and reindex. This part won't deal about designing a mapping, it's way beyond the cluster design. We'll talk about 2 things: sharding and replication.

#### Sharding {#3615}

Sharding is one of the reasons Elasticsearch is elastic. Elasticsearch divides the data in logical parts, so he can allocate them on all the cluster data nodes. The bad news is: sharding is defined when you create the index. Once done, the only way to change the number of shards is to delete your indices, create them again, and reindex. I've written a comprehensive post about[resizing your Elasticsearch clusters in production](https://thoughts.t37.net/resizing-your-elasticsearch-indexes-in-production-d7a0402d137e)which might help.

Choosing the right number of shards is complicated because you never know how many documents you'll get before you start. When I know more or less the future size of an index, I do the following:

* less 3M documents: 1 shard
* between 3M and 5M documents with an expected growth over 5M: 2 shards.
* More than 5M: int \(number of expected documents / 5M +1\)

Having lots of shards can be both good and terrible for a cluster. Indices and shards management can overload the master node, which might become unresponsive, leading to some strange and nasty behaviour. Allocate your master nodes enough resources to cope with the cluster size. I try not to run more than 10.000 open indices and 50.000 primary shards on the same cluster.

If you plan to use[Elasticsearch routing](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-routing-field.html), don't create more shards than you need, otherwise you'll end up having lots of empty shards that just add useless pressure on the data and master nodes.

On the other hand, having lots of shards on huge indices is good too, specially when you have a large cluster \(20 data nodes and more\).

* Multiple shards allow a better allocation between nodes.
* Small shards on multiple nodes make the cluster recovery much faster when you lose a data node or shutdown the cluster.
* Spreading smaller shards on lots of nodes might solve your memory management problems when running queries on a large data set.
* Large shards makes indices optimization harder, specially when you run _force\_merge _with _max\_num\_segments=1 _since you need twice the shard size in free space.

There's one more thing about sharding. Lucene, the search engine that powers Elasticsearch, creates many files to manage parallel indexing on the same shard. These files are called segments, and depending on your indexing workload, Lucene can create and open thousands of them at the same time. That's why it's important to run Elasticsearch with\_max\_open\_files\_at 32.000, if not more.

#### Replication {#e70b}

Elasticsearch has a built in replication system. Data is replicated amongst the data nodes so losing one of them won't mean a data loss. Elasticsearch default replication factor is 1, but it might be interesting to have a higher replication factor.

Before you start playing with replication, you might want to understand Elasticsearch replication consistency formula:

> int\( \(primary + number\_of\_replicas\) / 2 \) + 1

Going beyond the factor 1 can be extremely useful when you have a small dataset and a huge amount of queries. By allocating the whole data set to every node, you can leverage the search thread pools to run much more queries in parallel. I once went to a replication factor of 51, on 52 nodes, with the whole dataset in memory.

#### Optimising allocation {#ee92}

Once you use rack awareness, it might be interesting to optimise shard allocations using[elasticsearch zones](https://www.elastic.co/guide/en/elasticsearch/reference/current/allocation-awareness.html).

For example, if you have indices that are accessed more frequently than others, you might want to allocate more data nodes to those indices while the less frequently accessed indices have less resources allocated. This is extremely efficient with timestamped indices.

Let's say you have 20 data nodes, and 30 indices, you can create 3 zones. Allocate your 30 nodes to these zones according to the needed resources:

* new: 15 nodes
* general: 10 nodes
* old: 5 nodes

Every day, run a crontab to reallocate your indices to their new zone. For example, move a less accessed index into the "general" zone:

```
PUT /index/_settings 
{
    "transient": {
        "cluster.routing.allocation.include._zone" : "general",
         "cluster.routing.allocation.exclude._zone" : "new,old" 
    }
}
```



