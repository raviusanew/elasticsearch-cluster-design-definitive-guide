# Elasticsearch is elastic, for real

There are 2 basic concepts you need to understand about Elasticsearch.

The first one is: "you know, for search". This is the response you get when you run an empty query on an Elasticsearch cluster, and that's for a reason.

> Elasticsearch is a search engine. Elasticsearch is not a datastore and it won't replace MySQL.

If you're looking for a distributed data store, close your tab, you've hit the wrong place.

There's another basic concept that's often poorly understood.

> Elasticsearch is elastic, for real.

There's 2 things about elasticity when you design your cluster.

The first one is horizontal scaling. You can build a cluster with virtually an infinity of hosts, depending on your needs and the bottleneck you face. Sometimes, running your dataset on a lot of small machines will provide bette performances than using a few large hosts. Sometimes, running on medium hosts with lots of disk is better. And sometimes, you'll need gonzo CPU but storage and memory won't be a problem.

The other one is index sharding. Elasticsearch divides indexes in physical spaces called shards. They allow you to easily split the data between hosts, but there's a drawback as the number of shards is defined at index creation. Elasticsearch default is 5 shards per index, but only your workload will help you to define the right number of shards. Thankfully, there's[a way to scale existing indexes in production using reindexing and index aliases](https://thoughts.t37.net/resizing-your-elasticsearch-indexes-in-production-d7a0402d137e).

