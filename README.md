# Designing the Perfect Elasticsearch Cluster: the \(almost\) Definitive Guide {#ed1d}

Ever since I started to write about [operating Elasticsearch](https://thoughts.t37.net/tagged/elasticsearch), I have answered many questions about cluster design. Every one of them we asked by people who were already running Elasticsearch and ran into trouble after the first users came.

Cluster design is an overlooked part of running Elasticsearch. Offical documentation and blog posts focus on the magic of deploying a cluster in a giffy, while the first problem people face when deploying in production is memory management issues, aka garbage collection madness. This guide answers most questions I was asked, and summarizes everything you should know about designing the perfect Elasticsearch cluster.

That's the moment I'm supposed to stop writing and leave you with a blank page.

> There’s no way anyone can tell you how to design a perfect cluster, and those who claim they do are liars.

What you'll find here is a list of things you need to know and understand to design the cluster that fits your needs. And that's already a lot.

Before we get into the details, there's one thing you need to understand.

> Unless you plan to use Elasticsearch to power the search of your Wordpress blog or small e-commerce website, your first design will suck. Always.

You can't design a cluster without knowing what your workload will be, how much search and writes per second you'll get, how fast your indexes will grow, and what kind of queries your users will run, against what type of content. And you can't know about your workload unless you've ran your cluster in production for a while. You'll most probably have to iterate 2 or 3 times before you get the design right. I've been running Elasticsearch in production since mid 2011 and I never got it right the first time.

