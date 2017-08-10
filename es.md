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

### 获取文档

默认情况下，GET请求会返回整个文档，如果我们只对其中的部分字段感兴趣，单个字段能用_source参数获得，多个字段能通过逗号分割的列表制定

```
GET /website/blog/123?_source=title,text
```

如果只想得到_source字段，不需要任何元数据
```
GET /website/blog/123/_source
```

### 部分更新文档

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

* 分析（analysis）与分析器
分析包含下面的过程：
  + 将文本分成合适倒排索引的独立的词条
  + 将词条统一化为标准格式以提高他们的搜索性
分析器执行上面的工作，实际上是将三个功能封装
  + 字符过滤器
  + 分词器
  + token 过滤器

* 映射
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
<<<<<<< HEAD

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
```
