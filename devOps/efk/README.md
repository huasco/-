# 第一部分 整体搭建架构说明    



# 第二部分 程序安装    




# 第三部分 EFK配置说明    

### 3.1 filebeat 配置

**filebeat.yml**

```Yaml

# 加载config目录的input配置
filebeat.config.prospectors:
  path: config/*.yml
  reload.enabled: true
  reload.period: 10s

filebeat.prospectors:

processors:
  - drop_fields:
      fields: ["beat", "input_type", "type", "offset"]

# 文件输出，用于测试
output.file:
  path: "./tmp"
  filename: output
  
# 输出到es，注意pipeline和indices配置，indices对应input的配置
output.elasticsearch:
  hosts: ["localhost:9200"]
  template.enabled: false
  pipeline: "testpipline"
  indices:
    - index: "etbc-callcenter"
      when.contains:
        fields.app_id: "etbc-callcenter"

# 日志级别和日志目录
logging.level: error
logging.files:
  path: ./log/filebeat
  
```

**config/input_config.yml**

```Yaml

# 有多个目录可以配置多个input段，注意multiline的配置
- input_type: log
  paths:
    - /mnt/logs/etbc-callcenter/biz/*.log
  fields:
    app_id: etbc-callcenter
  multiline.pattern: '^\[[FATAL|ERROR|WARN|INFO|DEBUG|TRACE]'
  multiline.negate: true
  multiline.match: after

```

**filebeat.sh (来自网上的一段启动脚本, 用法： ./filebeat.sh test|start|stop|restart|status)**

```Bash

#!/bin/bash
PATH=/usr/bin:/sbin:/bin:/usr/sbin
export PATH

# 注意修改这里的路径
agent="/usr/filebeat/filebeat"
args="-c /usr/filebeat/filebeat.yml -path.home /usr/filebeat -path.config /usr/filebeat -path.data /usr/filebeat/data -path.logs /var/log/filebeat"
test_args="-e -configtest"

test() {
$agent $args $test_args
}
start() {
    pid=`ps -ef |grep /usr/filebeat/data |grep -v grep |awk '{print $2}'`
    if [ ! "$pid" ];then
        echo "Starting filebeat: "
        test
        if [ $? -ne 0 ]; then
            echo
            exit 1
        fi
        $agent $args &
        if [ $? == '0' ];then
            echo "start filebeat ok"
        else
            echo "start filebeat failed"
        fi
    else
        echo "filebeat is still running!"
        exit
    fi
}
stop() {
    echo -n $"Stopping filebeat: "
    pid=`ps -ef |grep /usr/filebeat/data |grep -v grep |awk '{print $2}'`
    if [ ! "$pid" ];then
echo "filebeat is not running"
    else
        kill $pid
echo "stop filebeat ok"
    fi
}
restart() {
    stop
    start
}
status(){
    pid=`ps -ef |grep /usr/filebeat/data |grep -v grep |awk '{print $2}'`
    if [ ! "$pid" ];then
        echo "filebeat is not running"
    else
        echo "filebeat is running"
    fi
}
case "$1" in
    start)
        start
    ;;
    stop)
        stop
    ;;
    restart)
        restart
    ;;
    status)
        status
    ;;
    *)
        echo $"Usage: $0 {start|stop|restart|status}"
        exit 1
esac

```

### 3.1 elasticsearch 和 kibana 配置

es没有配置文件，所有的配置都是通过 http rest 接口去实现的

**修改es默认密码**

```
PUT /_xpack/security/user/elastic/_password

{
  "password" : "huasco@51888"
}

```

**ingest pipeline配置（注意 date 里面要配置一个时区，否则 kibana 解析日期会差8小时）**

```
PUT /_ingest/pipeline/huasco_pipline

{
    "description": "huasco_pipline",
    "processors": [
        {
            "grok": {
                "field": "message",
                "patterns": [
                    "\\[%{WORD:level}\\]\\[%{TIMESTAMP_ISO8601:timestamp}\\]\\[%{PROG:logger}\\]\\[%{PROG:pid_pname}\\]\\[%{JAVACLASS:class}\\]\\[%{WORD:method}\\]\\[%{INT:line}\\]%{NEWLINE:ignoreme}%{GREEDYDATA:msg}"
                ]
            }
        },
        {
            "date": {
                "field": "timestamp",
                "formats": [
                    "YYYY-MM-dd HH:mm:ss,SSS"
                ],
                "timezone": "Asia/Shanghai"
            }
        }
    ]
}

```

**修改默认的 ingest pattern**

打开\modules\ingest-common\ingest-common-5.4.0.jar，修改里面patterns目录下的grok-patterns，有两处修改：
第一个，修改 GREEDYDATA（因为默认的不能读取多行，这个和logstash不一致），第二在它上面加一行配置 NEWLINE（表示任意多个换行），具体如下：

```
NEWLINE \n+
GREEDYDATA ([\s\S]*)
```

**kibana 配置**

kibana.yml

```Yaml
---
## Default Kibana configuration from kibana-docker.
## from https://github.com/elastic/kibana-docker/blob/master/build/kibana/config/kibana.yml
#
server.name: kibana
server.host: "0"
elasticsearch.url: http://elasticsearch:9200
elasticsearch.username: elastic
elasticsearch.password: huasco@51888
xpack.monitoring.ui.container.elasticsearch.enabled: true
```


# 第四部分 使用注意事项    





