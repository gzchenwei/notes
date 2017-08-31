elasticsearch

### 索引

索引是指向一个或者多个物理分片的逻辑命名空间

### 分片

分片是一个底层的工作单元，保存了全部数据的一部分，

一个分片是一个Lucene的实例，同时本身就是一个完整的搜索引擎

es利用分片将数据发往集群内部。

分片是数据的容器，文档保存在分片内，分片被分配到集群的各个节点

一个分片可以是主分片或者是副本分片，索引内任意的一个文档归属于一个主分片，主分片的数目决定索引能够保存的最大数据量

索引建立的时候已经确定了主分片数量，但是副本分片可以随时修改

**从技术上来说，一个主分片最多可以存储128个文档**


### 扩展

默认主分片为5

number_of_replicas：1 设置复制分片的数量1

以上的配置，那么就是5个主分片，5个复制分片，那么最多可以扩展到10个nodes

如果需要扩展到更多的nodes，那么需要修改number_of_replicas的数量


### 故障转移

如果其中一个node故障（假定为master节点）,那么会发生以下事件

+ 选取新主节点
+ 由于故障节点存在主分片，新的主节点会将其他node的副本提升为主分片

### 文档

es中的文档是指最顶层或者根对象，这个根对象被序列化为JSON对象并存储在es，指定了唯一的ID

* 元数据
   + _index

      文档存放位置
   + _type

      文档表示的对象类别
   + _id

      文档的唯一标识

#### 获取文档

默认情况下，GET请求会返回整个文档，如果我们只对其中的部分字段感兴趣，单个字段能用_source参数获得，多个字段能通过逗号分割的列表制定

```
GET /website/blog/123?_source=title,text
```

如果只想得到_source字段，不需要任何元数据
```
GET /website/blog/123/_source
```

#### 部分更新文档

update API可以部分更新文档

* update 请求最简单的一种形式是接收文档的一部分作为 doc 的参数

```
POST /website/blog/1/_update
{
   "doc" : {
      "tags" : [ "testing" ],
      "views": 0
   }
}
```

给文档添加2个字段

* 使用脚本部分更新文档

脚本可以在update API中改变_source的字段内容，它在更新脚本中成为ctx._source

```
POST /website/blog/1/_update
{
   "script" : "ctx._source.views+=1"
}
```

* 通过tags数组添加一个新的标签

```
POST /website/blog/1/_update
{
   "script" : "ctx._source.tags+=new_tag",
   "params" : {
      "new_tag" : "search"
   }
}
```

* 设置ctx.op为delete删除基于内容的文档

```
POST /website/blog/1/_update
{
   "script" : "ctx.op = ctx._source.views == count ? 'delete' : 'none'",
    "params" : {
        "count": 1
    }
}
```

* 更新的文档不存在

upsert 参数，制定文档不存在则创建

```
POST /website/pageviews/1/_update
{
   "script" : "ctx._source.views+=1",
   "upsert": {
       "views": 1
   }
}
```

第一次运行是，upsert作为新文档被索引 ，初始化views为1，后续的运行中，文档存在，script更新操作替代upsert应用

* 更新和冲突

如果
retry_on_conflict 设置失败之前重试的次数

* 取回多个文档

mget api将多个检索请求放到一个请求中，可以加快速度

多个请求中，如果其中一个未找到，并不妨碍其他文档被检索到，同时，请求的http状态码仍然为200，如果需要查看某个文档是否查找到，需要检查found标记

* 较小的代价批量操作

mget允许批量取回多个文档，bulk api允许在单个步骤中进行多次的create\index\update\delete请求
格式
{action:{metadata}}\n
{request body }\n
{action:{metadata}}\n
{request body}\n

默认配置，http的请求长度应该是不能超出100M，如果超出则需要调整http.max_content_length设置

一般建议bulk请求体的大小在15Mb，这里的15M，仅仅是指请求体的字节数，而不是bulk size，

bulk size 一般指数据的条目数

### 路由一个文档到一个分片

当索引一个文档时，文档会被存储在一个主分片中，决定文档存储在那个分片的公式

```
shard=hash(routing) % number_of_primary_shards
```
* routing是可变值，默认是文档的_id,也可以自定义。分片分布在0 -
* nuber_of_primary_shards-1之间
* 主分片的数量不能更改，更改后无法找到文档

所有的文档api都可以接受routing的路由参数，通过这个参数可以自定义文档到分片的映射，一个自定义的路由参数可以用来确保所有相关的文档--例如同一个用户的文档--被存储在同一分片

### 主分片和副本分片交互

可以将请求发送到集群的任意节点，每个节点都有能力处理请求，每个节点知道集群中任意文档位置,接收到请求的节点称为 **协调节点**


**发送请求的时候，为了负载均衡，更好的做法是轮训集群中所有节点**

* 新建、索引、删除文档

客户端 -->  协调节点 --> 主节点 --> 复制节点 --> 主节点 --> 协调节点 --> 客户端


![new](op.jpg)


* consistency

默认设置下，在执行写操作前，主分片都要求必须有规定数量的分片副本处于活跃可用状态，才会去执行写操作

规定数量公式：

```
int(primary + number_of_replicas)/2) + 1
```

consistency可以设置为one、all、quorum默认值为quorum

** 新索引默认1个副本分片，这意味需要有2个活动的分片副本，但是默认的设置会阻止只有单一节点的情况，为了避免这个问题，只有number_of_replicas>1 规定才会执行**

* 局部更新文档

![update](update.jpg)
    +  客户端向node 1 发送更新请求
    +  请求转发到主分片node 3
    +  node 3 从主分片检索文档，修改_source中的JSON，并且尝试重新索引主分片的文档，如果文档被另一个进程修改，会重试步骤3，超过retry_on_conflict后放弃
    +  如果node 3 成功更新文档，将新版本的文档并行转发node1和node2，重新建立索引，所有分片返回成功，node3向协调节点返回成功，协调节点向客户端返回成功

* 多文档模式

![multidoc](multidoc.jpg)

    + mget 获取多个文档
        - 客户端向node 1 发送mget请求
        - node 1为每个分片构建多文档获取请求，然后并行转发到主分片或者副本分片，一旦都到所有答复，node 1 构建响应并将响应返回给客户端
    + bulk 修改多个文档
        - 客户端向node 1 发送bulk请求
        - node 1 为每个节点创建一个批量请求，将这些请求并行煮饭给每个包含主分片的节点
        - 主分片顺序执行每个操作，操作成功，主分片并行转发新文档到副本分片，然后执行下一个操作，一旦所以的副本分片报告所有操作成功，该节点将向协调节点报告成功，协调节点收集整理响应并返回客户端

### 搜索

文档中的每个字段都将被索引并且可以查询

#### 分析（analysis）与分析器
分析包含下面的过程：
  + 将文本分成合适倒排索引的独立的词条
  + 将词条统一化为标准格式以提高他们的搜索性
分析器执行上面的工作，实际上是将三个功能封装
  + 字符过滤器
  + 分词器
  + token 过滤器

#### 映射
为了将时间域视为时间，数字域视为数字，字符串视为全文或者精确字符串，es需要知道每个域中数据的类型，这个信息包含在映射中

es支持以下域类型
  * 字符串： string
  * 整数： byte, short, integer, long
  * 浮点数: float, double
  * 日期： date

* 查看映射
/_mapping 查看es在一个或者多个索引中的一个或者多个类型的映射

当索引一个包含新域的文档--es会使用动态映射，通过JSON中基本数据类型，尝试猜测域类型

* 自定义域映射
自定义映射允许执行下面的操作
  + 全文字符串域和精确值字符串域的区别
  + 使用特定语言分析器
  + 优化域以适应部分匹配
  + 指定自定义数据格式
  + 还有更多
默认，string类型域会被认为包含全文，就是说，他们的值在索引前，会通过一个分析器，针对这个域的查询在搜索前也会经过一个分析器
string域映射的2个最重要的属性是**index** **analyzer**

* index
index属性控制怎样索引字符串，可以是下面三个值
  + analyzed（默认值）  首先分析字符串，然后索引它，换句话，以全文索引这个域
  + not_analyzed 索引这个域，所以他能够被搜索，但索引的是精确值
  + no 不索引这个域，这个域不会被索引到

* analyzer
默认es使用standard分析器，可以使用一个内置的分析器替代它
```
{
   “tweet”： {
        ”type”： “string”，
        ”analyzer”： “english”
   }
}
```

* 更新映射

首次创建一个索引的时候，可以指定类型的映射，你也可以使用_mapping为新类型增加映射
**尽管可以增加一个存在的映射，但是不能修改存在的映射，如果映射已经存在，那么该域的数据可能已经被索引，如果试图修改，则可能导致索引的数据出错

* 新增加映射
给tweet增加一个名为tag的文本域

```
  PUT /gb/_mapping/tweet
{
  "properties" : {
    "tag" : {
      "type" :    "string",
      "index":    "not_analyzed"
    }
  }
}
```

### 查询表达式

要使用查询表达式，只需要将查询语句传递给query参数
```
GET /_search
{
    "query": YOUR_QUERY_HERE
}

GET /_search
{
    "query": {
        "match_all": {}
    }
}
```
查询语句的结构
```
{
    QUERY_NAME: {
        ARGUMENT: VALUE,
        ARGUMENT: VALUE,...
    }
}
```
如果针对某个字段
```
{
    QUERY_NAME: {
        FIELD_NAME: {
            ARGUMENT: VALUE,
            ARGUMENT: VALUE,...
        }
    }
}
```

### 查询与过滤

es的QUERY DSL 含有一套查询组件，这套组件可以在以下2种情况使用：filtering context / query context
当使用filtering context时，查询被设置为不评分或者过滤查询，即这个查询只是简单的问，文档是否匹配，回答只是yes or no
当使用query context,那么查询就变成一个评分的查询，需要判断匹配度

### 查询与过滤的区别
* 过滤的速度快，查询慢
* 过滤会缓存结果到内存，查询不会缓存
* 过滤的目标是减少需要经过评分查询进行检查的文档

### 重要的查询

#### match 查询

无论是任何字段上进行的全文搜索还是精确查询，match是可用标准查询

#### multi_match查询

可以在多个字段上执行相同的match查询
```
{
    "multi_match": {
        "query":    "full text search",
        "fields":   [ "title", "body" ]
    }
}
```
#### range查询
查询在制定区间内的数字或者时间
```
{
    "range": {
        "age": {
            "gte":  20,
            "lt":   30
        }
    }
}
```
#### term查询
用于精确值匹配，这些精确值可能是数字、时间、布尔或者那些 not_analyzed 的字符串
```
{ "term": { "age":    26           }}
{ "term": { "date":   "2014-09-01" }}
{ "term": { "public": true         }}
{ "term": { "tag":    "full_text"  }}
```
#### terms查询
terms 查询和 term 查询一样，但它允许你指定多值进行匹配。如果这个字段包含了指定值中的任何一个值，那么这个文档满足条件
```
{ "terms": { "tag": [ "search", "full_text", "nosql" ] }}
```

### 组合多查询
bool查询可以将多查询组合一起，接受以下参数

* must
      文档必须匹配条件
* must_not
      文档必须不匹配
* should
      如果满足语句中的任意语句，将增加_score
* filter
      必须匹配，但不评分

### 验证查询
```
GET /gb/tweet/_validate/query?explain
```

### 索引管理

#### 创建一个索引
```
PUT /my_index
{
  "settings": { ... any settings ...}
  "mappings": {
    "type_one": { ... any mappings ...}
    "type_two": { ... any mappings ...}
    ...
  }
}
```
#### 删除索引
* 删除一个索引
```
delete /my_index
```
删除多个索引
```
delete /index_one,index_two
delete /index_*
delete /_all
delete /*
```
#### 索引设置
下面是2个最重要的设置
number_of_primary_shards
    每个索引的主分片数，默认为5，索引创建后不能修改
number_of_replicas
    每个主分片的副本数，默认为1，可以随时修改
```
PUT /my_temp_index
{
    "settings": {
        "number_of_shards" :   1,
        "number_of_replicas" : 0
    }
}

PUT /my_temp_index/_settings
{
    "number_of_replicas": 1
}
```
### 配置分析器

#### 自定义分析器
分析器就是一个包里面组合了三种函数的一个包装器，三种函数按照顺序执行

字符过滤器/分词器/词单元过滤器

#### 创建一个自定义分析器
```
PUT /my_index
{
    "settings": {
        "analysis": {
            "char_filter": { ... custom character filters ... },
            "tokenizer":   { ...    custom tokenizers     ... },
            "filter":      { ...   custom token filters   ... },
            "analyzer":    { ...    custom analyzers      ... }
        }
    }
}
```
示例：
```
* 使用html清除字符过滤器移除html部分
* 使用一个自定义的映射字符过滤器把&替换为"和":
"char_filter": {
    "&_to_and": {
        "type":       "mapping",
        "mappings": [ "&=> and "]
    }
}
```
* 使用标准分词器分词
* 小写词条，使用小写过滤器处理
* 使用自定义停止词过滤器移除自定义的停止次列表中包含的词
```
"filter": {
    "my_stopwords": {
        "type":        "stop",
        "stopwords": [ "the", "a" ]
    }
}
```

分析器定义用之前已经设置好的自定义过滤器组合
```
"analyzer": {
    "my_analyzer": {
        "type":           "custom",
        "char_filter":  [ "html_strip", "&_to_and" ],
        "tokenizer":      "standard",
        "filter":       [ "lowercase", "my_stopwords" ]
    }
}
```
完整的创建索引请求如下：
```
PUT /my_index
{
    "settings": {
        "analysis": {
            "char_filter": {
                "&_to_and": {
                    "type":       "mapping",
                    "mappings": [ "&=> and "]
            }},
            "filter": {
                "my_stopwords": {
                    "type":       "stop",
                    "stopwords": [ "the", "a" ]
            }},
            "analyzer": {
                "my_analyzer": {
                    "type":         "custom",
                    "char_filter":  [ "html_strip", "&_to_and" ],
                    "tokenizer":    "standard",
                    "filter":       [ "lowercase", "my_stopwords" ]
            }}
}}}
```
### 类型和映射

从技术上讲，多个类型可以在相同的索引中，只要他们的字段不冲突
类型可以很好的区分同一个集合中的不同细分，在不同的细分中数据的整体模式是相同的

如果有2个不同的类型，每个类型都有同名的字段，但映射不同，es将不会允许定义这个映射

每个lucene索引中的所有字段包含一个单一、扁平的模式，一个特定字段可以映射成string类型也可以是number，但是不能兼具


以上简单的说就是在一个索引里面，不同的type里面，同一个名称字段不能映射不同的类型

以下是具体的案例，从下面的映射看，在lucense里面，会将json数据映射到一个扁平的模式里面，在这个模式里面是没有type的
```
{
   "data": {
      "mappings": {
         "people": {
            "properties": {
               "name": {
                  "type": "string",
               },
               "address": {
                  "type": "string"
               }
            }
         },
         "transactions": {
            "properties": {
               "timestamp": {
                  "type": "date",
                  "format": "strict_date_optional_time"
               },
               "message": {
                  "type": "string"
               }
            }
         }
      }
   }
}
```
```
{
   "data": {
      "mappings": {
        "_type": {
          "type": "string",
          "index": "not_analyzed"
        },
        "name": {
          "type": "string"
        }
        "address": {
          "type": "string"
        }
        "timestamp": {
          "type": "long"
        }
        "message": {
          "type": "string"
        }
      }
   }
}
```

#### 根对象
映射的最高一层被称为根对象，它可能包含以下几项
* 一个properties节点，列出文档中可能包含的每个字段的映射
* 各种元数据字段
* 设置项，控制如何动态处理新的字段，例如 analyzer、dynamic_data_formats和dynamic_templates
* 其他设置，可以同时应用在根对象和其他类型的字段上例如enabled、dynamic和include_in_all

#### 属性
* type
字段的数据类型，例如string或date
* index
字段是否被当成全文来搜索(analyzed),或被当成一个准确的值（not_analyzed)，还是完全不可搜索（no）
* analyzer
确定在索引和搜索时全文字段使用的analyer

#### 元数据:_source字段
默认，es在_source字段存储达标文档体的json字符串，和所有被存储的字段一样，在被写入磁盘前会压缩
这个字段的存储几乎总是我们想要的，因为它意味着下面的这些：

* 搜索结果包括了整个可用的文档——不需要额外的从另一个的数据仓库来取文档。
* 如果没有 _source 字段，部分 update 请求不会生效。
* 当你的映射改变时，你需要重新索引你的数据，有了_source字段你可以直接从Elasticsearch这样做，而不必从另一个（通常是速度更慢的）数据仓库取回你的所有文档。
* 当你不需要看到整个文档时，单个字段可以从 _source 字段提取和通过 get 或者 search 请求返回。
* 调试查询语句更加简单，因为你可以直接看到每个文档包括什么，而不是从一列id猜测它们的内容。

#### 元数据： _all字段
把其他字段值当做一个大字符串来索引的特殊字段
如果查询不指定字段则使用_all
_all是一个仅仅经过分词的string字段，使用默认的分词器来分析
如果不需要all字段
```
PUT /my_index/_mapping/my_type
{
    "my_type": {
        "_all": { "enabled": false }
    }
}
```
可以为_all配置分词器
```
PUT /my_index/my_type/_mapping
{
    "my_type": {
        "_all": { "analyzer": "whitespace" }
    }
}
```

#### 元数据： 文档标识

默认情况下,
_uid 字段是存储和索引的
_type字段被索引的但是但是没有存储
_id和i_index字段既不索引也不存储，意味着他们并不是真实存在的

### 分片内部原理

#### 倒排索引的不变性

倒排索引在被写入磁盘后是不可改变的，带来的优点：
* 不需要锁
* 一旦索引被读入文件缓存，只要缓存有足够的空间，就会一直使用缓存
* 允许数据被压缩
#### 倒排索引动态更新

如何解决在保持不变的前提下实现倒排索引的更新，答案是用更多的索引
通过增加新的补充索引来反映最近的修改，而不是直接重写整个倒排索引，每个索引会被轮询，查询完后再对结果进行合并

es基于lucene，lucene引入了按段搜索的概念，每一段都是一个倒排索引，索引在lucene中除表示段的集合外，还增加了提交点的概念

容易被混淆的概念是
一个lucene索引在es被称为分片，一个es的索引是分片的集合

逐段搜索流程：
* 新文档被收集到内存索引缓存
* 不时，缓存被提交
   + 一个新的段被写入磁盘
   + 一个新的包含新段名字的提交点被写入磁盘
   + 磁盘同步---所有在文件系统缓存中等待的写入都刷新到磁盘，以确保写入物理文件
* 新的端被开启，包含的文档可被搜索
* 内存缓存被清空，等待接收新文档

#### 持久化变更
* 当一个文档被索引，被加入到内存缓存，同时加入到translog

* 以每s的频率将buff的文档写入到文件系统缓存（段），但是没有fsync
    - 段被打开，新的文档可以被搜索
    - 文件系统缓存被清除

* 更多的文档被加入到缓存，写入日志，过程会继续
* 不时地（默认30分钟，或者日志超过512MB)，日志很大，新的日志被创建，进行一次全提交
    - 内存缓存区的所有文档会写入到新段
    - 清除缓存
    - 一个提交点写入硬盘
    - 文件系统fsync到硬盘
    - translog被清除
