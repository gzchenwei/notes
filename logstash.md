## logstash

### logstash 执行模型

logstash 管道在每个阶段运行自己的线程

inputs将events写入到一个通用的java 同步队列

队列holds所有的events，每个管道worker 线程 处理一批的事件

如果logstash异常终止将会丢失数据

batch的size和pipeline的worker threads是可以配置的

### logstash 目录结构

home： 主目录
bin： 二进制程序目录
settings：配置文件，包括logstash.yml，jvm.options，主要指定控制logstash启动和执行的选项
conf：logstash 管道配置文件，定义logstash 管道处理
logs: 日志文件
plugins：插件

### logstash配置文件

#### 配置文件结构

```
input {
}
filter {
}
output {
}
```
如果有多个filter，会顺序执行

#### 值类型
* 数组，大多情况下已经弃用
```
users => [ {id => 1, name => bob}, {id => 2, name => jane} ]
```
* lists
```
path => [ "/var/log/messages", "/var/log/*.log" ]
  uris => [ "http://elastic.co", "http://example.net" ]
```
* boolean
```
ssl_enable => true
```
* bytes
```
my_bytes => "1113"   # 1113 bytes
my_bytes => "10MiB"  # 10485760 bytes
my_bytes => "100kib" # 102400 bytes
my_bytes => "180 mb" # 180000000 bytes
```
* codec
```
codec => "json"
```
* hash
```
match => {
  "field1" => "value1"
  "field2" => "value2"
  ...
}
```
* number
```
port => 33
```
* password
```
my _password => "password"
```
* url/path
* string/comments
### 配置里访问event和fields

访问fileds的语法是[fieldname],如果是顶级字段，可以忽略[]，如果需要引用内嵌字段，需要指定全路径[top-level field][nested field]

#### 条件
条件语法
```
if exp {
} else if exp {
} else {
}
```

exp 包含

* equality: == != < > <= >=
* regexp: =~,!~
* inclusion: in, not in
* boolean: and or nand xor
* unary op: !

#### @metadata字段

1.5版本后增加了@metadata字段，这个字段不包含在events里面
```
input { stdin { } }

filter {
  mutate { add_field => { "show" => "This data will be in the output" } }
  mutate { add_field => { "[@metadata][test]" => "Hello" } }
  mutate { add_field => { "[@metadata][no_show]" => "This data will not be in the output" } }
}

output {
  if [@metadata][test] == "Hello" {
    stdout { codec => rubydebug }
  }
}
```

输出如下：
```
$ bin/logstash -f ../test.conf
Pipeline main started
asdf
{
    "@timestamp" => 2016-06-30T02:42:51.496Z,
      "@version" => "1",
          "host" => "example.com",
          "show" => "This data will be in the output",
       "message" => "asdf"
}
```

如果希望在输出显示完整的metadata数据（仅仅只有rubydebug才允许显示)
```
 stdout { codec => rubydebug { metadata => true } }
```
@metadata最常用应该是在date字段的过滤，在此之前，在处理nginx或者apache日志的时候，需要delete时间戳字段，然后使用日志的字段去覆盖，使用@metadata字段，就不需要这样了
```
 input { stdin { } }

filter {
  grok { match => [ "message", "%{HTTPDATE:[@metadata][timestamp]}" ] }
  date { match => [ "[@metadata][timestamp]", "dd/MMM/yyyy:HH:mm:ss Z" ] }
}
output {
  stdout { codec => rubydebug }
}
```
 #### 环境变量

```
 export TCP_PORT=12345
 input {
  tcp {
    port => "${TCP_PORT}"
  }
}
```
如果TCP_PORT没有设置，logstash返回配置错误
可以通过制定缺省的值解决这个问题
```
input {
  tcp {
    port => "${TCP_PORT:54321}"
  }
}
```
变量是不可变的，如果需要修改，需要重启logstash

### 配置多行事件

配置多行codec最重要的特征：
1 pattern 选项指定正则，匹配正则的行或者是前行的延续，或者开始一个新的多行
2 what选项 有2个值：previous /next ,previous 指定匹配正则的行是前行的一部分，next 指定匹配正则的行following line的一部分
3 negate选项 应用多行解码器到不匹配正则表达式的行

示例：
处理java stack traces
```
input {
  stdin {
    codec => multiline {
      pattern => "^\s"
      what => "previous"
    }
  }
}
```
不包含时间戳的行都属于前行
```
input {
  file {
    path => "/var/log/someapp.log"
    codec => multiline {
      pattern => "^%{TIMESTAMP_ISO8601} "
      negate => true
      what => previous
    }
  }
}
```

### glob pattern支持
\* 匹配任何文件 ，如果需要匹配隐藏文件，需要使用.*
\*\* 匹配递归目录
? 匹配任何单个字符
[set] 匹配任何在set里面的单个字符
{p,q} 匹配等价于(p|q）
\\
