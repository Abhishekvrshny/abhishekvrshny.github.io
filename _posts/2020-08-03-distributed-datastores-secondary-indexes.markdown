---
layout: post
title: Distributed Datastores and Secondary Indexes
date: 2021-01-09 00:00:00
description: This post talks about various strategies that distributed datastores use to provide the capabilty of secondary indexes.
absolute_image: http://pld.cs.luc.edu/database/images/mysql_index.png
---

#### Databases and Indexing
All sorts of databases or datastores provide some or the other king of indexing for the data they hold. Indexes provide a way, given a value for the indexed field, to find the record or records with that field value. A primary index is a clustered index on the primary key of a sorted file. Any index other than a clustered index is called a secondary index. Secondary indices does not impact physical storage locations unlike primary indexes. Secondary indexes are needed as you might want to query the database with field values which are not the primary key.

The concepts around database indexing in realtional databases like MySQL and Postgresql are well documented and understood. In this post, I would talk about secondary indexes patterns in distributed datastores and the considerations while moving from a realtional to a distributed datastore.

#### Secondary Indexes in Distributed Databases
Secondary indexes in NoSQL/distributed datastores is a challenge and always comes at a cost (performance). This is because sharding happens by primary index which is easier to lookup on a shard. Secondary indexing requires global view which is not easy. Different datastores handle secondary indexes differently.

1. **Local secondary indexes**: Cassandra, MongoDB primarily support local secondary indexes. Global secondary indexing is just map/reduce operations after lookups from these different local secondary indexes and comes with restrictions and performance implications.

2. **Global indexes**: TiDB (based on spanner paper) supports global indexes, but the implementation is such, that secondary indexes are again key value pairs, no different than how actual data is stored. Again, the index here will not really be performant.

3. **No secondary indexes**: This is simplest. HBase, for eg, doesn't support secondary indexes. Implementors are free to choose any secondary indexing strategy they want. For eg, some do it by following approach 2, some use external systems like elastic search to do it. eg apache phoenix.

These restrictions are again because, these distributed systems assume that data is truly distributed. One way to do efficient secondary indexing while also distributing data is application level sharding (on mysql/mongo/postgres etc), but that has own challenges in terms of moving data between shards when one bucket overflows etc. 