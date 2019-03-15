## 小张的基地
![banner](https://github.com/zhangyage/dockerfile/blob/master/dockerfile/lab-load-balance/docs/images/banner.png)
## elasticsearch+logstash+kibana
链接：https://pan.baidu.com/s/1E3xgY-kxvvH0ADfaGo1kew 
提取码：hkov  
### 概念
Elasticsearch是个开源分布式搜索引擎，提供搜集、分析、存储数据三大功能。它的特点有：分布式，零配置，自动发现，索引自动分片，索引副本机制，restful风格接口，多数据源，自动搜索负载等。

Logstash 主要是用来日志的搜集、分析、过滤日志的工具，支持大量的数据获取方式。一般工作方式为c/s架构，client端安装在需要收集日志的主机上，server端负责将收到的各节点日志进行过滤、修改等操作在一并发往elasticsearch上去。

Kibana 也是一个开源和免费的工具，Kibana可以为 Logstash 和 ElasticSearch 提供的日志分析友好的 Web 界面，可以帮助汇总、分析和搜索重要数据日志。

官方文档：
Filebeat：

https://www.elastic.co/cn/products/beats/filebeat
https://www.elastic.co/guide/en/beats/filebeat/5.6/index.html

Logstash：
https://www.elastic.co/cn/products/logstash
https://www.elastic.co/guide/en/logstash/5.6/index.html

Kibana:

https://www.elastic.co/cn/products/kibana

https://www.elastic.co/guide/en/kibana/5.5/index.html

Elasticsearch：
https://www.elastic.co/cn/products/elasticsearch
https://www.elastic.co/guide/en/elasticsearch/reference/5.6/index.html

安装一个切屏软件
```
yum -y install screen
```
es和Logstash 依赖java安装一下java依赖
```
 mkdir /usr/local/java -p
 tar -zxvf jdk1.8.0_111.tar.gz -C /usr/local/java/

#java env
JAVA_HOME=/usr/local/java/jdk1.8.0_111
PATH=$JAVA_HOME/bin:$PATH
CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export JAVA_HOME             
export PATH
```

### 安装es:
```
tar -zxvf elasticsearch-6.6.1.tar.gz -C /opt/
修改配置文件监听端口
[root@localhost config]# pwd
/opt/elasticsearch-6.6.1/config
[root@localhost config]# vim elasticsearch.yml
network.host: 0.0.0.0
启动服务：
./bin/elasticsearch
```
这里需要注意的是es使用上面的方式启动的时候是启动在前台，如果需要后台启动，需要使用-d命令

#### 启动es常见的报错和处理方法
###### 报错1  
[2019-03-08T12:30:53,806][WARN ][o.e.b.ElasticsearchUncaughtExceptionHandler] [unknown] uncaught exception in thread [main]
org.elasticsearch.bootstrap.StartupException: java.lang.RuntimeException: can not run elasticsearch as root
解决办法：
创建一个普通用户启动
useradd elk

###### 报错2
Exception in thread "main" java.nio.file.AccessDeniedException: /opt/elasticsearch-6.6.1/config/jvm.options
解决办法：
[root@localhost ~]# chown -R elk:elk /opt/elasticsearch-6.6.1/

报错3
[1]: max file descriptors [4096] for elasticsearch process is too low, increase to at least [65536]  
每个进程最大同时打开文件数太小，可通过下面2个命令查看当前数量  
ulimit -Hn  
修改/etc/security/limits.conf文件，增加配置，用户退出后重新登录生效  
```
*               soft    nofile          65536  
*               hard    nofile          65536  
*               soft    nproc           4096  
*               hard    nproc           4096 
```
[2]: max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]  
修改/etc/sysctl.conf文件，增加配置vm.max_map_count=262144  

vi /etc/sysctl.conf  
sysctl -p  

##### 验证：  
```
[elk@localhost logstash-6.6.1]$ curl localhost:9200
{
  "name" : "R6_CNXv",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "xCe1UxHPR_GTZwOaevHyPw",
  "version" : {
    "number" : "6.6.1",
    "build_flavor" : "default",
    "build_type" : "tar",
    "build_hash" : "1fd8f69",
    "build_date" : "2019-02-13T17:10:04.160291Z",
    "build_snapshot" : false,
    "lucene_version" : "7.6.0",
    "minimum_wire_compatibility_version" : "5.6.0",
    "minimum_index_compatibility_version" : "5.0.0"
  },
  "tagline" : "You Know, for Search"
}
```
到这里es的配置就结束了，我们使用的是单机版测试的。  
#### es的常用命令：
``` 
curl 'localhost:9200/_cat/nodes?v' #查询节点  
curl 'localhost:9200/_cat/indices?v'  #查看所有的索引  
curl -XPUT 'localhost:9200/customer?pretty'  #创建索引
```

### 配置logstash
#修改tomcat日志格式
```
        <!--  
        <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"  
               prefix="localhost_access_log" suffix=".txt"  
               pattern="%h %l %u %t &quot;%r&quot; %s %b" />  
        -->  
        <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"  
               prefix="tomcat_access_log" suffix=".log"  
               pattern="{&quot;clientip&quot;:&quot;%h&quot;,&quot;ClientUser&quot;:&quot;%l&quot;,&quot;authenticated&quot;:&quot;%u&quot;,&quot;AccessTime&quot;:&quot;%t&quot;,&quot;method&quot;:&quot;%r&quot;,&quot;status&quot;:&quot;%s&quot;,&quot;SendBytes&quot;:&quot;%b&quot;,&quot;Query?string&quot;:&quot;%q&quot;,&quot;partner&quot;:&quot;%{Referer}i&quot;,&quot;AgentVersion&quot;:&quot;%{User-Agent}i&quot;}i" />
```
将上面的内容使用下面的内容代替

#### 书写配置文件：
定义存储的日志
```
input {
    file {
      path => "/opt/apache-tomcat-8.5.38/logs/tomcat_access_log*.log"
      type => "tomcat-access-log-ceshi"
      start_position => "beginning"
      stat_interval => "2"
    }
}
output {
    elasticsearch {
      hosts => ["192.168.32.149:9200"]
      index => "logstash-tomcat-access-log-ceshi-%{+YYYY.MM.dd}"
    }
}
```
catalina.out
```
###catalina.out###
input {
  file {
        path => "/opt/apache-tomcat-8.5.38/logs/catalina.out"
        sincedb_path => "/opt/elasticsearch-6.6.1/logs/sincedb_apache_access.log.txt"
        start_position => "beginning"
        stat_interval => "2"
    }
}
filter {
    if [path] =~ "access" {
        mutate { replace => { "type" => "tomcat catalina.out" } }
        grok {
            match => { "message" => "%{COMBINEDAPACHELOG}" }
        }
   }
  date {
    match => [ "timestamp" , "dd/MMM/yyyy:HH:mm:ss Z" ]
  }
}
output {
        elasticsearch {
        hosts => ["192.168.32.149:9200"]
        index => "tomcat_out"
        }
        stdout {
            codec => rubydebug
        }
}
```
写好配置文件后可以执行命令验证文件语法的正确性
```
[elk@localhost logstash-6.6.1]$ ./bin/logstash -f root-tomcat.conf -t 
```
如果没有语法问题直接执行  
```
[elk@localhost logstash-6.6.1]$ ./bin/logstash -f root-tomcat.conf
```
执行完成后由于我们收集的是tomcat的日志，这个时候我们访问一下tomcat的web见面刷新一下，更新一下日志：
查看es中是否有新的key生成
```
curl 'localhost:9200/_cat/indices?v'
```
查看有新的key生成
![banner](https://github.com/zhangyage/elk/blob/master/elk/images/es-key-check.png)

#### nginx的日志收集
先定义一下日志格式为json:
```
    log_format log_json '{ "@timestamp": "$time_local", '
    '"remote_addr": "$remote_addr", '
    '"referer": "$http_referer", '
    '"request": "$request", '
    '"status": $status, '
    '"bytes": $body_bytes_sent, '
    '"agent": "$http_user_agent", '
    '"x_forwarded": "$http_x_forwarded_for", '
    '"up_addr": "$upstream_addr",'
    '"up_host": "$upstream_http_host",'
    '"up_resp_time": "$upstream_response_time",'
    '"request_time": "$request_time"'
    ' }';

    access_log  logs/access.log log_json;
```
定义一下nginx收集的配置文件：
```
[root@localhost logstash-6.6.1]# cat nginx_log.conf 
input {
    file {
      path => "/usr/local/nginx/logs/access.log"
      type => "nginx"
      start_position => "beginning"
      stat_interval => "2"
    }
}
output {
    elasticsearch {
      hosts => ["192.168.32.149:9200"]
      index => "logstash-nginx-access-log-%{+YYYY.MM.dd}"
    }
}
```

### kibana
```
[root@localhost config]# cat kibana.yml | grep -v '^#' |grep -v '^$'  
server.host: "0.0.0.0"  
elasticsearch.hosts: ["http://192.168.32.149:9200"]  
```
启动kibana
```
[elk@localhost kibana-6.6.1-linux-x86_64]$ ./bin/kibana
```
监听端口5601  浏览器访问测试
![k1](https://github.com/zhangyage/elk/blob/master/elk/images/k1.png)
![k2](https://github.com/zhangyage/elk/blob/master/elk/images/k2.png)
![k3](https://github.com/zhangyage/elk/blob/master/elk/images/k3.png)

