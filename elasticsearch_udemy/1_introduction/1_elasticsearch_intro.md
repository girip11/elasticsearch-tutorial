# Introduction

- Elasticsearch is an **analytics** and full text **search engine**.
- Used in application monitoring and also as analytics platform when used along with Kibana, logstash which is called as the elastic stack.

## ES overview

- In ES, data is stored as documents. Each document is similar to a row in a relational DB.
- Document contains fields (columns in relational DB)
- A document insert/update is using JSON.
- Queries are also written in JSON.
- ES written in Java. Scales very well.

## Elastic stack

- Elasticsearch - analytics and text search engine.
- Kibana - analytics and visualization platform. Can create dashboards. Provides some management interface as well.
- Logstash - Process logs from applications and send to ES. It's a data processing pipeline. It consists of input source(read from file, kafka), filters(filter and data enrichment) and output(destination to send the processed data). We have various plugins available for input and output.
- X-pack - Access control(RBAC), monitoring elastic stack(cpu, memory, disk), alerting for both elastic stack infra as well as based on the logs stored in elastic search, report generating, enables kibana to do the machine learning, graph(related data - product recommendations), elasticsearch sql.
- Beats - Collect data and send data to logstash or directly to ES. Beats has plugins(called beats like FileBeat, HEARTBEAT) that help read data from different logs(nginx,apache webserver)

**NOTE**- Its recommended to use elasticsearch query DSL rather than using SQL.

## Elastic search configuration

`elasticsearch.yml` is the configuration file. Some of the key properties are

- `cluster.name`
- `node.name`
- `path.data` and `path.logs` - directories for storing data and logs
- `network.host` - IP to bind to
- `http.port` - Default is 9200
- `discovery.seed_hosts`
- `clustern.initial_master-nodes`

Elastic search in written in Java. So it runs on JVM. We might need to configure the heap size for the JVM.

Modules are shipped with ES, while plugins can be added to ES. Plugins can be removed while modules cannot. `ES_HOME` is environment variable that points to elastic search root directory.

## Architecture

- A node is an elastic search server. In production, a node runs on a separate machine/vm/container.
- Each node belongs to a cluster. Clusters are independent of each other. A cluster can have many nodes. We could have one cluster for product recommendation, while we can have another cluster that is used for application monitoring. We can run cross cluster queries (but that is rare).
- Each node stores a part of the data.
- A node can join an existing cluster, or it can create one and join it(single node cluster)

- Each record is a document which is a JSON object along with some metadata(indicated by fields starting with `_`).

- **Index** is a collection of similar and logically related documents. It's like a table in relational db. Search queries are run against indices to fetch documents.

## Inspecting the Cluster

- Each query that is made to elasticsearch is through a REST api.
- Thus each query in Kibana will be associated with HTTP verb(GET, POST, PUT, DELETE)

Some basic queries that we can tryout in Kibana developer tools (kibana converts the query in the below format to HTTP request)

- `GET _cluster/health` - Cluster health
- `GET _cat/nodes?v` - List the nodes in the ES cluster. [cat nodes api](https://www.elastic.co/guide/en/elasticsearch/reference/current/cat-nodes.html) and [Nodes API](https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster-nodes-info.html)
- `GET _cat/indices` - Lists all the indices in the cluster

**NOTE**: Kibana persists its data into elastic search. So when we connect to ES from any kibana interface, we could get the kibana config from the ES cluster(index name starts with `.kibana`).

- We can also send search requests to elastic search using CURL, postman etc.

## Sharding and scalability

- Sharding - Divide index in to multiple shards. Sharding is done at index level. Helps horizontally scale.
- A shard of an index can be placed on any node in the cluster. Each shard functions independently.
- Each shard is a apache lucene index.
- A shard does not have a limit in terms of size, but in terms of the number of documents it can store. A shard can only store up to 2 billion documents.
- Sharding allows the index to scale horizontally(ex: Storing index containing documents of size 1TB, where each node in the cluster has only 250GB disk space.)
- Sharding also improves performance, search query can run in parallel on multiple shards.
- An index contains a single shards by default (from version 7). To increase the number of shards, a new index is created and the documents are copied over from old to new index(split api).
- Shrink api - shrinks the shards.

NOTE: Adding additional shards at the beginning is good practice, if we think the data would keep growing. For storing millions of documents, 5 shards is a good default.

## Understanding replication

Replication serves 2 purposes 1. fault tolerance 2. Improve throughput

- A shard contains a part of the data in an Index. A shard is stored in one node by default. If the node fails, the shard's data for that index will be lost/unavailable.
- So shard needs to be replicated for fault tolerance. Replication of shards are enabling by default in ES.
- Replication configured at index level. Default is 1.
- Replica shards - replicated copies of shards.
- When a shard has many replicas, then one of it is designated as a primary shard and others as secondary shards. All the replicas of a shard form a replication group. Number of replication groups = number of shards
- Replica shards are always stored in different nodes than the primary shards.
- Replication is added only on clusters with more than 1 node.
- Write requests go through only the primary shard. While replica shards sync with the primary.
- Replica shards of the same replication group can serve different search requests.

- `GET _cat/shards?v` - gives information about shards of indices.

## Snapshots

- Backups for restoring at later point in time.
- Index level backups or entire cluster possible
- Bad deployment corrupts the index documents etc, we would like to restore the index to an older state, then snapshots are very useful.

## Adding more nodes to the cluster(development)

- In local development using docker-compose, we can add more copies of elastic search under `services`.

## Node roles

### Master-Eligible

- master-eligible - Node with this role can become cluster's master node. Many nodes can have this role and one of the node will be elected as the master. Master nodes perform operations like creating, deleting indices, shard allocation etc.
- For huge clusters, dedicated masters are required because of the resources that it requires. If master hosts shards, then it may spend CPU on serving requests.

### Data

- Store part of the cluster's data
- Serves queries on the stored data.
- When configuring dedicated master roles, those nodes should not have the data role.
- Dedicate data node - Node with only data role

### Ingest

- Ingest pipeline - series of steps in ingesting a document ES(adding a document)
- Preprocess document before ingesting in to ES like IP to lat/long
- This is a simplified version of logstash, used for simple data transformations. When ingest operations are complex, logstash usage is recommended.
- When having a logstash, disable this role, otherwise go with dedicated ingest nodes, so that the read and write throughput are good.

### Machine Learning

- `node.ml` flag - Set to true to make the node a machine learning node. Runs ML jobs.
- `xpack.ml.enabled` flag - enables ML API for the node.
- Since ML jobs are resource consuming, when a lot of ML work is required, it is recommended to have dedicated nodes with this role.

### Coordination

- This node handles the request and performs the delegation - Distribute queries internally to data nodes.
- No explicit role available for such coordination. Coordination is achieved by disabling all other roles.
- Useful in very large clusters

### Voting-Only

- Node with this role will participate in master election.
- It will participate only in voting and cannot be elected.

---

`GET _cat/nodes?v` - Returns node information. We can look at the node's rule from the column `node.role`. A node with role value `dim` means that the node is a data, ingest and master eligible node.

### When to change the node roles

- This is often done in large clusters.
- We observer the nodes with many roles `dim`, then based on the resource usage and type of operations performed by these nodes, we might want to have dedicated nodes.
