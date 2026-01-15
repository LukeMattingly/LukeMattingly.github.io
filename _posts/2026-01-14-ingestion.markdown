---
layout: post
title:  "Building Ingestion for Vertex AI Discovery Engine"
date:   2026-01-14 20:09:29 -0600
categories: work
---


# **Building Ingestion for Vertex AI Discovery Engine**



## **What are we trying to do**


I've setup a Vertex AI Search Datastore , previously known as Discovery Engine. 

It's essentially a managed hyrbid vector store, which is extremely helpful is someways, and very limiting in others. 

The challenge is to pump 5.8million docs (each only a few MBs) from Mongo our primary datastore into Discovery Engine to populate it with data for an intial bulk load. 


## **What's important to consider when doing this**


VertexAI offers a few ways to go about doing such a large import, but the main is an `importDocuments` api they expose where you can either upload files to GCS or provide them in BigQuery. 

I'm going with Google's cloud storage, as it's simpler to setup, and push data into withotu requiring a bunch of permissions/setup overhead. 

What are the limiting factors?

VertexAI has a limit of:
- 100 documents maximum that you can upload
- 2 GB file size max
- docs must be in a predefined file type (.jsonl) for these uploads

importDocuments api works simply that you give it a GCS directory and a reconcillationMode(FULL or INCREMENTAL) and it will try to pull in all the files. 

FULL - does a full replace of everything in your store
INCREMENTAL - only updates the documents you have provided (with the ability to provide an `update_mask` if you only want a specific field updated)

My 5.8million docs turn into about 87GB of data uncompressed so we're under this total 100files * 2GB = 200GB total max data size threshold. 

I'll also note that Vertex has a limit of 10million documents total you can keep in your datastore. 

## **What's the process going to look like**

From a high level we have few concrete steps
- Read the data from Mongo
- Transform it into our Java objects that represent what we're storing
- Distribute the work 
- Upload in data in either 100 files, 2GB files or a combination to GCS
- Call the import docs api once the uploads complete

Seems simple enough, but let's dive into the fun. 

## **What's hard/ what's interesting/ what are the challenges**

We want this to be highly performant, reusable, and highly automated system. 

I've gone through the above steps end-to-end on my laptop(mpb m4), and while do-able, were painfully manual, error prone, extremely slow, and very disk space consuming. but do-able. 

First we need to figure out how we're going to distribute this work. Fortunately we're in GCP and we have a streaming pipeline I also setup pushing data into VertexAI already, using Apache Beam and GCP Dataflow.

So we can leverage Apache Beam's batch job with Dataflow to run our code, and distribute the work to multiple works dynamically. This will be the biggest help is managing lots of reads, and lots of writes. 


## **What do we need to do to read performantly from Mongo**

To start you should always be using the built in BeamIO connectors, in this Mongo provides one and we'll leverage that here

Next up when reading from Mongo in this case we need to partition the incoming collection that's 5.8m docs, we can use `wtihNumSplits` provided to do this. Which disjointly partitions the incoming data and allows the reads to be split more easily. 

How many splits should we provide? 1, 100, 1000? Does it need to be dynamic based on the number of works, or based on our data size?

Our goal is to optimize for parrallism, and performance, so  too small and it doesn't help distribute the load and too large and we're spinning up too many workers rather than reading the data. 

We need to allow each Dataflow worker node to read 

> TODO show how we calculated the 900 number for read splits


## **Transforming**

Key observation here is to not use new Gson() operation on a chain in Beam, as it creates a new object for each new object that needs mapping
This is very inefficient, and instead we want to re-use one. 

Something about serialziation and not serializing things that can't be serialized. 


##  **Partitioning the data**

- How do we split up the data?
- Should we do 100 files at ~900MBs each or 46 files at 1.9GB each?

My initial intuition was fewer files, fewer uploads, simpler, faster. Maybe this logic made sense more when I was doing this locally, but when I have to scale out to many machines this was not a correct assumption. 

We know we're going to have 100 files, so we know we're going to much fewer than 100 Dataflow worker nodes, as an individual machine (n1-standard-n2) has 2vCPUs and X RAM. It should be able to store, and process many of these in parallel. 

Initially I thought we'd need to write each file as {jobId}-{date}-{part_}.jsonl as `1234-2026-1-14-part_001.jsonl` to keep the files unique, but this level of strict ordering wasn't required, and was only for human readability, and could only cause contention for fetching file names. All additional overhead I was looking to avoid. 

Since we'll have to buffer these files in memory and stream them, we want them to be as small as possible. Thus we need to maximum our number of files axis, to 100. 

Okay great, so we know we're going to have 100 files, but how much data should each file have, surely if I allow each file to fill itselt up to 1.9 GB, I'm only going to get 46, so I need to cap the size of individual files. 

If I cap the file size at 900MB then if a file comes in a 901MB that 900MB will be in 1 file and 1MB will be in another, potentially either pushing me over 100(which is a limit by Vertex) or losing out on that data, which isn't acceptable. 

How do we actually do this in Beam?

`BatchWithGroups` helps us out a lot here

but we need to know the number of shards, so we can create a partition key for our data. 

When we have new json objects come in they need to be partitioning to know which file they are going to. and only route new json objects to partitions that aren't full, or won't push them over their file size limits. 

The system we build needs to be flexible, if our data increases by 1 million documents up to 6.8 million, this job keeps humming away carefree. Same with if the data decreases in size likewise. We don't want hardcoded values except for the limitations that have been placed upon us. Those only currently being 100 files max, and 2GB file size max. 

Once each bucket fills up, we push it to GCS, and flush anything in memory. so we have a clean slate for the next batch of data for the next fike

## **Writing to GCS and waiting**

Writing to GCS is an async process, and since these files aren't small they will take some time.

GCS's client is known for commonly providing errors so you have to wrap it yourself with exponential backoff & retry, error handling etc, for uploads that don't succeed, complete or connection issues. Even using the official GCP SDK! kind of ridiculous but I digress. 

 I don't want to block a thread waiting for it to complete, so I can push the 'waiting for gcs upload to complete' to another beam stage

This next stage checks in on the uploads and won't complete the beam job until all uploads are complete. Once that's done, our Beam journey is done and the next stage to kick off

## ** Post Beam World** 

Behind the scenes there's another api that fired off this Beam Job and has been monitoring it for completition. This api is async and long lived as it's job is to handle the next stage which is to fire off that glorious `importDocuments` api. Which could've been done by Beam, but seemed unneccessary for 5-10 distributed workers to handle 1 api call. Especially when I'll need to poll this endpoint 

=====================

