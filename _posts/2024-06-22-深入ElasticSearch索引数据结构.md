---
title: ElasticSearch索引架构与存储数据结构
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
2. 复杂查询：如电商系统的商品检索、github站内检索；
3. 作为冗余数据提供多样化查询方式：如SQL分库分表无法支持非分片键字段的查询，使用ES冗余数据支持多样化查询；
4. 跨库关联查询：将跨库数据关联冗余至ES，关联查询直接查ES，同时解决了跨库分页的问题。

关于ElasticSearch的基本概念和原理可以参看：

- [分布式文档存储](https://www.elastic.co/guide/cn/elasticsearch/guide/current/distributed-docs.html)
- [ElasticSearch Docs](https://www.elastic.co/guide/en/elasticsearch/reference/current/elasticsearch-intro.html)
- [从原理到应用，Elasticsearch详解](https://segmentfault.com/a/1190000020022504)
- [Elasticsearch 教程](https://dunwu.github.io/db-tutorial/pages/74675e/)

官方讨论社区：[discuss.elastic.co](https://discuss.elastic.co/)

官方博客：[elastic.co/cn/blog](https://www.elastic.co/cn/blog)

本文介绍了ES索引的存储架构和数据结构。



## ES Index Storage Architecture

ES底层搜索使用的仍然是Lucene引擎，但Lucene只支持单节点搜索，不具备扩展性，ES在其基础上提供了分布式环境下的支持。

![](/assets/2024/06/22/architecture.png)

关于Lucene Index:

> index in Lucene means "a collection of segments plus a commit point"

关于Segment:

> A shard in Elasticsearch is a Lucene index, and a Lucene index is broken down into segments.



- [Understanding Segments in Elasticsearch](https://stackoverflow.com/questions/15426441/understanding-segments-in-elasticsearch)
- [segment、buffer和translog对实时性的影响](https://elkguide.elasticsearch.cn/elasticsearch/principle/realtime.html)
- [Near real-time search](https://www.elastic.co/guide/en/elasticsearch/reference/current/near-real-time.html)


在我们变更document时，实际上会document会生成一系列数据结构放入Segment中，放入的过程是先放入文件系统缓存（内存中）再刷到磁盘，放到文件系统缓存中即可被检索到，这个文件系统缓存刷新(refresh)间隔默认为1秒 (注意：只针对30秒内有查询的索引生效)，
所以说ES准实时。同时，会有translog保证文件写入磁盘时的数据一致性，如果刷盘期间发生故障，可以通过translog进行数据恢复，等到真正把 segment 刷到磁盘(flush)，且 commit 文件进行更新的时候， translog 文件才清空。

索引数据的一致性通过 translog 保证。那么 translog 文件刷盘本身如何保证一致性 ？ 类比MySQL等关系数据库的处理，这里肯定有同步/异步刷盘策略，
默认情况下，Elasticsearch 每 5 秒，或每次每次 index、bulk、delete、update结束前，会强制刷新 translog 日志到磁盘上。

索引查询时会依次检索各个Segment，同时后台也会有线程执行Segment合并动作，也可以手动执行强制合并，提升Segment检索效率。
一般索引会按时间周期性新建，老的索引不再写入，这些不再写入的索引可以进行强制段合并操作，提升查询性能(一个Shard中多个Segment是串行查询的)。

- [ElasticSearch中使用Index Template、Index Alias和Rollover 维护索引](https://bty834.github.io/posts/ES%E6%BB%9A%E5%8A%A8%E7%B4%A2%E5%BC%95/)
- [segment merge对写入性能的影响](https://elkguide.elasticsearch.cn/elasticsearch/principle/indexing-performance.html)
- [段合并优化及注意事项](https://learnku.com/articles/41593)


## ES Index

首先明确两个概念：**正排索引** 和 **倒排索引**。正派索引是根据文档（文档id）找到文档的value, 倒排索引是拿着文档的value找到对应的文档（文档id）。

![](/assets/2024/06/22/index.png)


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

查询Context可分为：
- Query Context : `How well does this document match this query clause?` 查询结果根据 relevance score 排序；
- Filter Context : `Does this document match this query clause?` 不参与打分，且结果会被缓存。

查询方式：

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

### Segment存储

Segment中存储了以下内容：

- [Segment info](https://lucene.apache.org/core/5_1_0/core/org/apache/lucene/codecs/lucene50/Lucene50SegmentInfoFormat.html): This contains metadata about a segment, such as the number of documents, what files it uses,
- [Field names](https://lucene.apache.org/core/5_1_0/core/org/apache/lucene/codecs/lucene50/Lucene50FieldInfosFormat.html): This contains the set of field names used in the index.
- [Stored Field values](https://lucene.apache.org/core/5_1_0/core/org/apache/lucene/codecs/lucene50/Lucene50StoredFieldsFormat.html): This contains, for each document, a list of attribute-value pairs, where the attributes are field names. These are used to store auxiliary information about the document, such as its title, url, or an identifier to access a database. The set of stored fields are what is returned for each hit when searching. This is keyed by document number.
- [Term dictionary](https://lucene.apache.org/core/5_1_0/core/org/apache/lucene/codecs/lucene50/Lucene50PostingsFormat.html): A dictionary containing all the terms used in all the indexed fields of all the documents. The dictionary also contains the number of documents which contain the term, and pointers to the term's frequency and proximity data.
- [Term Frequency data](https://lucene.apache.org/core/5_1_0/core/org/apache/lucene/codecs/lucene50/Lucene50PostingsFormat.html): For each term in the dictionary, the numbers of all the documents that contain that term, and the frequency of the term in that document, unless frequencies are omitted (IndexOptions.DOCS_ONLY)
- [Term Proximity data](https://lucene.apache.org/core/5_1_0/core/org/apache/lucene/codecs/lucene50/Lucene50PostingsFormat.html): For each term in the dictionary, the positions that the term occurs in each document. Note that this will not exist if all fields in all documents omit position data.
- [Normalization factors](https://lucene.apache.org/core/5_1_0/core/org/apache/lucene/codecs/lucene50/Lucene50NormsFormat.html): For each field in each document, a value is stored that is multiplied into the score for hits on that field.
- [Term Vectors](https://lucene.apache.org/core/5_1_0/core/org/apache/lucene/codecs/lucene50/Lucene50TermVectorsFormat.html): For each field in each document, the term vector (sometimes called document vector) may be stored. A term vector consists of term text and term frequency. To add Term Vectors to your index see the Field constructors
- [Per-document values](https://lucene.apache.org/core/5_1_0/core/org/apache/lucene/codecs/lucene50/Lucene50DocValuesFormat.html): Like stored values, these are also keyed by document number, but are generally intended to be loaded into main memory for fast access. Whereas stored values are generally intended for summary results from searches, per-document values are useful for things like scoring factors.
- [Live documents](https://lucene.apache.org/core/5_1_0/core/org/apache/lucene/codecs/lucene50/Lucene50LiveDocsFormat.html): An optional file indicating which documents are live.

对应存储的文件如下：

![](/assets/2024/06/22/files.png)

可以参看：
- [A Dive into the Elasticsearch Storage](https://www.elastic.co/cn/blog/found-dive-into-elasticsearch-storage)
- [Apache Lucene - Index File Formats](https://lucene.apache.org/core/5_1_0/core/org/apache/lucene/codecs/lucene50/package-summary.html#package_description)


### Inverted Index

#### Term Index + Term Dictionary + Posting List

[美团外卖搜索基于Elasticsearch的优化实践
](https://tech.meituan.com/2022/11/17/elasicsearch-optimization-practice-based-on-run-length-encoding.html)


输入查询多个Terms，会对每个Term依次从 Term Index -> Term Dictionary -> Posting List 找到



#### FST


#### KBD Tree

### Forward Index

#### Doc values



