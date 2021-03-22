# Managing documents

ES exposes REST API. All the below queries are to be tried out in Kibana query devtool.

## Index creation and deletion

- `PUT pages` - create index named pages.
- `DELETE pages`
- Create index with sharding and replication settings

```text
PUT pages
{
  "settings": {
    "number_of_shards": 2,
    "number_of_replicas": 2
  }
}
```

## Add documents to index

- POST request to add a document to index.
- Adding document to non existent index will create the index and add the document to the index(not recommended. In production we should disable this `auto_create_index` setting).
- We add documents to `<index_name>/_doc`

```text
POST products/_doc
{
  "name": "Pen",
  "price": 1,
  "in_stock": 10
}

# with custom ID, we use PUT
PUT products/_doc/100
{
  "name": "Pen",
  "price": 1,
  "in_stock": 10
}
```

**NOTE**: Check the `result` field in the response should have value `created`

## Retrieve document

- `GET products/_doc/<id>`

## Updating documents

- `POST products/_update/<id>`
- Status of the update operation is inferred from the result field.

```text
POST products/_update/1
{
  "doc":  {
    "in_stock": 2
  }
}

<!-- Can be verified using below query -->
GET products/_doc/1
```

- We can also add a new field to the existing document

- Elastic search documents are immutable. So update means, existing document is **replaced** with a new document with the given new fields added and the same ID.

### Scripted updates

- Custom logic to access a document values.

```text
POST products/_update/XrI2U3gBOolW2mVu0eyH
{
  "script": {
    # ctx _source gives the source document
    # basically the document for the given ID
    # then we decrement the in_stock field value
    "source": "ctx._source.in_stock--"
  }
}

# assignment
POST products/_update/1
{
  "script": {
    "source": "ctx._source.in_stock = 10"
  }
}

# Programatically we can pass parameters to the script
POST products/_update/1
{
  "script": {
    "source": "ctx._source.in_stock -= params.quantity",
    "params": {
      "quantity": 4
    }
  }
}
```

**NOTE** - Scripted update always updates the record unlike update API. But using `noop` and if check, we can prevent this unnecessary document update.

```text
# in the response result will be set to noop
# we can also set ctx.op = 'delete' to delete the document.
POST products/_update/XrI2U3gBOolW2mVu0eyH
{
  "script": {
    "source": """
    if (ctx._source.in_stock == 0) {
      ctx.op = 'noop';
    }

    ctx._source.in_stock--
    """
  }
}

# we can also set ctx.op = 'delete' to delete the document.
POST products/_update/XrI2U3gBOolW2mVu0eyH
{
  "script": {
    "source": """
    if (ctx._source.in_stock < 1) {
      ctx.op = 'delete';
    }

    ctx._source.in_stock--
    """
  }
}
```

## Upserts

- If document already exists script is run else new document is created.

```text
# update if doc with ID exists
# insert if the ID is not found
POST products/_update/100
{
  "script": {
    "source": "ctx._source.in_stock++"
  },
  "upsert": {
    "name": "Tennis Ball",
    "price": 5,
    "in_stock": 100
  }
}
```

## Replace documents

```text
# With this replacement, in_stock field in the document will be gone
PUT products/_doc/100
{
  "name": "Tennis Ball",
  "price": 10
}
```

## Delete document

- `DELETE products/_doc/100`

## Understanding routing

- Routing - process of resolving a shard for a document.
- Update, delete - identify shard where the document is located. For insert, routing tells us in which shard the document should reside.
- `shard_num = hash(_routing) % num_primary_shards` - Hash is on the document ID. This is the default routing strategy. But we can change this strategy.
- Default routing strategy ensures the documents are distributed evenly among the configured shards.
- Due to this routing strategy, when the shards are increased, we have to recreate the index with the newly configured number of shards.

## How ES reads data?

- Client sends the read request to ES cluster. Coordinating node receives the read request.

- ARS - Adaptive Replica Selection. Selecting the best replica(performance wise) from a replication group to serve the read request. Once the replica is selected, coordinating node sends the request to that node and collects the response and sends to the client(program or kibana)

## How ES write works?

- Request from client to coordinating node. Write requests always routed to the primary shard in the replication group.
- Primary shard before writing validates the data. Primary forwards the write operation in parallel to the other replica shards in the replication group.

- Primary term - Distinguish between old and new primary shards. This is a counter that contains how many times the primary shards got changed.
- There is one primary term counter for every replication group. These counter values are also persisted in the ES cluster. Notice the field called `"_primary_term" : 1,` in the response of the write requests.

- Sequence number is also given to each operation. `"_seq_no" : 3` in the JSON response can be found. Counter is incremented for each write operation. This counter will be incremented by the primary shard when it receives a write operation.

- Global and local checkpoints are used when documents are indexed and retrieved at a very high speed. Every replication group has a global checkpoint, each replica has a lcoal checkpoint.
- Global checkpoint - sequence number that all active shards within the replication group have seen/aligned till now.
- Local checkpoint - Only operations that have sequence number higher than the local checkpoint needs to be written to this shard.

- Primary terms and sequence numbers are available in both write responses and when we fetch the document by ID.

## Document versioning

- Remember document is immutable. Every update operation on a document increments the `_version` field. This is referred to as internal versioning
- External versioning - maintain version outside of ES and store the version in a DB. We can pass external version to the document in the ES. **External versioning is obselete now**.

```text
PUT products/_doc/111?version=3&version_type=external
{
  "name": "Ball",
  "price": 10,
  "in_stock": 1
}
```

## Optimistic concurrency control

- Prevent older version of document rewriting a newer version of the document when same document is updated concurrently.
- `_version` was used to resolve the conflict. But now primary term and sequence numbers are used to handle this issue.
- If request1 updates the document, then its sequence number would be incremented. If request2 tries to update the same document, the sequence number would have changed and hence the update will fail.
- I have seen fields like `Etag` used to solve this optimistic concurrency problem.
- When the application gets this error, then we need to fetch the document again and update it in the application logic.

## Update by query

- `POST products/_update_by_query` is used to update multiple documents (bulk update).

```text
POST products/_update_by_query
{
  "script": {
    "source": "ctx._source.in_stock--"
  },
  "query": {
    "match_all": {}
  }
}
```

## Delete multiple documents

- `POST products/_delete_by_query` - delete all documents matching the input criteria.
- `"conflict": "proceed"` can be added to resolve the conflicts.

```text
POST products/_delete_by_query
{
  "query": {
    "match_all": {}
  }
}
```

## Bulk API

- Bulk api expects the data formatted using the NDJson specification.
- Lines separated by new line character. Each line should be a json object.
- Multiple operations on various documents in one request.
- Actions are index, create, update and delete.
- Using bulk api, we can perform operations on different indices. That's why we have `"_index": "<index_name>"` specified along with each action.
- If all the actions are going to be performed on the same index, then we

```text
POST _bulk
# action and metadata line
# source document line
# action and metadata line
# source document line
```

```text
POST _bulk
{ "index": {"_index": "products", "_id": 100} }
{"name": "Expresso Machine", "price": 1, "in_stock": 1}
{ "create": {"_index": "products", "_id": 201}}
{"name": "a", "price": 1, "in_stock": 3}
```

**NOTE**: Delete action does not have a source document line.

```text
POST products/_bulk
{ "update": { "_id": 100} }
{ "doc": { "in_stock": 5 } }
{ "delete": {"_id": 201} }
```

We can check the results of these operations using the below search query.

```text
POST products/_search
{
  "query": {
    "match_all": {}
  }
}
```

- `Content-Type` header should be set to `application/x-ndjson` in the http request when using the bulk api. This is useful to know when making requests using tool like postman, curl.
- Each line in the request must end with `\n` or `\r\n` including the last line(last line should also end with newline character).
- A failed action will not affect the other actions. Response contains detailed information about every action. Order of operations is preserved in the response too.

- Bulk api is **efficient**. Ex: Many write operations(like import data) at the same time.
- Bulk api **supports optimistic concurrency control** by adding `if_primary_term` and `if_seq_no` parameters in the actions metadata.

## Bulk api using curl

```Bash
cd "directory_containing_json"
curl -H "Content-Type:application/x-ndjson" -XPOST "http://localhost:9201/products/_bulk" --data-binary "@filename.json"
```
