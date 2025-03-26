# Here I'll share some stories and experiences 

## Elasticsearch

At a previous job for a b2b ecommerce shop we built out a product search page. Something not that dissimilar from a home depot search screen: [homedepot screenshot](https://www.homedepot.com/b/Outdoors-Outdoor-Power-Equipment-Trimmers-Edgers-String-Trimmers/N-5yc1vZbx8i)

If your not so familiar with Elasticsearch a brief overview is it's a document based db, where documents are json blobs, and rather than tables or colllections they are stored in 'indexes'. 

Rather than hosting our own Elasticsearch cluster (which we had on-prem for logging, but was not well maintained), and since we were migrating to AWS, we checked out AWS OpenSearch (previously AWS Elasticsearch before the AWS v Elastic drama).

Similar to the home depot product search screen where you have a search box, product tiles with images, prices, descriptions etc facets and filters etc, our was the same. Each 'index' in our elasticsearch cluster (i'll get to it being a cluster soon) was based on a company whose products we supported. Think Dewalt(maybe has trimmers/ drills etc, whole long list of SKUs). We had a service where users would upload their file of products blah blah -> eventually it gets into Elasticsearch so we could use the lucene index to do some full text searching. 

Everything's great right? And it was, honestly for a while. 

I'd love to say we had scale issues, and in a sense we did, but not because business was booming or because we had customers like crazy, we just made a mistake instead. 

We started out small with a elastic search cluster on aws that was maybe 1 node(hardly a cluster), then more users uploaded files, we needed more storage, so we scaled up (larger machines). We wanted to follow all of the best practices, had 3 master nodes ( want to have an odd number of master nodes to avoid split brain problem for leader election), and 1-3 nodes. 

AWS Elasticsearch clusters follow the common pattern in distributed systems where your writes go to primary (in our case the master nodes) and then are replicated to the other nodes. Meanwhile our reads are pointed to the read replicas (man we were so hyped about CQRS at that time). 

All is fine and dandy, but we start to see issues occuring in our cluster, we have an async process that does the writes, and it's not able to do the write (create a new index for that new brand) because our node is limited in the total number of indexes it can have! AWS Elasticsearch had a limit of 1000 indexes per node. Ah no problem we'll just add another node, and we'll be fine, continue along. 

Now enter stage left our two other friends sharding and replication. Sharding- breaking up the data across multiple machines and replication - creating copies of the data, help immensely in databases for scalability and distributing load and work across machines. 

In Elasticsearch dat ais stored in the index, just like a db table, and each index is divided into multiple shards for distribution across nodes. Each shard can have replicas for redundancy. 

AWS Elastichsearch helps you out that for each primary shard you have at least one replica, how helpful! 

Cluster Architecture Diagram (ASCII Representation):
```sh
  +----------------------------+
    |    AWS OpenSearch Cluster  |
    +----------------------------+
          |        |        |
   ----------------------------------
   |          |          |          |
+--------+  +--------+  +--------+  +--------+
| Node 1 |  | Node 2 |  | Node 3 |  | Node 4 |
+--------+  +--------+  +--------+  +--------+
    |           |           |           
+--------+  +--------+  +--------+
| Shard 1 |  | Shard 2 |  | Shard 3 |  ... (Primary Shards)
+--------+  +--------+  +--------+
| Replica 1|  | Replica 2|  | Replica 3|
+--------+  +--------+  +--------+

```

How do queries work though? User searches for 'cordless drill', query is routed to relevant shards. The request is load balanced across both primary and replci shards. Each shard search within it's dataset (nice parallel processing), and results form different shards are combined, weighted and ranked and the aggregated results are returned to the client, then to the user. 

A little more on our indexing, as each brand "Dewalt" uploaded a file we created a specific index for them. company_uuid kept the indexes unique and each company only had one index(their whole dataset of products). We did a full replace of the index each time they uploaded, creating a temp index, and then swapping it over to live once all of the products/documents/jsons were populated. 

These indexes on average were quite small, each well under 1GB. This was great, it meant we store tons of data, for cheap. Well it turns out there's a downside to managing a large number of small indexes. Their associated shards.

Elasticsearch recommends aiming for shard sizes between 10GB and 50GB to optimize performance. When have many small shards each shard regardless of size consumer memry and file handles, and when manging thousands of small shards this can strain a cluster. Oh boy did we start to feel the pain. 

We scaled up, we scaled out. We had a huge 32GB high memory machine for each 3 node master, 5 node cluster. And what was the data? oh just a few hundred thousand product documents. Only a couple of gigs of actual data in total, surely under 50, likely under 10 in total. 

#### What had happened? 

well as we kept adding more brands and hence creating new indexes, thus creating new shards. and not just one shard, but multiple as AWS creates that read replica shard for you. 

As our data grew, the unintended side effect was a rapid increase in memory usage. Why did we have high memory machines? well we were able to mitigate the issue because " A common best practice is to aim for about 20 shards per GB of heap memory per node." so we could manage for a little. 

AWS domains typically have a default cluster-level limit of around 2000 shards per domain 

At this point we were runnign into issues where our cluster was full, we couldnt keep adding nodes(more cost), and it was being manually scaled. We had been deleting indexes manually to free up space, but users file uploads weren't getting into the cluster anymore. 

How did we fix it?

well when AWS and Elasticsearch tell you that you should plan based on your data volumne for your indexing strategy you should listen. for our data size, we could fit everything in one index. Brief example table

Example: Scaling Based on Data Volume
Data Size	Suggested Shards	Suggested Replicas
< 10GB	1-3 shards	1 replica
10GB-100GB	5-10 shards	1-2 replicas
> 100GB	10+ shards	2+ replicas

we combined all of teh data into one index that held everything, used a new field for brand/company_uuid and the day was saved. 

take aways:

Small Index Scenario:
If you’re storing many very small indexes (each under 1 GB), each index might still create its own primary shard (and associated replica) even though the data volume is low. In these cases:

The overhead of managing many small shards can add up.

Consolidating small datasets into fewer indexes (with multiple types or a distinguishing field) can help reduce the total shard count.

Performance Consideration:
Even if the raw data size is low, having too many shards (and by extension, too many indexes) can impact cluster performance, slow down cluster state updates, and increase memory usage. Following the guideline of “20 shards per GB of heap” ensures that your nodes aren’t overloaded.

big shout out to the AWS engineers who I met with this, in the end it was bad design on our part, but they were able to try and troubleshoot and assist. In the middle of this slowly creepign issue we did a major version upgrade as well, that failed halfway through due to our cluster being in such a degraded state, but we ended up making it out the other side!