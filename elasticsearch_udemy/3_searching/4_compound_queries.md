# Compound queries

## Querying with boolean logic

```JSON
GET recipes/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": {
          "ingredients.name": "parmesan"
        }},
        {"range": {
          "preparation_time_minutes": {
            "lte": 15
          }}
        }
      ]
    }
  }
}
```

In the above query, the `range` check and the `match` both execute within the query context(conditions inside the `must` clause execute in the query context).
But having the range inside the query context is expensive. So we could move the range condition to execute inside the filter context. `bool.filter` executes the query in the filter context.

```JSON
GET recipes/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": {
          "ingredients.name": "parmesan"
        }}],
        "filter": [
        {"range": {
          "preparation_time_minutes": {
            "lte": 15
          }}
        }
      ]
    }
  }
}
```

Query clauses are cached in the filter context.

Queries inside the `must_not` clause also execute in the filter context.

```JSON
GET recipes/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": {
          "ingredients.name": "parmesan"
        }}],
        "must_not": [
          {"match": {
            "ingredients.name": "tuna"
          }}
        ],
        "filter": [
        {"range": {
          "preparation_time_minutes": {
            "lte": 15
          }}
        }
      ]
    }
  }
}
```

Condition for adding preferences using `should`. `should` behaves differently inside the `bool` with query /filter context. If the `bool` contains other clauses like `must`, `filter`, then `should` acts like influencer on the relevant scores.

```JSON
GET recipes/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": {
          "ingredients.name": "parmesan"
        }}],
        "must_not": [
          {"match": {
            "ingredients.name": "tuna"
          }}
        ],
        "should": [
          {"match": {
            "ingredients.name": "parsley"
          }}
        ],
        "filter": [
        {"range": {
          "preparation_time_minutes": {
            "lte": 15
          }}
        }
      ]
    }
  }
}
```

When `should` is present along side `must` or `filter`, the match is optional. If there is a match on the should clause, the relevance score is boosted.

But if the `bool` contains only the `should` clause, then the match is not optional. It will only fetch documents that have a match.

## Named queries

- Named queries are helpful in debugging boolean logic.
- Query response contains the names of the boolean logic that the documents matched (`matched_queries` section in the JSON).

```JSON
GET recipes/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": {
          "ingredients.name": {
            "query": "parmesan",
            "_name": "parmesan_must"
          }
        }}],
        "must_not": [
          {"match": {
            "ingredients.name": {
            "query": "tuna",
            "_name": "tuna_mustnot"
          }
          }}
        ],
        "should": [
          {"match": {
            "ingredients.name": {
            "query": "parsley",
            "_name": "parsley_should"
          }
          }}
        ],
        "filter": [
        {"range": {
          "preparation_time_minutes": {
            "lte": 15,
            "_name": "prep_time_filter"
          }}
        }
      ]
    }
  }
}
```

Query response

```JSON
{
  "took" : 5,
  "timed_out" : false,
  "_shards" : {
    "total" : 2,
    "successful" : 2,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 2,
      "relation" : "eq"
    },
    "max_score" : 2.1246665,
    "hits" : [
      {
        "_index" : "recipes",
        "_type" : "_doc",
        "_id" : "10",
        "_score" : 2.1246665,
        "_source" : {
          "title" : "Penne With Hot-As-You-Dare Arrabbiata Sauce",
          "description" : """Exceedingly simple in concept and execution, arrabbiata sauce is tomato sauce with the distinction of being spicy enough to earn its "angry" moniker. Here's how to make it, from start to finish.""",
          "preparation_time_minutes" : 15,
          "servings" : {
            "min" : 4,
            "max" : 4
          },
          "steps" : [
            "In a medium saucepan of boiling salted water, cook penne until just short of al dente, about 1 minute less than the package recommends.",
            "Meanwhile, in a large skillet, combine oil, garlic, and pepper flakes. Cook over medium heat until garlic is very lightly golden, about 5 minutes. (Adjust heat as necessary to keep it gently sizzling.)",
            "Add tomatoes, stir to combine, and bring to a bare simmer. When pasta is ready, transfer it to sauce using a strainer or slotted spoon. (Alternatively, drain pasta through a colander, reserving 1 cup of cooking water. Add drained pasta to sauce.)",
            "Add about 1/4 cup pasta water to sauce and increase heat to bring pasta and sauce to a vigorous simmer. Cook, stirring and shaking the pan and adding more pasta water as necessary to keep sauce loose, until pasta is perfectly al dente, 1 to 2 minutes longer. (The pasta will cook more slowly in the sauce than it did in the water.)",
            "Continue cooking pasta until sauce thickens and begins to coat noodles, then remove from heat and toss in cheese and parsley, stirring vigorously to incorporate. Stir in a drizzle of fresh olive oil, if desired. Season with salt and serve right away, passing more cheese at the table."
          ],
          "ingredients" : [
            {
              "name" : "Kosher salt"
            },
            {
              "name" : "Penne pasta",
              "quantity" : "450g"
            },
            {
              "name" : "Extra-virgin olive oil",
              "quantity" : "3 tablespoons"
            },
            {
              "name" : "Clove garlic",
              "quantity" : "1"
            },
            {
              "name" : "Crushed red pepper"
            },
            {
              "name" : "Can whole peeled tomatoes",
              "quantity" : "400g"
            },
            {
              "name" : "Finely grated Parmesan cheese",
              "quantity" : "60g"
            },
            {
              "name" : "Minced flat-leaf parsley leaves",
              "quantity" : "Small handful"
            }
          ],
          "created" : "2017/04/27",
          "ratings" : [
            1.5,
            2.0,
            4.0,
            3.5,
            3.0,
            5.0,
            1.5
          ]
        },
        "matched_queries" : [
          "prep_time_filter",
          "parmesan_must",
          "parsley_should"
        ]
      },
      {
        "_index" : "recipes",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 1.037802,
        "_source" : {
          "title" : "Fast and Easy Pasta With Blistered Cherry Tomato Sauce",
          "description" : "Cherry tomatoes are almost always sweeter, riper, and higher in pectin than larger tomatoes at the supermarket. All of these factors mean that cherry tomatoes are fantastic for making a rich, thick, flavorful sauce. Even better: It takes only four ingredients and about 10 minutes, start to finish—less time than it takes to cook the pasta you're gonna serve it with.",
          "preparation_time_minutes" : 12,
          "servings" : {
            "min" : 4,
            "max" : 6
          },
          "steps" : [
            "Place pasta in a large skillet or sauté pan and cover with water and a big pinch of salt. Bring to a boil over high heat, stirring occasionally. Boil until just shy of al dente, about 1 minute less than the package instructions recommend.",
            "Meanwhile, heat garlic and 4 tablespoons (60ml) olive oil in a 12-inch skillet over medium heat, stirring frequently, until garlic is softened but not browned, about 3 minutes. Add tomatoes and cook, stirring, until tomatoes begin to burst. You can help them along by pressing on them with the back of a wooden spoon as they soften.",
            "Continue to cook until sauce is rich and creamy, about 5 minutes longer. Stir in basil and season to taste with salt and pepper.",
            "When pasta is cooked, drain, reserving 1 cup of pasta water. Add pasta to sauce and increase heat to medium-high. Cook, stirring and tossing constantly and adding reserved pasta water as necessary to adjust consistency to a nice, creamy flow. Remove from heat, stir in remaining 2 tablespoons (30ml) olive oil, and grate in a generous shower of Parmesan cheese. Serve immediately, passing extra Parmesan at the table."
          ],
          "ingredients" : [
            {
              "name" : "Dry pasta",
              "quantity" : "450g"
            },
            {
              "name" : "Kosher salt"
            },
            {
              "name" : "Cloves garlic",
              "quantity" : "4"
            },
            {
              "name" : "Extra-virgin olive oil",
              "quantity" : "90ml"
            },
            {
              "name" : "Cherry tomatoes",
              "quantity" : "750g"
            },
            {
              "name" : "Fresh basil leaves",
              "quantity" : "30g"
            },
            {
              "name" : "Freshly ground black pepper"
            },
            {
              "name" : "Parmesan cheese"
            }
          ],
          "created" : "2017/03/29",
          "ratings" : [
            4.5,
            5.0,
            3.0,
            4.5
          ]
        },
        "matched_queries" : [
          "prep_time_filter",
          "parmesan_must"
        ]
      }
    ]
  }
}
```

## How the match query works?

- Match query constructs a boolean query internally.
- Tokens obtained from the analysis process is used to construct a bool query joined by OR.

- Search title

```JSON
GET recipes/_search
{
  "query": {
    "match": {
      "title": "pasta carbonara"
    }
  }
}
```

Above query is equivalent as the below query(given we have manually done the match terms analysis like lowercasing)

```JSON
GET recipes/_search
{
  "query": {
    "bool": {
      "should": [
        {"term": {"title": "pasta"}},
          {"term": {"title": "carbonara"}}
      ]
    }
  }
}
```

- Search title using and

```JSON
GET recipes/_search
{
  "query": {
    "match": {
      "title": {
        "query": "pasta carbonara",
        "operator": "and"
      }
    }
  }
}
```

Above query is the equivalent as below query

```JSON
GET recipes/_search
{
  "query": {
    "bool": {
      "must": [
        {"term": {"title": "pasta"}},
          {"term": {"title": "carbonara"}}
      ]
    }
  }
}
```
