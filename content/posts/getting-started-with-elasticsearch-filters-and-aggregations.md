---
title: "Getting started with Elasticsearch filters and aggregations"
date: 2015-08-31
---

## What is Elasticsearch?

[Elasticsearch](http://www.elasticsearch.com/) is [Apache Lucene](http://lucene.apache.org/core/) on steroids. It uses Lucene at its core for full-text indexing and search. ES adds distribution, near real-time search, high availability, a RESTful interface and many more features to it.

It basically makes out of the box usage of Apache Lucene easy.

### Why use elasticsearch?

With Lucene at the core of ES, it's perfect for text-based search. Going from blog posts to addresses to social media data to logs.

Although it is excellent for text-based search, combining the search with aggregations it's also great for analytics and result calculations.

These are just some examples elasticsearch is good at, but at the core there's always search. Hence their punchline "You Know, for Search".

For a brief overview of possibilities you can take a look at the [Case Studies](http://www.elasticsearch.org/case-studies/) page on the ES site.

## Data access

As previously said, elasticsearch adds a RESTful interface to Apache Lucene. This means that you can access your data (and also add data) through an HTTP interface. Just throw JSON documents to the right index and it'll get added, query on the right index and you'll get a JSON response.

### Data structure

To understand the following examples, it's good to know out of which parts an elasticsearch query exists.

You'll see URLs like *http://localhost:9200/blog/posts/1*, but what do these separate parts actually mean?

The blog part is the elasticsearch index. You can have several of those in your cluster and they basically represent a database. The posts part is called the type and would represent the database tables. The last part, the 1 is the id of our document in this case. You'll see later on that we can replace this with _search for example to actually execute a query.

### Example

I won't go very deep into this as many other blogposts cover this, but I will recap the basics. (You can find other resources at the bottom of this page)

**Adding data**

Adding data to your indices is pretty simple, just do a PUT request to the index you want to add the data to with the JSON string and that's it.

```
$ curl -XPUT http://localhost:9200/books/book/1 -d '
{"title": "Elasticsearch one on one", "content": "This is a book to learn the basics of elasticsearch", "author": "elasticauthor"}'

{"_index":"books","_type":"book","_id":"1","_version":1,"created":true}
```

This will create a new document with id 1 in the post type for our blog index.

**Getting data**

Retrieving data is very easy as well, just do a GET request to the post URL and that's it!

```
$ curl -XGET http://localhost:9200/books/book/1

{
    "_index": "books",
    "_type": "book",
    "_id": "1",
    "_version": 1,
    "found": true,
    "_source" :  {
        "title": "Elasticsearch one on one",
        "content": "This is a book to learn the basics of elasticsearch",
        "author": "elasticauthor"
    }
}
```

## Searching

Searching in Elasticseach happens through the Query DSL which is entirely based on JSON. There are two categories, basic queries and filters. Some of these queries can contain other queries, which we'll see in the filters section.

Both of these can be used in different parts of the API, we'll be exploring the search query API in this part. Later on, we'll show you how to use it in the aggregations API.

## Queries

If you're looking for most relevant items on a full text search, then these are the way to go. Elasticsearch provides the scoring mechanism from what's under the hood, Apache Lucene, through their interface.

Several of these queries use other queries to execute a search. One of these is the filtered query. This is used to combine another query with a filter (which we'll talk about later). The main goal here is to exclude as many documents with a filter and perform the query on that.

Below is an example of a simple query to search all the books with elasticsearch in the title. You can find the dataset [on Gist](https://gist.github.com/jelmersnoeck/917643910cc66f64e5df).

```
$ curl -XGET http://localhost:9200/books/book/_search?pretty=true -d '{
        "query": {
            "term": { "title": "elasticsearch" }
        }
    }'

{
    "took" : 4,
    "timed_out" : false,
    "_shards" : {
        "total" : 5,
        "successful" : 5,
        "failed" : 0
    },
    "hits" : {
        "total" : 4,
        "max_score" : 0.30685282,
        "hits" : [ {
            "_index" : "books",
            "_type" : "book",
            "_id" : "4",
            "_score" : 0.30685282,
            "_source" : {
                "title": "Elasticsearch",
                "content": "This is the main book about Elasticsearch",
                "author": "Shay Banon"
            }
        }, {
            "_index" : "books",
            "_type" : "book",
            "_id" : "2",
            "_score" : 0.2972674,
            "_source" : {
                "title": "Elasticsearch and Lucene",
                "content": "This is a book to learn about Lucene and Elasticsearch",
                "author": "elasticauthor"
            }
        }, {
            "_index" : "books",
            "_type" : "book",
            "_id" : "1",
            "_score" : 0.15342641,
            "_source" : {
                "title": "Elasticsearch one on one",
                "content": "This is a book to learn the basics of elasticsearch",
                "author": "elasticauthor"
            }
        } ]
    }
}
```

### Filters

Previously we executed a query to find all books. But what if we want to find books from a specific author? This is where filters come in. Below you can find the same query but with a filter on the author.

```
$ curl -XGET http://localhost:9200/books/book/_search?pretty=true -d '{
        "query": {
            "filtered": {
                "query": { "term": { "title": "elasticsearch" } },
                "filter": {
                    "term": {
                        "author": "elasticauthor"
                    }
                }
            }
        }
    }'
```

This will filter the books based on the author, in this case elasticauthor and perform the term query for the title on that data set.

```
{
    "took" : 1,
    "timed_out" : false,
    "_shards" : {
        "total" : 5,
        "successful" : 5,
        "failed" : 0
    },
    "hits" : {
        "total" : 2,
        "max_score" : 0.15342641,
        "hits" : [ {
            "_index" : "books",
            "_type" : "book",
            "_id" : "2",
            "_score" : 0.15342641,
            "_source" : {
                "title": "Elasticsearch and Lucene",
                "content": "This is a book to learn about Lucene and elasticsearch",
                "author": "elasticauthor"
            }
        }, {
            "_index" : "books",
            "_type" : "book",
            "_id" : "1",
            "_score" : 0.15342641,
            "_source" : {
                "title": "Elasticsearch one on one",
                "content": "This is a book to learn the basics of elasticsearch",
                "author": "elasticauthor"
            }
        } ]
    }
}
```

### Filters explained

So what exactly are these filters? These are basically queries to narrow down your dataset. If you know an exact value of something and you want all the data for that value to perform some more logic on, you can use a filter. Remember, filters won't have a score associated with them, which makes them much faster. They're also cacheable by adding "_cache": true. (or false, you can use this to manage cache control. You can also use "_cache_key" to set the key that will be used)

You'll be able to use most of the search query queries in a filter, although not all of them. One of them which pops in my mind is the match and span query. You will also see some new available filters, like the and filter.

You can view a full set of [queries](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/query-dsl-queries.html) and [filters](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/query-dsl-filters.html) on the elasticsearch website.

### Aggregations

In the beginning I talked about elasticsearch being useful for analytics. This is where aggregations come in. Aggregations are basically executing analytical queries on the dataset you retrieved.

Say we want to count the number of words in the books that we've selected with our previous search. We could simply add a value_count aggregation for that.

```
$ curl -XGET http://localhost:9200/books/book/_search?search_type=count -d '{
    "query": {
        "filtered": {
            "query": { "term": { "title": "elasticsearch" } },
                "filter": {
                    "term": {
                        "author": "elasticauthor"
                    }
                }
        }
    },
    "aggs": {
        "words": {
            "value_count": { "field": "content" }
        }
    }
}'
```

Note the search_type=count in the querystring. This is added because we don't really care about the results, we just want the results of our aggregations.

```
{
    "took": 2,
    "timed_out": false,
    "_shards": {
        "total": 5,
        "successful": 5,
        "failed": 0
    },
    "hits": {
        "total": 3,
        "max_score": 0,
        "hits": []
    },
    "aggregations": {
        "words": {
            "value": 20
        }
    }
}
```

**Sub aggregations**

Aggregations are quite powerful and useful in several scenarios. Below I've illustrated a small example (not with books) where sub aggregations can be useful. With sub aggregations I mean aggregations within aggregations. The example is a bolt store which categorizes their bolts and we want to get the average weight and price for each category.

The dataset can be found [here](https://gist.github.com/jelmersnoeck/b03ce4274209d1a5a990).

To get the averages per category, we can just use the terms aggregation. This will create a bucket per category and apply the specified aggregations on that.

```
$ curl -XGET http://localhost:9200/store/bolts/_search?search_type=count -d '{
    "aggs": {
        "averages": {
            "terms": { "field": "category" },
            "aggs": {
                "weight": { "avg": { "field": "weight" } },
                "price": { "avg": { "field": "price" } }
            }
        }
    }
}'

{
    "took": 1,
    "timed_out": false,
    "_shards": {
        "total": 5,
        "successful": 5,
        "failed": 0
    },
    "hits": {
        "total": 4,
        "max_score": 0,
        "hits": []
    },
    "aggregations": {
        "averages": {
            "buckets": [
            {
                "key": "c1022",
                "doc_count": 2,
                "weight": {
                    "value": 3
                },
                "price": {
                    "value": 8.09
                }
            },
            {
                "key": "en14592",
                "doc_count": 2,
                "weight": {
                    "value": 3.55
                },
                "price": {
                    "value": 5.6
                }
            }
            ]
        }
    }
}
```

As you can see, we have two buckets here. The key represents the category (the field that you specified in the terms query) and then you have both the aggregations you specified.

## Resources

- [Elasticsearch](http://www.elasticsearch.org/guide/) — the elasticsearch guide
- [Case studies](http://www.elasticsearch.org/case-studies/)
- [Marvel](http://www.elasticsearch.com/marvel/) — an elasticsearch monitoring tool (paid)
- [Bigdesk](https://github.com/lukas-vlcek/bigdesk) — another monitoring tool (free)
- [Exploring elasticsearch](http://exploringelasticsearch.com/) — a book that covers a great deal about elasticsearch
- [Elasticsearch introduction](http://www.jurriaanpersyn.com/archives/2013/11/18/introduction-to-elasticsearch/) — great introduction on how to get started
- [Getting started with elasticsearch](http://red-badger.com/blog/2013/11/08/getting-started-with-elasticsearch/) — another great introduction on how to get started
