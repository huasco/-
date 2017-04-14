# ELK

Elasticsearch+Logstash+Kibana搭建实时日志分析平台

# 搭建环境

MacOS Sierra+Java8

# 搭建过程

### logstash

1. 使用以下的命令：

`brew install logstash`

1. 安装好后使用以下命令检查是否安装成功

```
$ logstash --version  
#(output)logstash 5.2.2

```

### Elasticsearch

1. 安装命令（ps：最好使用的Java环境是1.8，不然会有奇奇怪怪的错误）

`brew install elasticsearch`

1. 安装好之后使用以下指令查看是否安装成功

```
$ elasticsearch --version
(output)Version: 5.2.2, Build: f9d9b74/2017-02-24T17:26:45.835Z, JVM: 1.8.0_121

```

1. 启动停止Elasticsearch

```
brew services start elasticsearch
brew services stop elasticsearch

```

或者在Elasticsearch的安装目录下使用(brew安装的路径一般都是/usr/local/Cellar中的)

`bin/elasticsearch`

当然，还可以将elasticsearch添加至环境变量中

1. 浏览器访问`http://localhost:9200` 可以看到如下信息

```
{
  "name" : "8dfa-dd",
  "cluster_name" : "elasticsearch_limbo",
  "cluster_uuid" : "Np6rkkJWTHSAfOOql902Pw",
  "version" : {
    "number" : "5.2.2",
    "build_hash" : "f9d9b74",
    "build_date" : "2017-02-24T17:26:45.835Z",
    "build_snapshot" : false,
    "lucene_version" : "6.4.1"
  },
  "tagline" : "You Know, for Search"
}

```

### Kibana

1. Kibana不需要通过brew安装，直接下载压缩包后，解压
2. 在Kibana目录下使用bin/kibana,然后访问`http://localhost:5601` 什么都不用管。

### X-Pack

1.在Elasticsearch 5.x的时代，监控和管理由X-Pack统一完成，包含：

- 安全：用户权限管理
- 告警：自动告警
- 监控：监控Elasticsearch集群的状态
- 报告：发送报告、导出数据
- 图表：可视化数据

1. 在安装X-Pack之前，需要停止Kibana和Elasticsearch

2. 然后分别在Kibana和Elasticsearch目录下输入

   ```
   bin/elasticsearch-plugin install x-pack
   bin/kibana-plugin install x-pack

   ```

3. 安装完成后，启动elasticsearch和kibana，访问kibana时发现需要登录了， 默认用户名和密码是elastic/changeme。

4. 后续可以在Management面板中进行用户和角色的配置，也可以看到新增了Reporting。

5. 在Monitoring页面中可以看到Elasticsearch和Kibana的状态，点击Indices还可以看到具体索引的状态。

### 例子

**官网上关于shakespeare(莎士比亚全集的例子)**

1. 下载官网上的 shakespeare.json 链接在[这里](https://www.elastic.co/guide/en/kibana/current/tutorial-load-dataset.html)
2. shakespeare.json中的信息格式大概是这样的：

```
{
    "line_id": INT,
    "play_name": "String",
    "speech_number": INT,
    "line_number": "String",
    "speaker": "String",
    "text_entry": "String",
}
```

1. 然后如果安装了X-Pack插件支持，则需要输入用户名密码如下：

```
curl -XPUT -u elastic http://localhost:9200/shakespeare -d '
{
 "mappings" : {
  "_default_" : {
   "properties" : {
    "speaker" : {"type": "string", "index" : "not_analyzed" },
    "play_name" : {"type": "string", "index" : "not_analyzed" },
    "line_id" : { "type" : "integer" },
    "speech_number" : { "type" : "integer" }
   }
  }
 }
}
';

```

如果没有安装X-Pack的支持，则吧其中的-u elastic删了就好了

1. 上述就是在elasticsearch中建立了一个索引，之后的kibana就是用这个索引来读取展示elasticsearch中的信息。
2. 接下来就是开启kibana，kibana目录下命令行输入`bin/kibana` ,然后在management中（x-pack安装之后setting就变成了这个了），配置index，具体可以看官网上的[说明](https://www.elastic.co/guide/en/kibana/current/tutorial-define-index.html)，或者是这篇[博客](http://blog.csdn.net/ywheel1989/article/details/60519151)
3. 至此，都已经配置结束了，只要去Discover中去检索就好啦！

**使用Logstash实时写入数据**

1. 编写logstash的.conf文件，文件内容如下：

```
input{
    file{
        path => ["/Users/limbo/Desktop/测试数据/test/test.md"]
	    start_position => beginning
        ignore_older => 0
        sincedb_path => "/dev/null"
    }
}   
filter {
    grok {
        match => { "message" => "%{IPORHOST:clientip} - %{USER:auth} \[%{HTTPDATE:timestamp}\] \"(?:%{WORD:verb} %{NOTSPACE:request}(?: HTTP/%{NUMBER:httpversion})?|%{DATA:rawrequest})\" %{NUMBER:response} (?:%{NUMBER:bytes}|-)"}
    }
    date {
        match => [ "timestamp" , "dd/MMM/YYYY:HH:mm:ss +0800" ]
    }
}
output{
    elasticsearch{
        user => "elastic"     #使用x-pack需要使用账户信息
        password => "elastic"
   }
    stdout{}
}
```

1. 文件内容解析：

- input：表示logstash读取文件的一下信息和相关读取配置

  1. 其中我只是用了file这个类型，还有其他四个类型，下面是摘自官网的一段简介：

  > Inputs
  >
  > You use inputs to get data into Logstash. Some of the more commonly-used inputs are:
  >
  > - file: reads from a file on the filesystem, much like the UNIX command tail -0F
  > - syslog: listens on the well-known port 514 for syslog messages and parses according to the RFC3164 format
  > - redis: reads from a redis server, using both redis channels and redis lists. Redis is often used as a "broker" in a centralized Logstash installation, which queues Logstash events from remote Logstash "shippers".
  > - beats: processes events sent by Filebeat.

  `start_position => beginning`告诉logstash从我的log文件的头部开始往下找，不要从半中间开始。 `ignore_older => 0`告诉logstash不要管我的log有多古老，一律处理，否则logstash缺省会从今天开始，就不管老日志了。 `sincedb_path => "/dev/null"`这句话也很关键，特别是当你需要反复调试的时候，因为logstash会记住它上次处理到哪儿了，如果没有这句话的话，你再想处理同一个log文件就麻烦了，logstash会拒绝处理。现在有了这句话，就是强迫logstash忘记它上次处理的结果，从头再开始处理一遍。

  - fliter:表示配置中需要用到的过滤器，其中的grok是logstash的一个重要的插件，相当于一个正则表达式，date就是根据你logstash读入的日志的日期进行过滤
  - output：顾名思义，这就是输出，其中这里的elasticsearch表示输出到elasticsearch。

  实际上Logstash 不只是一个`input | filter | output` 的数据流，而是一个 `input | decode | filter | encode | output` 的数据流，具体的配置可以参照官方的[文档](https://kibana.logstash.es/content/logstash/)

  至此，单节点的搭建就已经完成了

  ## 坑点

  1. kibana右上角的时间选择器一定要选好，否则什么数据都有可能看不到