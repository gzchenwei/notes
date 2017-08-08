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















