# Term level queries

- Commonly used for querying structured data like datetime, numbers, enum values etc.

## Searching a term

Example-1: Search directly with value

```JSON
GET some_index_name/_search
{
  "query": {
    "term": {
      "status": "active"
    }
  }
}
```

Examplel-2: Search a field with options.

```JSON
GET products/_search
{
  "query": {
    "term": {
      // notice here we pass an object to the field_name
      "field_name": {
        "value": "field_value_to_lookup"
      }
    }
  }
}

```

## Searching multiple terms

- This is equivalent to `IN` clause in SQL

```JSON
// matches tags with either electronics or android
GET products/_search
{
  "query": {
    "terms": {
      "tags.keyword": [
        "electronics",
        "android"
      ]
    }
  }
}
```

## Get document based on IDs

- Fetch a bunch of documents when the IDs are known.

```JSON
GET products/_search
{
  "query": {
    "ids": {
      "values": [100, 200, 201]
    }
  }
}
```

## Matching fields with range values

- `gte`, `lte`, `gt`, `lt`

- Range also works with date values.

```JSON
GET products/_search
{
  "query": {
    "range": {
      "in_stock": {
        "gte": 1,
        "lte": 10
      }
    }
  }
}

// format of date values should match the format field
GET products/_search
{
  "query": {
    "range": {
      "added_on": {
        "gte": "2020-01-01",
        "lte": "2020-03-31",
        "format": "yyyy-MM-dd"
      }
    }
  }
}
```

## Working with relative date

- Syntax: `anchor_date_string||<integer_value>[y|d|M]/[d|M]`
- Value after `||` are either added or subtracted from the anchor date.
- Valuer after `/` is used to round of the date by day, month

```JSON
GET products/_search
{
  "query": {
    "range": {
      "created": {
        "gte": "2020/01/01||-1y"
      }
    }
  }
}

//current time, subtract 1 year, and round by month
GET products/_search
{
  "query": {
    "range": {
      "created": {
        "gte": "now||-1y/M"
      }
    }
  }
}
```

**NOTE**: Visit the lecture 88(working with relative date) for detailed explanation.

## Matching documents with non null values

- `exists` is the query type.
- For primitive types, the value should be non null
- For array fields, the field should have atleast 1 value.

```JSON
// returns all those products that have atleast one tag
GET products/_search
{
  "query": {
    "exists": {
      "field": "tags"
    }
  }
}
```

## Matching documents based on prefixes

- `prefix` is the query type

```JSON
// all products starting with cricket
GET products/_search
{
  "query": {
    "prefix": {
      "name": "cricket"
    }
  }
}
```

## Searching with wildcards

- Supported wildcards `*` and `?`
- `?` - match single character
- `*` - sequence of characters(no characters also)

```JSON
GET products/_search
{
  "query": {
    "wildcard": {
      "name": "ba*"
    }
  }
}
```

- Wildcard queries can be slow when we place `*` as the term prefix.

## Searching with regular expressions

- Character classes are not supported. Anchors like `^` and `$` are also not supported. [regexp query doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-regexp-query.html#regexp-syntax)

```JSON
GET products/_search
{
  "query": {
    "regexp": {
      "name": "[c|p].*"
    }
  }
}
```

- Limit the use of wildcard patterns to speed up the query particularly as the prefix of the search term. Affects query time in large clusters.
