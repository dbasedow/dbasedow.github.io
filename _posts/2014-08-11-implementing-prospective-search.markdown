---
layout: post
title:  "Implementing prospective search"
date:   2014-08-11 13:53:07
---
The last couple of weeks I have been busy working on [Prospecter](https://github.com/dbasedow/prospecter), an open source
prospective search implementation. Prospective search switches the role of documents and queries compared to regular search.
Queries get indexed and documents get matched against the query index as they arrive. A well known example of this is Google
Alerts, where you get notified when new pages matching your query are discovered.

Some implementations, for example ElasticSearch's percolator feature, offer prospective search capabilities. But they do
this by creating an in-memory index of the document and running all saved queries against that index. With Lucene's memory
index this is very fast. However, it does not scale as well as it could.

So far progress has been really good. Full text queries return the same results as ElasticSearch when using the same
Analyzer. Just much faster. I'm using the Excite search log to test the full text implementation, it contains 944,940
queries. Prospecter is able to find all matching queries in under 70 ms, for a news article containing 909 words. A
random tweet from the CNN feed containing 17 words took under 25 ms on average.

The numbers are from benchmarks run on my 2013 MacBook Pro (i5 2.6 GHz, 8GB RAM). It's also important to note, that no
real performance tuning has been applied to the code. There are still some obvious and simple improvements to be applied.
