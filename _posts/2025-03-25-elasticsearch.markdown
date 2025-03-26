---
layout: post
title:  "Lessons from Scaling Elasticsearch the Wrong Way"
date:   2025-03-25 20:09:29 -0600
categories: work
---

# **Lessons from Scaling Elasticsearch the Wrong Way**

## **Introduction**

We were building a B2B e-commerce platform where brands could list their products and businesses could search for what they needed. Think of a typical **product search page**—users enter keywords, apply filters (categories, price ranges, availability), and expect near-instant results.

Elasticsearch seemed like the obvious choice. It offered full-text search, fast indexing, and scalability—everything we needed. Or so we thought.

We made what seemed like logical scaling decisions, but instead of improving performance, we introduced inefficiencies that nearly crippled our system. Here’s what went wrong and how we ultimately fixed it.

---

## **Understanding Elasticsearch and Our Initial Design**

Elasticsearch is a document-based search engine built on **Apache Lucene**. Instead of relational tables or collections, it organizes data into **indexes**, where each index contains **JSON documents**.

For our system, every product was stored as a document, with fields like:

```json
{
  "product_id": 12345,
  "name": "Cordless Drill",
  "category": "Electronics",
  "price": 69.99,
  "stock": 50,
  "description": "An electric drill which uses rechargeable batteries"
}
```

### **How Search Worked**

Users could enter queries like **“cordless drill”**, and Elasticsearch would match products based on:

- **Full-text search**: Analyzing the `name` and `description` fields.
- **Filters**: Category, price range, availability, and more.
- **Ranking**: Using relevance scoring to surface the best results first.

This was the core functionality of our platform—**fast, relevant product discovery**. And as more brands joined, we had to scale. That’s where we made our biggest mistakes.

---

## **Scaling Decisions That Backfired**

To isolate brand data, we decided to create **a separate Elasticsearch index for each brand**. The reasoning was simple:

- Easy data management per brand.
- Faster reindexing when a brand updated its catalog.
- Simple permissioning—each brand’s data lived in its own space.

At first, this worked fine. But as the number of brands grew, we hit serious problems.

### **Problem 1: Index Explosion**

Elasticsearch allows multiple **shards** per index, where each shard is a separate Lucene instance. By default, AWS OpenSearch (our hosted solution) created **five primary shards per index**, with at least **one replica per shard**.

With **hundreds of brands**, this meant **thousands of shards**, each consuming memory—even for small brands with just a few dozen products. Our cluster was managing **shards, not data**, and we hadn’t even hit significant traffic yet.

### **Problem 2: Hard Limits and Resource Waste**

AWS OpenSearch enforces **hard limits**:

- **1,000 indexes per node** (we hit this quickly).
- **Total shard limits (~2000 per domain)** (we exceeded this too).

Every brand’s data was relatively small (often < 1GB), but since each index had multiple shards, we wasted resources on managing thousands of unnecessary Lucene instances.

### **Problem 3: Query Performance Degradation**

Initially, searches were fast. But as indexes multiplied, queries that needed to scan multiple brands (e.g., “cordless drill” across all suppliers) suffered:

- **High coordination overhead**: Queries had to touch multiple indexes.
- **Memory bottlenecks**: Each shard consumed resources, even if it stored minimal data.
- **Inconsistent performance**: Some queries were near-instant, while others lagged significantly.

When we hit these issues, our first instinct was to **add more nodes**. This delayed the inevitable but didn't fix the underlying problem.

---

## **The Breaking Point**

At peak misconfiguration, our setup looked like this:

- **5 Elasticsearch nodes** (high-memory, high-cost)
- **3 dedicated master nodes** (to prevent split-brain issues)
- **Thousands of shards** (most underutilized)
- **Degraded query performance** (especially for cross-brand searches)
- **Uploads failing due to shard limits**
- **An attempted version upgrade that nearly took the cluster offline**

Scaling by adding nodes was no longer an option—we needed a fundamental redesign.

---

## **How We Fixed It**

Instead of creating an index per brand, we moved to **a single shared index** for all brands, with a `brand_id` field to distinguish data.

### **Optimized Indexing Strategy**

- **Single index, fewer shards**: We allocated shards dynamically based on total data size, not arbitrary brand counts.
- **Filtered queries**: Instead of searching multiple indexes, we used `brand_id` filters within a consolidated index.
- **Better shard sizing**: Elasticsearch recommends keeping shard sizes between **10GB-50GB**. We adjusted accordingly.

#### **Before vs. After**

| **Metric**          | **Before** (Bad)    | **After** (Optimized) |
|--------------------|------------------|------------------|
| **Indexes**        | Hundreds (1 per brand) | 1 shared index |
| **Shards**         | Thousands | ~10-50 total |
| **Memory Usage**   | High | Lower |
| **Query Speed**    | Slower (many indexes) | Faster (fewer shards) |
| **Maintenance**    | High overhead | Minimal |

🚀 **Results:**

✅ **Memory usage dropped significantly**  
✅ **Query performance improved dramatically**  
✅ **Cluster stability restored**  
✅ **Lower AWS costs**  
✅ **No more manual index deletions**  

---

## **Key Takeaways for Scaling Elasticsearch**

- **Avoid excessive indexes**: Fewer, larger indexes reduce overhead.
- **Shard wisely**: More shards ≠ better performance. Plan based on data size.
- **Understand hosting limitations**: AWS OpenSearch has hard index/shard limits.
- **Scale efficiently**: More nodes don’t fix inefficient design.
- **Test scaling early**: Don’t wait until you’re hitting system limits.

In the end, Elasticsearch wasn’t the problem—**our scaling strategy was**. We assumed that more indexes meant better isolation, but in reality, it just fragmented our data and overloaded our cluster.

By consolidating our index structure, we **reduced complexity, improved performance, and saved money**—all without adding unnecessary infrastructure.

### **Final Thought**

Scaling Elasticsearch isn’t just about adding more nodes—it’s about designing data structures that make sense for your workload. If you’re planning to scale, **think beyond storage and CPU—shard management and query efficiency matter just as much.**

