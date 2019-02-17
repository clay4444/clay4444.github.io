---
title: ElasticSearch入门（二）
tags:
  - elasticsearch
abbrlink: 16aceb61
date: 2018-10-07 17:41:17
---

## ES 多种搜索方式

<br/>

### query string search

搜索全部商品：GET /ecommerce/product/_search

took：耗费了几毫秒
timed_out：是否超时，这里是没有
_shards：数据拆成了5个分片，所以对于搜索请求，会打到所有的primary shard（或者是它的某个replica shard也可以）
hits.total：查询结果的数量，3个document
hits.max_score：score的含义，就是document对于一个search的相关度的匹配分数，越相关，就越匹配，分数也高
hits.hits：包含了匹配搜索的document的详细数据

```json
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
    "max_score": 1,
    "hits": [
      {
        "_index": "ecommerce",
        "_type": "product",
        "_id": "2",
        "_score": 1,
        "_source": {
          "name": "jiajieshi yagao",
          "desc": "youxiao fangzhu",
          "price": 25,
          "producer": "jiajieshi producer",
          "tags": [
            "fangzhu"
          ]
        }
      },
      {
        "_index": "ecommerce",
        "_type": "product",
        "_id": "1",
        "_score": 1,
        "_source": {
          "name": "gaolujie yagao",
          "desc": "gaoxiao meibai",
          "price": 30,
          "producer": "gaolujie producer",
          "tags": [
            "meibai",
            "fangzhu"
          ]
        }
      },
      {
        "_index": "ecommerce",
        "_type": "product",
        "_id": "3",
        "_score": 1,
        "_source": {
          "name": "zhonghua yagao",
          "desc": "caoben zhiwu",
          "price": 40,
          "producer": "zhonghua producer",
          "tags": [
            "qingxin"
          ]
        }
      }
    ]
  }
}
```

<br/>

query string search的由来，因为search参数都是以http请求的query string来附带的

搜索商品名称中包含yagao的商品，而且按照售价降序排序：GET /ecommerce/product/_search?q=name:yagao&sort=price:desc

适用于临时的在命令行使用一些工具，比如curl，快速的发出请求，来检索想要的信息；但是如果查询请求很复杂，是很难去构建的
在生产环境中，几乎很少使用query string search

<br/>

### query DSL

DSL：Domain Specified Language，特定领域的语言
http request body：请求体，可以用json的格式来构建查询语法，比较方便，可以构建各种复杂的语法，比query string search肯定强大多了

查询所有的商品

```json
GET /ecommerce/product/_search
{
    "query": { "match_all": {} }
}
```

<br/>

查询名称包含yagao的商品，同时按照价格降序排序

```json
GET /ecommerce/product/_search
{
    "query" : {
        "match" : {
            "name" : "yagao"
        }
    },
    "sort": [
        { "price": "desc" }
    ]
}
```

<br/>

分页查询商品，总共3条商品，假设每页就显示1条商品，现在显示第2页，所以就查出来第2个商品

```json
GET /ecommerce/product/_search
{
  "query": { "match_all": {} },
  "from": 1,
  "size": 1
}
```

<br/>

指定要查询出来商品的名称和价格就可以

```json
GET /ecommerce/product/_search
{
  "query": { "match_all": {} },
  "_source": ["name", "price"]
}
```

<br/>

更加适合生产环境的使用，可以构建复杂的查询

<br/>

### query filter

搜索商品名称包含yagao，而且售价大于25元的商品

```json
GET /ecommerce/product/_search
{
    "query" : {
        "bool" : {
            "must" : {
                "match" : {
                    "name" : "yagao" 
                }
            },
            "filter" : {
                "range" : {
                    "price" : { "gt" : 25 } 
                }
            }
        }
    }
}
```

<br/>

### full-text search（全文检索）

```json
GET /ecommerce/product/_search
{
    "query" : {
        "match" : {
            "producer" : "yagao producer"
        }
    }
}
```

<br/>

producer这个字段，会先被拆解，建立倒排索引

special -> 4
yagao	->	4
producer ->	1,2,3,4
gaolujie ->	1
zhognhua ->	3
jiajieshi -> 	2

yagao producer ---> yagao和producer

<br/>

```json
{
    "took": 4,
    "timed_out": false,
    "_shards": {
        "total": 5,
        "successful": 5,
        "failed": 0
    },
    "hits": {
        "total": 4,
        "max_score": 0.70293105,
        "hits": [
            {
                "_index": "ecommerce",
                "_type": "product",
                "_id": "4",
                "_score": 0.70293105,
                "_source": {
                    "name": "special yagao",
                    "desc": "special meibai",
                    "price": 50,
                    "producer": "special yagao producer",
                    "tags": [
                        "meibai"
                    ]
                }
            },
            {
                "_index": "ecommerce",
                "_type": "product",
                "_id": "1",
                "_score": 0.25811607,
                "_source": {
                    "name": "gaolujie yagao",
                    "desc": "gaoxiao meibai",
                    "price": 30,
                    "producer": "gaolujie producer",
                    "tags": [
                        "meibai",
                        "fangzhu"
                    ]
                }
            },
            {
                "_index": "ecommerce",
                "_type": "product",
                "_id": "3",
                "_score": 0.25811607,
                "_source": {
                    "name": "zhonghua yagao",
                    "desc": "caoben zhiwu",
                    "price": 40,
                    "producer": "zhonghua producer",
                    "tags": [
                        "qingxin"
                    ]
                }
            },
            {
                "_index": "ecommerce",
                "_type": "product",
                "_id": "2",
                "_score": 0.1805489,
                "_source": {
                    "name": "jiajieshi yagao",
                    "desc": "youxiao fangzhu",
                    "price": 25,
                    "producer": "jiajieshi producer",
                    "tags": [
                        "fangzhu"
                    ]
                }
            }
        ]
    }
}
```

<br/>

### phrase search（短语搜索）

<br/>

跟全文检索相对应，相反，全文检索会将输入的搜索串拆解开来，去倒排索引里面去一一匹配，只要能匹配上任意一个拆解后的单词，就可以作为结果返回
phrase search，要求输入的搜索串，必须在指定的字段文本中，完全包含一模一样的，才可以算匹配，才能作为结果返回

```json
GET /ecommerce/product/_search
{
    "query" : {
        "match_phrase" : {
            "producer" : "yagao producer"
        }
    }
}
```

结果

```json
{
  "took": 11,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 1,
    "max_score": 0.70293105,
    "hits": [
      {
        "_index": "ecommerce",
        "_type": "product",
        "_id": "4",
        "_score": 0.70293105,
        "_source": {
          "name": "special yagao",
          "desc": "special meibai",
          "price": 50,
          "producer": "special yagao producer",
          "tags": [
            "meibai"
          ]
        }
      }
    ]
  }
}
```

<br/>

### highlight search（高亮搜索结果）

```json
GET /ecommerce/product/_search
{
    "query" : {
        "match" : {
            "producer" : "producer"
        }
    },
    "highlight": {
        "fields" : {
            "producer" : {}
        }
    }
}
```

<br/>

<br/>

## ElasticSearch 的数据分析（嵌套，聚合，下钻）

<br/>

### 第一个分析需求：计算每个tag下的商品数量

```json
GET /ecommerce/product/_search
{
    "size":0,
    "aggs": {
        "group_by_tags": {
            "terms": { "field": "tags" }    //terms就是按照某个字段分组
        }
    }
}
```

<br/>

操作之前需要将将文本field的fielddata属性设置为true

```json
PUT /ecommerce/_mapping/product
{
  "properties": {
    "tags": {
      "type": "text",
      "fielddata": true
    }
  }
}
```

结果

```json
{
  "took": 20,
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
    "group_by_tags": {
      "doc_count_error_upper_bound": 0,
      "sum_other_doc_count": 0,
      "buckets": [
        {
          "key": "fangzhu",
          "doc_count": 2
        },
        {
          "key": "meibai",
          "doc_count": 2
        },
        {
          "key": "qingxin",
          "doc_count": 1
        }
      ]
    }
  }
}
```

<br/>

### 第二个聚合分析的需求：对名称中包含yagao的商品，计算每个tag下的商品数量

```json
GET /ecommerce/product/_search
{
  "size": 0,
  "query": {
    "match": {
      "name": "yagao"
    }
  },
  "aggs": {
    "all_tags": {
      "terms": {
        "field": "tags"
      }
    }
  }
}
```

<br/>

### 第三个聚合分析的需求：先分组，再算每组的平均值，计算每个tag下的商品的平均价格

```json
GET /ecommerce/product/_search
{
    "size": 0,
    "aggs" : {
        "group_by_tags" : {
            "terms" : { "field" : "tags" },
            "aggs" : {
                "avg_price" : {
                    "avg" : { "field" : "price" }
                }
            }
        }
    }
}
```

结果

```json
{
    "took": 8,
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
        "group_by_tags": {
            "doc_count_error_upper_bound": 0,
            "sum_other_doc_count": 0,
            "buckets": [
                {
                    "key": "fangzhu",
                    "doc_count": 2,
                    "avg_price": {
                        "value": 27.5
                    }
                },
                {
                    "key": "meibai",
                    "doc_count": 2,
                    "avg_price": {
                        "value": 40
                    }
                },
                {
                    "key": "qingxin",
                    "doc_count": 1,
                    "avg_price": {
                        "value": 40
                    }
                }
            ]
        }
    }
}
```

<br/>

### 第四个数据分析需求：计算每个tag下的商品的平均价格，并且按照平均价格降序排序

```json
GET /ecommerce/product/_search
{
    "size": 0,
    "aggs" : {
        "all_tags" : {
            "terms" : { "field" : "tags", "order": { "avg_price": "desc" } },
            "aggs" : {
                "avg_price" : {
                    "avg" : { "field" : "price" }
                }
            }
        }
    }
}
```

<br/>

### 第五个数据分析需求：按照指定的价格范围区间进行分组，然后在每组内再按照tag进行分组，最后再计算每组的平均价格

~~~json
GET /ecommerce/product/_search
{
    "size": 0,
    "aggs": {
        "group_by_price": {
            "range": {
                "field": "price",
                "ranges": [
                    {
                        "from": 0,
                        "to": 20
                    },
                    {
                        "from": 20,
                        "to": 40
                    },
                    {
                        "from": 40,
                        "to": 50
                    }
                ]
            },
            "aggs": {
                "group_by_tags": {
                    "terms": {
                        "field": "tags"
                    },
                    "aggs": {
                        "average_price": {
                            "avg": {
                                "field": "price"
                            }
                        }
                    }
                }
            }
        }
    }
}
~~~

