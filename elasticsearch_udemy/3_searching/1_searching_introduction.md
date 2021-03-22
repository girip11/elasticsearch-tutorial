# Introduction to Searching

## Query styles

### 1. Using Query DSL

Recommended using this

```text
GET products/_search
{
  "query": {
    "match": {
      "description": "red wine"
    }
  }
}
```

### 2. Query using request URL

- Syntax: `GET products/_search?q=<field_name>:<value>`. Ex- `GET products/_search?q=name:ball`
- This syntax is short, but with complex search queries, this form becomes unreadable. Not recommended for production.
- This approach is also referred to as **query string**

Examples

```text
GET products/_search?q=name:Ball AND tags:Tennis
```

## Query DSL

- In the below example, `match_all` is the type of the query performed. There are different types of queries available like `match`, `exists`, `match_phrase` etc.

```text
GET products/_search
{
  "query": {
    "match_all": {

    }
  }
}
```

## How the search works?

- By default every node in the cluster can act as coordinating node.
- In case of search queries, the search request is broadcasted to all shards(1 per replication group) and the results from all those shards are collected, merged and sorted and returned by the coordinating node.

## Understanding query results

- `took` - time taken in milliseconds
- `timed_out` - boolean indicating whether the search request timed out or not.
- `_shards` - how many shards were searched
- Under `hits` object, `total` - how many documents matched the query and `hits` subobject contains all the matched documents.
- Every subobjects `hits` has a `_score` and the parent `hit` object has `max_score`.
- By default matches are sorted in descending order by the `_score`. `max_score` is the max of all `_score`

```JSON
{
  "took" : 10,
  "timed_out" : false,
  "_shards" : {
    "total" : 2,
    "successful" : 2,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 4,
      "relation" : "eq"
    },
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "products",
        "_type" : "_doc",
        "_id" : "100",
        "_score" : 1.0,
        "_source" : {
          "name" : "Espresso Machine",
          "price" : 1,
          "in_stock" : 1
        }
      },
      {
        "_index" : "products",
        "_type" : "_doc",
        "_id" : "101",
        "_score" : 1.0,
        "_source" : {
          "price" : 15,
          "name" : "Cricket Bat",
          "in_stock" : 39
        }
      },
      {
        "_index" : "products",
        "_type" : "_doc",
        "_id" : "XrI2U3gBOolW2mVu0eyH",
        "_score" : 1.0,
        "_source" : {
          "price" : 1,
          "name" : "Pen",
          "in_stock" : 5,
          "tags" : [
            "electronics"
          ]
        }
      },
      {
        "_index" : "products",
        "_type" : "_doc",
        "_id" : "201",
        "_score" : 1.0,
        "_source" : {
          "name" : "a",
          "price" : 1,
          "in_stock" : 3
        }
      }
    ]
  }
}
```

## Relevance score

- `_score` contains the relevance score.
- Relevance score algorithm - ES used an approach called TF/IDF which is Term Frequency/Inverse Document Frequency. Now Okapi BM25 algorithm is used.

### TF/IDF algorithm

- Term frequency - More time the search term appears in the field, higher the relevant score.
- Inverse document frequency - How many times the search term appears in the whole index. More often is appears in the index, lesser the relevance score. Hence the name inverse document frequency (i.e) relevant score is inversely proportional to the document frequency.
- Field length norm - Term appearing in a short field has more relevance score.

**NOTE**: Every time a document is added, these 3 metrics are computed and stored.

### BM25 algorithm

- Better at handling stop words. Ex: `the` is a stop words.
- Nonliear term frequency saturation - occurrences increases, then the boost stops to increase.
- Improves the field length norm factor.

## Explaining a query

```JSON
GET products/_search
{
  "explain": true,
  "query": {
    "term": {
      "name": "pen"
    }
  }
}
```

## Troubleshoot query

- [Explain API documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-explain.html)
- `GET products/_explain/<doc_id>`

```JSON
// we expected the name to match on the document 100
GET products/_explain/100
{
  "query": {
    "term": {
      "name": "Pen"
    }
  }
}
```

## Query contexts

- Query context or filter context
- Filter context - ES does not evaluate relevance scores.
- Relevance score is computed only in the query context.
- Filter context is boolean (does the field contains the search term or not)

## Full text queries vs term level queries

- Term level query - Search for exact match in the inverted index. Here the terms are not analyzed.
- Terms get looked up in the inverted index directly using the search term in the request without analyzing the search term.
- In the inverted index are all stored in lower case letters(or words in the document are analyzed and stored after normalizing the text).
- Thus when we try to match a term with uppercase letters, our query would not return the expected results because its matching uppercase letters with contents in inverted index which are lowercased.

- Casing matters in term level queries. Hence donot use term level queries to match text. Term level queries are suitable **for searching ENUM values, numbers, values**

```JSON
// This will match
GET products/_search
{
  "explain": true,
  "query": {
    "term": {
      "name": "pen"
    }
  }
}

// This won't match even though the name value is Pen
GET products/_search
{
  "explain": true,
  "query": {
    "term": {
      "name": "Pen"
    }
  }
}
```

- Full text query - Here the search term is analyzed. ES preprocesses the term and converts it to lowercase before comparing it with the **inverted index**.

- In summary, full text analyzer uses the same analyzer that was used to create the inverted index for the searched field(so the search term goes through the same normalization process).

```JSON
// this is an example of full text query
// here casing does not matter since ES converts them to lowercase
// before comparison
GET products/_search
{
  "query": {
    "match": {
      "name": "pen"
    }
  }
}
```

**NOTE**: Always use full text queries to perform full text searches. Full text query, analyzer also does other steps like stemming etc.

---

## References

- [Query string documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-query-string-query.html)
