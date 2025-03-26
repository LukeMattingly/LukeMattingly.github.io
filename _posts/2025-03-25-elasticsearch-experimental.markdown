---
layout: post
title:  "Ignore Testing other formats"
date:   2025-03-25 20:09:29 -0600
categories: databases, work, sharding, replication
---

# Lessons from Scaling Elasticsearch the Wrong Way 🚨

## Introduction
Picture this: You're building a B2B e-commerce platform (think Shopify, but smaller and simpler), and you need a solid search function for all those products. Enter **Elasticsearch**—fast, powerful, and… deceptively easy to scale **the wrong way**.

I was a backend engineer on a small team of three, and we thought we were following best practices. But instead of scaling for success, we scaled straight into **self-inflicted chaos**.

This is the story of how we turned our Elasticsearch cluster into a ticking time bomb—and what we learned when it finally exploded.

## What is Elasticsearch? (A Crash Course)
For the uninitiated, Elasticsearch is a document-based database optimized for full-text search, backed by Apache Lucene. Instead of tables (like in MySQL) or collections (like MongoDB), it uses indexes, and inside those indexes, you store JSON documents.

Example document:

json
Copy
Edit
{
  "product_id": 12345,
  "name": "Cordless Drill",
  "category": "Electronics",
  "price": 69.99,
  "stock": 50,
  "description": "An electric drill which uses rechargeable batteries"
}
Our setup? We hosted each brand’s products in its own Elasticsearch index, meaning every company got its own index.

At first, it seemed like a great idea. But little did we know, we were on a one-way path to disaster.

Scaling Up… And Digging Our Own Grave
We started with a single-node Elasticsearch cluster on AWS OpenSearch (formerly AWS Elasticsearch). But as more brands uploaded products, we needed more storage.

So, we followed the textbook scale-up approach:

Bigger machines ✅

Three master nodes (to avoid split-brain issues) ✅

Read replicas for faster queries ✅

Primary → Replica writes (CQRS hype!) ✅

Everything was great—until it wasn’t.

Then We Hit a Hard Limit…
One day, our async process failed to write new indexes. Why?

👉 AWS OpenSearch has a limit of 1,000 indexes per node.

"No problem," we thought. "We'll just add another node." (Famous last words.)

Enter: Sharding & Replication—Our Unintentional Villains
Elasticsearch distributes data using shards (splitting data across nodes) and replicas (backup copies for redundancy).

Each index is made up of multiple shards.

AWS automatically creates at least one replica per primary shard.

So, every time we added a new brand:
✅ New index → ✅ New primary shard → ✅ New replica shard → ❌ More memory & file handle overhead

Multiply that across hundreds of brands… and our cluster started suffocating.

How Bad Did It Get?
We kept adding nodes—but instead of solving the problem, we were just delaying the inevitable.

Cluster ballooned to 🚀 3 master nodes + 5 high-memory nodes

Total product data? A few GBs at most

Total shards? THOUSANDS (each eating memory & resources)

Query performance? Degrading fast 💀

Uploads failing? Yep 😵

Elasticsearch recommends shard sizes between 10GB - 50GB.
👉 Ours? Mostly under 1GB each.

We had created too many tiny shards—and it was killing our cluster.

The Breaking Point
At our peak cluster madness:

AWS OpenSearch had a hard shard limit (~2000 shards per domain).

Our memory was maxed out trying to manage thousands of small indexes.

Deleting old indexes manually just to keep things running.

A failed Elasticsearch version upgrade left our cluster in a degraded state.

At this point, we had two choices:

🛑 Keep scaling (and burn $$$) OR
✅ Fix our indexing strategy once and for all.

How We Fixed It (And What You Should Do Instead)
Instead of one index per brand, we switched to a single consolidated index with a field for brand_id.

Scaling the Right Way: How Many Shards Do You Actually Need?
Data Size	Suggested Shards	Suggested Replicas
< 10GB	1-3 shards	1 replica
10GB-100GB	5-10 shards	1-2 replicas
> 100GB	10+ shards	2+ replicas
🚀 Result?

✅ Memory usage dropped drastically
✅ Cluster performance improved instantly
✅ New uploads stopped failing
✅ No more manual index deletions
✅ We saved a ton of $$$ on AWS costs

Key Takeaways for Scaling Elasticsearch Properly
🔹 Avoid too many small indexes → Consolidate when possible.
🔹 Shards consume memory, even if they’re tiny → Fewer, larger shards are better.
🔹 Plan your indexing strategy based on data size → Not arbitrary business rules.
🔹 Read the AWS OpenSearch fine print → Hard limits exist, and they hurt.
🔹 Scaling out ≠ Scaling right → More nodes don’t always fix bad design.

Big shoutout to AWS engineers who helped us troubleshoot this mess. (Yes, we even managed to break things during an upgrade. 😅)

Final Thoughts
Was this Elasticsearch’s fault? Nope.
Was this AWS OpenSearch’s fault? Also nope.
Was this our fault for designing it poorly? 100%.

Lesson learned: Scaling Elasticsearch is easy. Scaling it correctly is hard.

So, the next time you're designing an Elasticsearch cluster—think before you shard. 😉