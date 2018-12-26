---
layout: post
title: elastic-search学习
category: 技术
tags: es
keywords:
description:
---

# 官方文档

https://www.elastic.co/guide/cn/elasticsearch/guide/current/preface.html


# es 中的概念和 关系型数据库中概念的对比

Relational DB -> Databases -> Tables -> Rows -> Columns
Elasticsearch -> Indices   -> Types  -> Documents -> Fields

# 机器扩展

分片就是一个Lucene实例，并且它本身就是一个完整的搜索引擎

列出所有分片：

        curl -XGET http://127.0.0.1:9200/_cat/indices

删除分片：
        curl -XDELETE http://127.0.0.1:9200/blogs

查询集群健康度：

        curl -XGET http://127.0.0.1:9200/_cluster/health?pretty

集群监控度 status 字段含义：

1. green	所有主要分片和复制分片都可用
2. yellow	所有主要分片可用，但不是所有复制分片都可用
3. red	不是所有的主要分片都可用


创建一个 包含3个主分片和1个复制分片(每个主分支一个复制分片)的索引

        curl -XPUT http://127.0.0.1:9200/blogs -d '
        {
           "settings" : {
              "number_of_shards" : 3,
              "number_of_replicas" : 1
           }
        }
        '

运行时修改复制分片的数量
        curl -XPUT 'http://127.0.0.1:9200/blogs/_settings?pretty' -d '
        {
              "number_of_replicas" : 10
        }
        '

# 数据

es以json作为文档存储方式，使文档更加易于扩展。

文档(document)这个术语有着特殊含义。它特指最顶层结构或者根对象(root object)序列化成的JSON数据（以唯一ID标识并存储于Elasticsearch中）。

文档元数据：
* _index: 文档存储的地方
* _type: 文档类型
* _id: 文档唯一标识符

存储一个数据(可以不指定id，让es自动生成id)：

        curl -XPUT 'http://127.0.0.1:9201/blogs/user/1?pretty' -d '{
            "name":         "jaky.wang.1",
            "age":          42,
            "confirmed":    true,
            "join_date":    "2014-06-01",
            "home": {
                "lat":      51.5,
                "lon":      0.1
            },
            "accounts": [
                {
                    "type": "facebook",
                    "id":   "johnsmith"
                },
                {
                    "type": "twitter",
                    "id":   "johnsmith"
                }
            ]
        }'


查询一部分数据:

        curl -XGET 'http://127.0.0.1:9201/blogs/user/1?_source=name,accounts&pretty'

只返回source字段

        curl -XGET 'http://127.0.0.1:9201/blogs/user/1?_source&pretty'

检查文档是否存在(只会返回http头)，存在则code为200，否则为404

        curl -i -XHEAD http://127.0.0.1:9201/blogs/user/1

使用 PUT 可更新整个文档。

使用 PUT /website/blog/123/_create 可强制指定为新建非不会覆盖，如果创建成功http code 返回201，否则返回409

        curl -XPUT 'http://127.0.0.1:9201/blogs/user/1/_create' -d '{
            "name":         "John Smith",
            "age":          42,
            "confirmed":    true,
            "join_date":    "2014-06-01",
            "home": {
                "lat":      51.5,
                "lon":      0.1
            },
            "accounts": [
                {
                    "type": "facebook",
                    "id":   "johnsmith"
                },
                {
                    "type": "twitter",
                    "id":   "johnsmith"
                }
            ]
        }'

删除文档：

        curl -XDELETE http://127.0.0.1:9201/blogs/user/1

es使用乐观锁进行并发控制，可指定版本号进行更新：
        curl -XPUT 'http://127.0.0.1:9201/blogs/user/1?version=1' -d '{
            "name":         "John Smith",
            "age":          42,
            "confirmed":    true,
            "join_date":    "2014-06-01",
            "home": {
                "lat":      51.5,
                "lon":      0.1
            },
            "accounts": [
                {
                    "type": "facebook",
                    "id":   "johnsmith"
                },
                {
                    "type": "twitter",
                    "id":   "johnsmith"
                }
            ]
        }'

可通过如下方式执行指定version版本        

        PUT /website/blog/2?version=5&version_type=external
        {
          "title": "My first external blog entry",
          "text":  "Starting to get the hang of this..."
        }

外部版本号与之前说的内部版本号在处理的时候有些不同。它不再检查_version是否与请求中指定的一致，而是检查是否小于指定的版本。如果请求成功，外部版本号就会被存储到_version中


使用 _update 进行更新(内部也是 检索-修改-重建索引 的过程), 有name字段进行更新，views之前没有，会新增该字段     

        curl -XPOST 'http://127.0.0.1:9201/blogs/user/1/_update?pretty' -d '
        {
           "doc" : {
              "name" : [ "jaky.wang" ],
              "views": 0
           }
        }
        '

使用 _mget 合并多个查询请求进行查询, 可在指定 /_index 或者/_index/_type 进行查询，多个查询之间互相隔离，互不影响


          curl -XPOST 'http://127.0.0.1:9201/_mget?pretty' -d '
          {
             "docs" : [
               {
                 "_index" : "blogs",
                 "_type" : "user",
                 "_id" : 1,
                 "age" : 42,
                 "views": 0
               },
               {
                   "_index" : "website",
                   "_type" :  "pageviews",
                   "_id" :    1,
                   "_source": "views"
                }
             ]
          }
          '

使用 _bulk 实现单一请求来处理多个文档的create、index、update或delete.(消息体最后一行必须有一个换行)          


_bulk 的请求体如下(delete 这种action没有 request body)：

          { action: { metadata }}\n
          { request body        }\n
          { action: { metadata }}\n
          { request body        }\n

例如：

          curl -XPOST 'http://127.0.0.1:9201/_bulk?pretty' -d '
          { "create":  { "_index": "website", "_type": "blog", "_id": "123" }}
          { "title":    "My first blog post" }

          '

Elasticsearch是从网络缓冲区中一行一行的直接读取数据。它使用换行符识别和解析action/metadata行，以决定哪些分片来处理这个请求。这些行请求直接转发到对应的分片上。这些没有冗余复制，没有多余的数据结构。整个请求过程使用最小的内存在进行。从而去掉了解析json，构建json的过程。

# 分布式增删改查

路由算法：

          hash(routing) % number_of_primary_shards

routing默认是_id,可配置。由于采用取余的方式选项分片，这也解释了为什么主分片的数量只能在创建索引时定义且不能修改：如果主分片的数量在未来改变了，所有先前的路由值就失效了，文档也就永远找不到了。


主分片在写入时，需要规定数量(quorum)或过半的分片（可以是主节点或复制节点）可用。这是防止数据被写入到错的网络分区。规定的数量计算公式如下：

        int( (primary + number_of_replicas) / 2 ) + 1

注意number_of_replicas是在索引中的的设置，用来定义复制分片的数量，而不是现在活动的复制节点的数量。如果你定义了索引有3个复制节点，那规定数量是：

        int( (primary + 3 replicas) / 2 ) + 1 = 3

但如果你只有2个节点，那你的活动分片不够规定数量，也就不能索引或删除任何文档。        

*但是我在本机尝试启动了两个节点，将 number_of_replicas 设置为了10，依然可以正常写入或者删除文档 ？*


写请求只能在主分片上进行，读请求会均匀的分发到含有对应分片的节点上。写请求默认是同步的，即所有分片都返回写成功之后，才会给客户端返回成功(可配置为异步，不建议使用异步)

可能的情况是，一个被索引的文档已经存在于主分片上却还没来得及同步到复制分片上。这时复制分片会报告文档未找到，主分片会成功返回文档。一旦索引请求成功返回给用户，文档则在主分片和复制分片都是可用的。

当主分片转发更改给复制分片时，并不是转发更新请求，而是转发整个文档的新版本。记住这些修改转发到复制节点是异步的，它们并不能保证到达的顺序与发送相同。如果Elasticsearch转发的仅仅是修改请求，修改的顺序可能是错误的，那得到的就是个损坏的文档。


# 搜索

        GET /_search?timeout=10ms

timeout 不会停止执行查询，它仅仅告诉你目前顺利返回结果的节点然后关闭连接。在后台，其他分片可能依旧执行查询，尽管结果已经被发送。

分页查询：

        GET /_search?size=5&from=10

现在假设我们请求第1000页——结果10001到10010。工作方式都相同，不同的是每个分片都必须产生顶端的10010个结果。然后请求节点排序这50050个结果并丢弃50040个！


* 映射(Mapping)	数据在每个字段中的解释说明，将每个字段匹配为一种确定的数据类型(string, number, booleans, date等)。
* 分析(Analysis)	全文是如何处理的可以被搜索的，进行全文文本(Full Text)的分词，以建立供搜索用的反向索引。
* 领域特定语言查询(Query DSL)	Elasticsearch使用的灵活的、强大的查询语言        

Elasticsearch使用一种叫做倒排索引(inverted index)的结构来做快速的全文搜索。倒排索引由在文档中出现的唯一的单词列表，以及对于每个单词在文档中的位置组成。


分析器包含：
* 字符过滤器(去除html标签，将 "&" 转成 "and" 等)
* 分词器
* 标记过滤(大小写转换，近义词添加等)

属性的index参数控制字符串以何种方式被索引。它包含以下三个值当中的一个：

* analyzed	首先分析这个字符串，然后索引。换言之，以全文形式索引此字段。
* not_analyzed	索引这个字段，使之可以被搜索，但是索引内容和指定值一样。不分析此字段。
* no	不索引这个字段。这个字段不能为搜索到。

假如一个包含内部对象的数组如何索引。 我们有个数组如下所示：

        {
            "followers": [
                { "age": 35, "name": "Mary White"},
                { "age": 26, "name": "Alex Jones"},
                { "age": 19, "name": "Lisa Smith"}
            ]
        }

此文件会如我们以上所说的被扁平化，但其结果会像如此：

        {
            "followers.age":    [19, 26, 35],
            "followers.name":   [alex, jones, lisa, smith, mary, white]
        }

{age: 35}与{name: Mary White}之间的关联会消失，因每个多值的栏位会变成一个值集合，而非有序的阵列。 这让我们可以知道：

是否有26岁的追随者？

但我们无法取得准确的资料如：

是否有26岁的追随者且名字叫Alex Jones？

关联内部对象可解决此类问题，我们称之为嵌套对象，我们之後会在嵌套对象中提到它。


结构化查询(query DSL) 和 结构化过滤(filter DSL) 的差别:
* 过滤语句，询问每个文档的字段值是否包含特定值
* 查询语句，询问的是每个文档字段值与特定值的匹配程度，一条查询语句会计算每个文档与查询语句的相关性，会给出一个相关性评分 _score，并且 按照相关性对匹配到的文档进行排序。 这种评分方式非常适用于一个没有完全配置结果的全文本搜索。

过滤语句可以缓存，查询语句没法缓存，幸亏有了倒排索引，一个只匹配少量文档的简单查询语句在百万级文档中的查询效率会与一条经过缓存 的过滤语句旗鼓相当，甚至略占上风。 但是一般情况下，一条经过缓存的过滤查询要远胜一条查询语句的执行效率。

原则上来说，使用查询语句做全文本搜索或其他需要进行相关性评分的时候，剩下的全部用过滤语句

一些查询过滤语句：
* term 过滤
* terms 过滤
* range 过滤
* exists 和 missing 过滤
* bool 过滤
* match_all 查询
* match 查询
* multi_match 查询
* bool 查询，bool 查询与 bool 过滤相似，用于合并多个查询子句。不同的是，bool 过滤可以直接给出是否匹配成功， 而bool 查询要计算每一个查询子句的 _score （相关性分值）。

查询语句和过滤语句一起使用

          GET /_search
          {
              "query": {
                  "filtered": {
                      "query":  { "match": { "email": "business opportunity" }},
                      "filter": { "term": { "folder": "inbox" }}
                  }
              }
          }

验证查询语句的合法性

          GET /gb/tweet/_validate/query
          {
             "query": {
                "tweet" : {
                   "match" : "really powerful"
                }
             }
          }          

理解查询语句，类似于mysql的explain

        GET /_validate/query?explain
        {
           "query": {
              "match" : {
                 "tweet" : "really powerful"
              }
           }
        }


# 排序

指定排序字段

          GET /_search
          {
              "query" : {
                  "filtered" : {
                      "filter" : { "term" : { "user_id" : 1 }}
                  }
              },
              "sort": { "date": { "order": "desc" }}
          }

多级排序

          GET /_search
          {
              "query" : {
                  "filtered" : {
                      "query":   { "match": { "tweet": "manage text search" }},
                      "filter" : { "term" : { "user_id" : 2 }}
                  }
              },
              "sort": [
                  { "date":   { "order": "desc" }},
                  { "_score": { "order": "desc" }}
              ]
          }

多值字段排序，从多个值中取出一个来进行排序，你可以使用min, max, avg 或 sum这些模式。 比说你可以在 dates 字段中用最早的日期来进行排序：

        "sort": {
            "dates": {
                "order": "asc",
                "mode":  "min"
            }
        }

字符串排序： https://es.xiaoleilu.com/056_Sorting/88_String_sorting.html

es相关性计算：
ElasticSearch的相似度算法被定义为 TF/IDF，即检索词频率/反向文档频率，包括一下内容：

检索词频率::
检索词在该字段出现的频率？出现频率越高，相关性也越高。 字段中出现过5次要比只出现过1次的相关性高。

反向文档频率::
每个检索词在索引中出现的频率？频率越高，相关性越低。 检索词出现在多数文档中会比出现在少数文档中的权重更低， 即检验一个检索词在文档中的普遍重要性。

字段长度准则::
字段的长度是多少？长度越长，相关性越低。 检索词出现在一个短的 title 要比同样的词出现在一个长的 content 字段。



# 分布式搜索的执行方式

分为两个阶段： 查询阶段 和 取回阶段

查询阶段 只会 返回包含documentID值和排序需要用到的值的数据 给协调节点，数据个数大小为 from+size。

取回阶段：在上面的查询阶段确定了需要获取哪些 documentID的数据时，再去各个分片获取对应的文档详细数据，并返回给客户端。


可使用 preference参数来控制使用哪个分片或节点来处理搜索请求。她接受如下一些参数 _primary， _primary_first， _local， _only_node:xyz， _prefer_node:xyz和_shards:2,3，可以用来避免 结果震荡(由于搜索请求是在所有有效的分片副本间轮询的，这两个document可能在原始分片里是一种顺序，在副本分片里是另一种顺序).

通过search_type来设置搜索类型（包含 count，query_and_fetch，dfs_query_then_fetch，dfs_query_and_fetch，scan）：

        GET /_search?search_type=count

* count : 查询记录条数
* query_and_fetch：将查询和取回阶段合并成一个步骤
* dfs_query_then_fetch 和 dfs_query_and_fetch dfs搜索类型有一个预查询的阶段，它会从全部相关的分片里取回项目频数来计算全局的项目频数。
* scan（扫描）搜索类型是和scroll（滚屏）API连在一起使用的，可以高效地取回巨大数量的结果。它是通过禁用排序来实现的。

通过scan and scroll 来获取大量数据： [传送门](https://es.xiaoleilu.com/060_Distributed_Search/20_Scan_and_scroll.html)

因为 Lucene 没有文档类型的概念，每个文档的类型名被储存在一个叫 _type 的元数据字段上。当我们搜索一种特殊类型的文档时，Elasticsearch 简单的通过 _type 字段来过滤出这些文档。


设置不存储 _source 字段：

_source 字段存的是文档的原始数据，如果设置为false，虽然可减小es使用的存储空间，但是会损失一些es的特性( update, update_by_query，高亮等，)，详细说明: [传送门](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-source-field.html)

          PUT /my_index
          {
              "mappings": {
                  "my_type": {
                      "_source": {
                          "enabled":  false
                      }
                  }
              }
          }

_all 字段：一个所有其他字段值的特殊字符串字段。query_string 在没有指定字段时默认用 _all 字段查询。(随着应用的发展，搜索需求会变得更加精准。你会越来越少的使用 _all 字段。_all 是一种简单粗暴的搜索方式。通过查询独立的字段，你能更灵活，强大和精准的控制搜索结果，提高相关性。)

禁用 _all:

          PUT /my_index/_mapping/my_type
          {
              "my_type": {
                  "_all": { "enabled": false }
              }
          }


文档唯一标识由四个元数据字段组成：

* _id：文档的字符串 ID
* _type：文档的类型名
* _index：文档所在的索引
* _uid：_type 和 _id 连接成的 type#id

默认情况下，_uid 是被保存（可取回）和索引（可搜索）的。_type 字段被索引但是没有保存，_id 和 _index 字段则既没有索引也没有储存，它们并不是真实存在的。
尽管如此，你仍然可以像真实字段一样查询 _id 字段。Elasticsearch 使用 _uid 字段来追溯 _id。虽然你可以修改这些字段的 index 和 store 设置，但是基本上不需要这么做。


通过 dynamic 设置来控制对未知字段的操作行为，它接受下面几个选项：
* true：自动添加字段（默认）
* false：忽略字段
* strict：当遇到未知字段时抛出异常


关闭日期检测：

          PUT /my_index
          {
              "mappings": {
                  "my_type": {
                      "date_detection": false
                  }
              }
          }

使用索引别名来解决重建索引切换：

          // 指定 my_index_v1 的别名为 my_index
          PUT /my_index_v1/_alias/my_index

# 深入分片

每一个分片中的Lucene索引都是不可变的，所以增加文档是增加新的索引，删除文档是将文档的索引分片标记为不可用，修改文档是先增加再删除。

ES默认1s刷一次Lucene的索引，使新的索引可以被搜索，所以ES是准实时的，而不是实时的。

ES通过事务日志确保数据不会丢失，在将数据持久化到磁盘(fsync)并更新提交点之后，才会删除事务日志(每30分钟，或事务日志过大会进行一次flush操作)。

由于一直新增索引段会使得查询越来越慢(每次查询都需要查询所有的索引段)，所以ES会在refresh时选择一些小的段合并成大的段(这个过程不会中断索引和搜索),并删除无用的段。


# 结构化搜索

查询准确值：
使用 term 过滤器，会指定查询等于查询条件的数据，不会计算其相关值，其他数据也不会被查询出来，但是需要注意，如果是像下面这种查询语句，如果 productID 没有设置 为 not_analyzed ，则不能查询到对应的记录，因为在默认的 analyzed 模式下，productID会被分割成 XHDK A 1293 FJ3 等多个token

          GET /my_store/products/_search
          {
              "query" : {
                  "filtered" : {
                      "filter" : {
                          "term" : {
                              "productID" : "XHDK-A-1293-#fJ3"
                          }
                      }
                  }
              }
          }


使用 bool 过滤器来做组合过滤：

下面的查询查询的数据类似于sql中的 `SELECT product FROM   products WHERE  (price = 20 OR productID = "XHDK-A-1293-#fJ3") AND  (price != 30)`

          GET /my_store/products/_search
          {
             "query" : {
                "filtered" : { <1>
                   "filter" : {
                      "bool" : {
                        "should" : [
                           { "term" : {"price" : 20}}, <2>
                           { "term" : {"productID" : "XHDK-A-1293-#fJ3"}} <2>
                        ],
                        "must_not" : {
                           "term" : {"price" : 30} <3>
                        }
                     }
                   }
                }
             }
          }

bool过滤器中：

must：所有分句都必须匹配，与 AND 相同。
must_not：所有分句都必须不匹配，与 NOT 相同。
should：至少有一个分句匹配，与 OR 相同。


使用 terms 查询多个准确值：

        {
            "terms" : {
                "price" : [20, 30]
            }
        }


注意,term 和 terms 是包含操作，而不是相等操作,所以如果你需要完全匹配这种行为，最好是通过添加另一个字段来实现。在这个字段中，你索引原字段包含值的个数。引用上面的两个文档。[传送门](https://es.xiaoleilu.com/080_Structured_Search/20_contains.html)


使用 range 做范围过滤：

          GET /my_store/products/_search
          {
              "query" : {
                  "filtered" : {
                      "filter" : {
                          "range" : {
                              "price" : {
                                  "gte" : 20,
                                  "lt"  : 40
                              }
                          }
                      }
                  }
              }
          }

可选操作：

* gt: > 大于
* lt: < 小于
* gte: >= 大于或等于
* lte: <= 小于或等于


由于 倒排索引 中不存在null，所以一个属性的值为null是，它是不存在与倒排索引中的，所以可以使用 exists 和 missing 来查询包含或者不含有某个字段的数据。

使用exists 查询包含 tags 字段的文档：

          GET /my_index/posts/_search
          {
              "query" : {
                  "filtered" : {
                      "filter" : {
                          "exists" : { "field" : "tags" }
                      }
                  }
              }
          }

使用 missing 查询不包含tags的文档：

          GET /my_index/posts/_search
          {
              "query" : {
                  "filtered" : {
                      "filter": {
                          "missing" : { "field" : "tags" }
                      }
                  }
              }
          }          


          {
             "name.first" : "John",
             "name.last"  : "Smith"
          }

由于倒排索引中都是拍平的数据，所以 像上面这种数据，如果需要检查 name 是否存在，实际执行的是下面这样查询：

        {
            "bool": {
                "should": [
                    { "exists": { "field": { "name.first" }}},
                    { "exists": { "field": { "name.last"  }}}
                ]
            }
        }

意味着假如 first 和 last 都为空，那么 name 就是不存在的。

过滤器大部分是可复用的，一次的查询结果会被缓存起来，下次再遇到相同的过滤条件，不会再重新计算一遍，但是某些过滤器是不会被缓存的，例如时间戳的过滤器。

过滤顺序：在 bool 条件中过滤器的顺序对性能有很大的影响。更详细的过滤条件应该被放置在其他过滤器之前，以便在更早的排除更多的文档。假如条件 A 匹配 1000 万个文档，而 B 只匹配 100 个文档，那么需要将 B 放在 A 前面。


使用 文档中的指定字段来作为 _id (虽然这样很方便，但是注意它对 bulk 请求（见【bulk 格式】）有个轻微的性能影响。处理请求的节点将不能仅靠解析元数据行来决定将请求分配给哪一个分片，而需要解析整个文档主体):

          PUT /my_index
          {
              "mappings": {
                  "my_type": {
                      "_id": {
                          "path": "doc_id" <1>
                      },
                      "properties": {
                          "doc_id": {
                              "type":   "string",
                              "index":  "not_analyzed"
                          }
                      }
                  }
              }
          }
