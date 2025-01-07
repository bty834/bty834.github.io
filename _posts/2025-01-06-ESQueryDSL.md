---
title: ES Query DSL
categories: [编程, 中间件]
tags: [elasticsearch, dsl]
---

查看es版本：
```
GET /

{
  "name" : "node-1",
  "cluster_name" : "es.piper",
  "cluster_uuid" : "Dd1luz7oSBGMsRyW8i0xBw",
  "version" : {
    // es版本
    "number" : "7.17.10", 
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "fecd68e3150eda0c307ab9a9d7557f5d5fd71349",
    "build_date" : "2023-04-23T05:33:18.138275597Z",
    "build_snapshot" : false,
    "lucene_version" : "8.11.1",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```

---

_cat
```
查看节点
GET /_cat/nodes?v

ip         heap.percent ram.percent cpu load_1m load_5m load_15m node.role   master name
172.19.0.3           17          57   0    3.28    2.62     1.71 cdfhilmrstw *      node-1


查看段
GET /_cat/segments?v


查看别名
GET /_cat/aliases?v


查看插件
GET /_cat/plugins


查看索引
GET /_cat/indices?v

health status index                            uuid                   pri rep docs.count docs.deleted store.size pri.store.size
yellow open   index_test                       F7oqc9RaQ_25ZzH40yKqQA   1   1          0            0       227b           227b
green  open   kibana_sample_data_flights       O1eP_K2yTZ-Qi79lL2qBcw   1   0      13059            0      5.6mb          5.6mb

GET /_cat/health?v

epoch      timestamp cluster  status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1736177896 15:38:16  es.piper yellow          1         1     12  12    0    0        1             0                  -                 92.3%


GET /_cat/indices/kibana_sample_data_flights?v

health status index                      uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   kibana_sample_data_flights O1eP_K2yTZ-Qi79lL2qBcw   1   0      13059            0      5.6mb          5.6mb

```
Elasticsearch 集群的健康状态通常用三种颜色表示：

| 颜色          | 含义                                           |
|-------------|----------------------------------------------|
| 绿色 (Green)  | 所有主要和副本分片均已分配；集群可以完全运行，性能正常 ；这是理想状态          |
| 黄色 (Yellow) | 所有主分片已分配，但至少缺少一个副本；没有数据丢失，搜索结果仍完整；高可用性受到一定影响 |
| 红色 (Red)    | 至少缺少一个主分片及其所有副本；表示数据丢失，搜索结果不完整；建议立即检查并修复     |

--- 


mapping
```
查看所有index的mapping
GET /_all/_mapping
查看多个index的mapping
GET /kibana_sample_data_flights,index_test/_mapping


查看index kibana_sample_data_flights的mapping
GET /kibana_sample_data_flights/_mapping

{
  "kibana_sample_data_flights" : {
    "mappings" : {
      "properties" : {
        "AvgTicketPrice" : {
          "type" : "float"
        },
        "Cancelled" : {
          "type" : "boolean"
        },
        "Carrier" : {
          "type" : "keyword"
        },
        "Dest" : {
          "type" : "keyword"
        },
        "DestAirportID" : {
          "type" : "keyword"
        },
        "DestCityName" : {
          "type" : "keyword"
        },
        "DestCountry" : {
          "type" : "keyword"
        },
        "DestLocation" : {
          "type" : "geo_point"
        },
        "DestRegion" : {
          "type" : "keyword"
        },
        "DestWeather" : {
          "type" : "keyword"
        },
        "DistanceKilometers" : {
          "type" : "float"
        },
        "DistanceMiles" : {
          "type" : "float"
        },
        "FlightDelay" : {
          "type" : "boolean"
        },
        "FlightDelayMin" : {
          "type" : "integer"
        },
        "FlightDelayType" : {
          "type" : "keyword"
        },
        "FlightNum" : {
          "type" : "keyword"
        },
        "FlightTimeHour" : {
          "type" : "keyword"
        },
        "FlightTimeMin" : {
          "type" : "float"
        },
        "Origin" : {
          "type" : "keyword"
        },
        "OriginAirportID" : {
          "type" : "keyword"
        },
        "OriginCityName" : {
          "type" : "keyword"
        },
        "OriginCountry" : {
          "type" : "keyword"
        },
        "OriginLocation" : {
          "type" : "geo_point"
        },
        "OriginRegion" : {
          "type" : "keyword"
        },
        "OriginWeather" : {
          "type" : "keyword"
        },
        "dayOfWeek" : {
          "type" : "integer"
        },
        "timestamp" : {
          "type" : "date"
        }
      }
    }
  }
}
```





_count
```
GET /kibana_sample_data_flights/_count
{
  "query": {
    "match_all": {}
  }
}

{
  "count" : 13059,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  }
}



GET /kibana_sample_data_flights/_search
{
  // 表示不返回实际的结果,只返回聚合结果
  "size": 0, 
  "aggs": {
    // 聚合distinct
    "unique_values": {
      "terms": {
        // 根据DestRegion字段聚合
        "field": "DestRegion"
      }
    }
  }
}

{
  "took" : 21,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 10000,
      "relation" : "gte"
    },
    "max_score" : null,
    "hits" : [ ]
  },
  "aggregations" : {
    "unique_values" : {
      "doc_count_error_upper_bound" : 0,
      "sum_other_doc_count" : 4974,
      "buckets" : [
        {
          "key" : "SE-BD",
          "doc_count" : 3721
        },
        {
          "key" : "IT-34",
          "doc_count" : 1027
        },
        {
          "key" : "CH-ZH",
          "doc_count" : 691
        },
        {
          "key" : "CA-MB",
          "doc_count" : 460
        },
        {
          "key" : "GB-ENG",
          "doc_count" : 449
        },
        {
          "key" : "PL-MZ",
          "doc_count" : 405
        },
        {
          "key" : "AT-9",
          "doc_count" : 377
        },
        {
          "key" : "IT-21",
          "doc_count" : 332
        },
        {
          "key" : "RU-AMU",
          "doc_count" : 331
        },
        {
          "key" : "IT-25",
          "doc_count" : 292
        }
      ]
    }
  }
}

```





[field data types](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-types.html)




```
GET /kibana_sample_data_flights/_doc/{docId}

// 根据timestampe倒序排limit 1
GET /kibana_sample_data_flights/_search
{
  "query": {
    "match_all": {}
  },
  "sort": [
    {
      "timestamp": {
        "order": "desc"
      }
    }
  ],
  "size": 1
}

{
  "took" : 14, // 毫秒
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 10000,
      "relation" : "gte"
    },
    "max_score" : null,
    "hits" : [
      {
        "_index" : "kibana_sample_data_flights",
        "_type" : "_doc",
        "_id" : "OuFcL5QBJ7mq5T07D6py",
        "_score" : null,
        "_source" : {
          "FlightNum" : "GDZWNB0",
          "DestCountry" : "CN",
          "OriginWeather" : "Clear",
          "OriginCityName" : "London",
          "AvgTicketPrice" : 952.4522444587226,
          "DistanceMiles" : 5743.8378391883825,
          "FlightDelay" : false,
          "DestWeather" : "Rain",
          "Dest" : "Shanghai Hongqiao International Airport",
          "FlightDelayType" : "No Delay",
          "OriginCountry" : "GB",
          "dayOfWeek" : 6,
          "DistanceKilometers" : 9243.810963470789,
          "timestamp" : "2025-02-02T23:50:12",
          "DestLocation" : {
            "lat" : "31.19790077",
            "lon" : "121.3359985"
          },
          "DestAirportID" : "SHA",
          "Carrier" : "Kibana Airlines",
          "Cancelled" : false,
          "FlightTimeMin" : 770.3175802892324,
          "Origin" : "London Gatwick Airport",
          "OriginLocation" : {
            "lat" : "51.14810181",
            "lon" : "-0.190277994"
          },
          "DestRegion" : "SE-BD",
          "OriginAirportID" : "LGW",
          "OriginRegion" : "GB-ENG",
          "DestCityName" : "Shanghai",
          "FlightTimeHour" : 12.838626338153874,
          "FlightDelayMin" : 0
        },
        "sort" : [
          1738540212000
        ]
      }
    ]
  }
}

```

查询条件分为：
- Leaf query clauses : 叶子查询条件，如 match, term, range
- Compound query clauses : 组合查询条件，可包含叶子查询条件和嵌套组合查询条件，如 bool, dis_max

查询可分为：
- 相关性查询： query context , How well does this document match this query clause
- 过滤查询：filter context , Does this document match this query clause

过滤查询的优点：
1. simple binary logic: 只有是或否两种匹配结果
2. performance: 不需要计算  relevance scores, 查询很快
3. caching: 查询结果可以被缓存
4. resource efficiency: 相比于full-text query, filter消耗更少的cpu
5. query combination: 可以和相关性查询搭配查询

以下查询方式会使用filter context查询：
- bool中的filter和must not
- constant_score中的filter
- filter 聚合


先来看看bool查询。

boolean包含4种查询条件：
- must: 所有的查询子句都必须匹配，类似于"AND"操作，计算relevant scores
- should: 至少一个查询子句必须匹配，类似于"OR"操作，计算relevant scores
- filter: 与must类似，不计算relevant_scores
- must_mot: 查询子句不应该匹配，类似于"NOT"操作，不计算relevant scores

You can use the `minimum_should_match` parameter to specify the number or percentage of should clauses returned documents must match.

If the bool query includes at least one `should` clause and no `must` or `filter` clauses, the default value is 1.
Otherwise, the default value is 0.

```
GET /kibana_sample_data_flights/_search
{
  "query": {
    "bool": {
      "filter": {
        "term": {
          "DestRegion": "CH-ZH"
        }
      }
    }
  }
}
```
