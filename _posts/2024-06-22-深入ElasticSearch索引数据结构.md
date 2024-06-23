---
title: 深入理解ElasticSearch索引数据结构
categories: [编程, 中间件]
tags: [elasticsearch]
---

关于ES官网的介绍:
> Elasticsearch provides `near real-time` `search` and `analytics` for all types of data. 
> Whether you have `structured or unstructured` text, numerical data, or geospatial data, 
> Elasticsearch can efficiently store and index it in a way that supports fast searches. 
> You can go far beyond simple data retrieval and aggregate information to discover trends and patterns in your data. 
> And as your data and query volume grows, the `distributed` nature of Elasticsearch enables your deployment to grow seamlessly right along with it.

可以看到ES具有一下特点：
1. 近实时
2. 用于检索和分析：OLAP型数据库
3. 支持非结构化数据存储：NoSQL数据库
4. 分布式：支持无缝横向扩展

ES主要的使用场景如下：
1. 日志、指标等数据的存储和分析；
2. 复杂查询：如电商系统的商品检索、github站内repository检索；
3. 作为冗余数据提供多样化查询方式：SQL分库分表无法支持非分片键字段的查询，使用ES冗余数据支持多样化查询；
4. 跨库关联查询：将跨库数据关联冗余至ES，关联查询直接查ES，同时解决了跨库分页的问题。

关于ElasticSearch的基本概念可以参看：
- [ElasticSearch Docs](https://www.elastic.co/guide/en/elasticsearch/reference/current/elasticsearch-intro.html)
- [Elasticsearch 教程](https://dunwu.github.io/db-tutorial/pages/74675e/)

本文介绍了ES索引涉及到的数据结构。

首先明确两个概念：**正排索引** 和 **倒排索引**。正派索引是根据文档（文档id）找到文档的value, 倒排索引是拿着文档的value找到对应的文档（文档id）。

![](/assets/2024/06/22/index.png)


# Inverted Index

ES中根据不同的字段类型和查询方式， 底层会使用不同的数据结构进行倒排索引存储，这些索引存在于内存中。

字段类型：

| 类型类别                         | 类型                                                                                                                                |
|------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| Common types                 | binary, **boolean**, **Keywords**(keyword constant_keyword wildcard), **Numbers**(long double), **Dates**(date date_nanos), alias |
| Objects and relational types | object, flattened, nested, join                                                                                                   |
| Structured data types        | Range(long_range double_range date_range ip_range), ip, version, murmur3                                                          |
| Aggregate data types         | aggregate_metric_double, histogram                                                                                                |
| Text search types            | **text**, match_only_text, annotated_text, completion, search_as_you_type, semantic_text, token_count                             |
| Document ranking types       | dense_vector, sparse_vector, rank_feature, rank_features                                                                          |
| Spatial data types           | geo_point, geo_shape, point shape                                                                                                 |
| Other types                  | **Arrays**, Multi-fields                                                                                                          |


查询方式：

查询Context可分为：
- Query Context : `How well does this document match this query clause?` 查询结果根据 relevance score 排序； 
- Filter Context : `Does this document match this query clause?` 不参与打分，且结果会被缓存。

| 查询类别                | 类型                                                                                                                                      |
|---------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| Compound queries    | bool(must, filter, should, must_not), boosting, constant_score, dis_max, function_max                                                   |
| Full text queries   | intervals, match, match_bool_prefix, match_phrase, match_phrase_prefix, multi_match, combined_fields, query_string, simple_query_string |
| Geo queries         | geo_bounding_box, geo_distance, geo_grid, geo_polygon, geo_shape                                                                        |
| Shape queries       | shape                                                                                                                                   |
| Joining queries     | nested, has_child has_parent                                                                                                            |
| Match all           | match_all match_none                                                                                                                    |
| Span queries        | span_containing, span_field_masking, span_first, span_multi, span_near, span_not, span_or, span_term, span_within                       |
| Specialized queries | distance_feature, more_like_this, percolate, knn, rank_feature, script, script_score, wrapper, pinned, rule                             |
| Term-level queries  | exists, fuzzy, ids, prefix, range, regexp, term, terms, terms_set, wildcard                                                             |
| Other               | text_expansion, minimum_should_match, regexp, query_string                                                                              |


- Keyword/Text: Term Index + Term Dictionary + Posting List
- Numeric: KBD Tree

[ES中的FST数据结构](https://juejin.cn/post/7244335987576602680)

## Term Index + Term Dictionary + Posting List

![](/assets/2024/06/22/inverted_index.png)



### FST


## KBD Tree

# Forward Index

## Doc values



