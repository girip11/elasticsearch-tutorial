# Full text queries

Before proceeding with this tutorial, create `recipes` index.

```JSON
PUT recipes
{
  "settings": {
    "number_of_shards": 2,
    "number_of_replicas": 2
  }
}
```

Then add documents to the recipes index using the bulk api from cURL.

```Bash
cd elasticsearch_udemy/3_searching

curl -H "Content-Type:application/x-ndjson" -XPOST "http://localhost:9201/recipes/_bulk" --data-binary "@full_text_query_test_data.json"
```

To look at the recipe documents schema use the query `GET recipes/_mapping`.

## Matching terms

We give a sentence, phrase, then that is searched throughout all the documents.

Below query will match all documents containing any of the terms in the sentence `Recipe with pasta or spaghetti`. But terms like `with` would have low relevance score because of the inverse document frequency.

```JSON
GET recipes/_search
{
  "query": {
    "match": {
      "title": "Recipe with pasta or spaghetti"
    }
  }
}
```

Below query will fetch those documents that have all the three terms in the title

```JSON
// three terms are pasta, or , spaghetti
GET recipes/_search
{
  "query": {
    "match": {
      "title": {
        "query" :"pasta or spaghetti",
        "operator": "and"
      }
    }
  }
}
```

## Match phrases

- Phrase - sequences of words/terms
- Exact phrase must appear in the title field for the document to be returned.

```JSON
GET recipes/_search
{
  "query": {
    "match_phrase": {
      "title": "spaghetti puttanesca"
    }
  }
}
```

## Search multiple fields

- Search the term in multiple fields.

```JSON
GET recipes/_search
{
  "query": {
    "multi_match": {
      "query": "pasta",
      "fields": ["title", "description"]
    }
  }
}
```

To select only the required fields from the search result, we need to specify the `fields` with the query JSON

```JSON
// select title and description from the matches
GET recipes/_search
{
  "query": {
    "multi_match": {
      "query": "pasta",
      "fields": ["title", "description"]
    }
  },
  "fields": ["title", "description"],
  "_source": false
}
```
