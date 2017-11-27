title: Elastic Stack(ELK) 5.x版本部署概述
date: 2017-02-01 00:43:55
tags:
    - CS相关
    - 日志分析
    - ELK stack
categories:
    - 技术相关
    
---

## 前言
Elastic Stack是ELK日志系统的官方称呼，而ELK则是盛名在外的一款开源分布式日志系统，一般说来包括了Elasticsearch、Logstash和Kibana，涵盖了后端日志采集、日志搜索服务和前端数据展示等功能。
本文将会对Elastic Stack的安装部署流程进行一系列简单的介绍，并记录下了一些部署过程中遇到的坑及解决方法。<!--more-->
在本次实践中，我们所部署的ELK分布式日志系统，其架构大致如下：
![](http://7qnaj2.com1.z0.glb.clouddn.com/ELK%E6%9E%B6%E6%9E%84.png)
首先在各日志产生机上部署收集器Filebeat，然后Filebeat将监控到的log文件变化数据传至Kafka集群，Logstash负责将数据从kafka中拉取下来，并进行字段解析，向Elasticsearch输出结构化后的日志，Kibana负责将Elasticsearch中的数据进行可视化。

### 【重点参考】：

1. ELK中文书 http://kibana.logstash.es/content/

## 一、Elasticsearch的部署
首先在`https://www.elastic.co`中找到ES的安装包。下文中所用的安装包均为Linux 64的tar.gz压缩包，解压即可用。
其中，Elasticsearch需要依赖Java JDK1.8。JDK的安装方法在此不做赘述。
### 1.1 Elasticsearch的配置
ES的配置文件在解压根目录下的config文件夹中，其中elasticsearch.yml是主配置文件。
以基本可用作为部署目标，在该文件中仅需要设置几个重要参数：

1. `cluster.name`、`node.name`这两者顾名思义，作为集群和节点的标识符。
2. `Paths`部分下的`path.data`和`path.logs`，表示ES的数据存放位置，前者为数据存储位置，后者为ES的log存储位置。请尽量放到剩余空间足够的地方，此外在进行调优时有一种方法是将数据放置到SSD上。
3. `bootstrap.memory_lock: true`，设为true以确保ES拥有足够的JVM内存。
4. `network.host: localhost`和`http.port`，在此处设置ES对外服务的IP地址与端口

设置完以上几项参数后，即可在ES根目录下使用命令`./bin/elasticsearch`启动ES进程。也有相应的后台启动方式，具体不赘述。

### 1.2 Elasticsearch 5.x的Bootstrap Checks
Elasticsearch在升级到5.x版本后，启动时会强制执行[Bootstrap Checks(官方文档)](https://www.elastic.co/guide/en/elasticsearch/reference/current/bootstrap-checks.html)
其中经常性的问题是需要增大系统可使用的最大FileDescriptors数（参考[https://www.elastic.co/guide/en/elasticsearch/reference/current/file-descriptors.html](https://www.elastic.co/guide/en/elasticsearch/reference/current/file-descriptors.html)）
剩下的其他问题可以查询官方文档。

### 1.3 Elasticsearch的X-pack插件
X-pack是官方提供的一系列集成插件，包括了alert、monitor、secure等功能，十分强大（但是并不免费）。
在ELK 5.0中安装大部分插件仅需要输入命令：`./bin/elasticsearch-plugin install <plugin name>`即可
X-pack插件安装后会自动开启ELK的权限功能，需要注意的是如果启用了X-pack，则在向ES输入数据或发起API请求时，均需要附带相应的auth信息。
考虑到X-pack并非免费且价格昂贵，暂时不安装X-pack包。

### 1.4 Elasticsearch的Head插件
Head插件作为ELK 2.x版本中较为通用的前端管理插件，在ELK 5.x版本中无法直接使用`./bin/elasticsearch-plugin install head`的方式安装，但是可以采取standalone的方式进行运行。
参考官方文档：[https://github.com/mobz/elasticsearch-head#running-with-built-in-server](https://github.com/mobz/elasticsearch-head#running-with-built-in-server)
一篇较好的ES 5.x安装Head的博文：[http://www.cnblogs.com/xing901022/p/6030296.html](http://www.cnblogs.com/xing901022/p/6030296.html)
【特别注意】：暂时没有找到x-pack和head相互兼容的方法，目前由于认证的问题，如果启用了x-pack的secure功能，会导致head插件无法连接ES集群。


## 二、Logstash的部署
与Elasticsearch类似，在官网下载压缩包后，解压即可用。
在非高级场景下，Logstash本身不需要进行太多的配置（配置文件在logstash根目录下的`./config/logstash.yml`），高级场景请参考官方文档。
logstash的启动命令为:`./bin/logstash -f <pipeline_conf_file> --config.reload.automatic`，其中`-f`指定了pipeline配置文件的位置，`--config.reload.automatic`指定了pipeline配置文件可以进行热加载。
本次我们使用Logstash作为日志解析模块（Logstash其实也可以作为日志采集器），重点需要配置pipeline的三大部分：input、filter和output。pipeline文件需要自己创建。
### 2.1 input配置
我们选取Kafka作为数据输入。

```
input {
    kafka{
        bootstrap_servers => "xxx1:9092,xxx2:9092,xxx3:9092"
        topics => ["log_analysis","system_metric_monitor","sf_web_log","sf_engine_log","sf_platform_log"]
        codec => "json"
        group_id => "logstash"
        session_timeout_ms => "80000"
        request_timeout_ms => "810000"
        heartbeat_interval_ms => "1000"
        consumer_threads => 8
        #max_poll_records => "100"
        auto_commit_interval_ms => "1000"
        }
}
```
其中需要重点注意`session_timeout_ms`与`request_timeout_ms`两项，在logstash消费能力不足时，需要酌情修改上述两个timeout值，以防止kafka无法及时获取数据offset的问题。

### 2.2 filter配置
filter是重中之重，负责针对不同来源的日志进行解析。
本次使用数据中的`fields.msg_type`字段判断日志类型（属于文件日志还是系统metric），并使用`fields.log_tag`标记具体的来源模块。
当前使用的一个例子：

```
filter {
# file logs configuration
    if [fields][msg_type] == "sense-file-log" {
    # sf engine log
        if [fields][log_tag] == "sf_engine_log" {
            grok {
                match => {
                    "message" => "(?<log_level>.+)\s(?<log_time>.+)\s(?<tid>\d+)\s(?<filename>.+):(?<line>\d+)\]\s(?<msg>.*)"
                    }
            }
            grok {
                match => [
                    "msg","PushFrame: (?<PushFrame>\d+)",
                    "msg","encode time:(?<encode_time>\d+).*upload time:(?<upload_time>\d+)",
                    "msg","vnetwork: (?<vnetwork>\d+)ms",
                    "msg","network: (?<network>\d+)ms",
                    "msg","getH264Timeval: (?<getH264Timeval>\d+)"
                ]
            }
            date{
                match => [
                "log_time","HH:mm:ss"
                ]
               target => ["log_time"]
              }
        }
        # sf web service log
        else if [fields][log_tag] =~ /sf_web_log_service*/ {
            grok {
                match => {
                    "message" => "%{TIMESTAMP_ISO8601:log_time}\s\[%{DATA:thread_info}\]\s%{DATA:log_level}\s%{DATA:module_name}\s-\[.*\]\s>>>\s(?<msg>.*)"
                    }
            }
            date{
                match => [
                "log_time","yyyy-MM-dd HH:mm:ss"
                ]
               target => ["log_time"]
              }
        }
        # sf web tools log
        else if [fields][log_tag] =~ /sf_web_log_tools*/ {

        }
        # sf platform log
        else if [fields][log_tag] == "sf_platform_log"{
            grok {
                match => [
                    "message","%{TIMESTAMP_ISO8601:log_time} (?<thread_info>\d+\-\d+) (?<log_level>.+)\|(?<filename>.+)\|(?<line>\d+)\|(?<module_name>.+)\|(?<msg>.*)",
                    "message","%{TIMESTAMP_ISO8601:log_time} (?<log_level>.+)\|(?<filename>.+)\|(?<line>\d+)\|(?<module_name>.+)\|(?<msg>.*)",
                     "message","(?<msg>.*)"
                    ]
            }
            date{
                match => [
                "log_time","yyyy-MM-dd HH:mm:ss"
                ]
               target => ["log_time"]
              }
        }
    }
    mutate {
        convert => ["filename", "string"]
        convert => ["msg","string"]
        convert => ["pid","integer"]
        convert => ["tid","integer"]
        convert => ["line","integer"]
        convert => ["sleep_time","float"]
        convert => ["msg_type","string"]
        convert => ["via","string"]
        convert => [
                "PushFrame","integer",
                "encode_time","float",
                "upload_time","float",
                "vnetwork","float",
                "network","float",
                "getH264Timeval","float"
                 ]
        }
}

```

#### 2.2.1 配置文件中的变量引用
直接使用`[<field name>]`进行引用即可，类似json的层级，多级引用直接叠加，如`[fields][log_tag]`表示引用`fields.log_tag`

#### 2.2.2 grok表达式
重点使用grok进行日志的解析，其中有两种方式：

1. pattern模板，logstash自带了一系列匹配模板，也可以自己定义一些模板，在此不介绍模板定义的方法。使用上，例如：`%{DATA:log_level}`，表示使用`DATA`模板进行匹配，并将捕获到的数据放置到`log_level`字段下。
2. 直接使用类似正则表达式的方式进行捕获。例如：`(?<log_level>.+)`表示使用正则表达式`.+`进行匹配，并将匹配到的数据放置到`log_level`字段下。

#### 2.2.3 date处理
默认情况下，Logstash会使用数据输入的时间作为该条数据的时间戳`@timestamp`，但有些情况下我们需要使用日志内容中的时间作为数据时间戳，因此使用date插件。
用法如下：

```
date{
    match => [
    "log_time","yyyy-MM-dd HH:mm:ss"
            ]
    target => ["log_time"]
}
```
其中，match表示从哪个字段获取时间的原始字符串，target表示将时间解析后存入哪个字段（覆盖），默认情况下会覆盖掉`@timestamp`

#### 2.2.4 数据类型的转换
通常情况下使用正则表达式解析到的数据，字段类型都是string。在Kibana中（或者其他情况下），需要将某个字段转换为具体的数值类型，因此会用到mutate插件。
用法如下:

```
mutate {
        convert => ["filename", "string"]
        convert => ["msg","string"]
        convert => ["pid","integer"]
        convert => ["tid","integer"]
        convert => ["line","integer"]
        convert => ["sleep_time","float"]
        convert => ["msg_type","string"]
        convert => ["via","string"]
        convert => [
                "PushFrame","integer",
                "encode_time","float",
                "upload_time","float",
                "vnetwork","float",
                "network","float",
                "getH264Timeval","float"
                 ]
        }
```
仅支持string、integer、float类型。

### 2.3 output配置
一个简单的output例子：

```
output {
    if [fields][msg_type] == "sense-file-log" {
        if [fields][log_tag] == "sf_platform_log" {
            elasticsearch {
                hosts => ["localhost:9200"]
                index => "sf_platform_log"
            }
        }
        else if [fields][log_tag] == "sf_engine_log"{
            elasticsearch {
                hosts => ["localhost:9200"]
                index => "sf_engine_log"
            }
        }
        else if [fields][log_tag] =~/sf_web_log_service_*/{
            elasticsearch {
                hosts => ["localhost:9200"]
                index => "sf_web_log_service"
            }
        }
```
在配置文件中可以使用条件判断，其中`==`代表精确匹配，`=~`代表正则匹配
详情参考官方文档：[https://www.elastic.co/guide/en/logstash/current/event-dependent-configuration.html](https://www.elastic.co/guide/en/logstash/current/event-dependent-configuration.html)

```Plain
You can use the following comparison operators:
equality: ==, !=, <, >, <=, >=
regexp: =~, !~ (checks a pattern on the right against a string value on the left)
inclusion: in, not in

The supported boolean operators are:
and, or, nand, xor

The supported unary operators are:
!
```



## 三、Kibana的部署
Kibana是功能强大的前端可视化工具，同样通过解压部署。
解压后直接使用`./bin/kibana`即可启动默认配置下的Kibana，配置文件在`./config/kibana.yml`下。
### 3.1 配置内容
重点需要配置的地方为：

1. `server.host: "0.0.0.0"`，按照需求将Kibana暴露给指定IP
2. `elasticsearch.url: "http://localhost:9200"`，填入ES的地址与读取端口
3. 如果启用了x-pack，请不要忘记设置`elasticsearch.username: "elastic"`和`elasticsearch.password: "changeme"`，此处的用户名和密码设置为ES的指定用户名与密码。
4. `elasticsearch.requestTimeout: 60000`，设置Kibana读取ES数据的最大超时时间，该值如果设置过小，会导致在前端进行大范围搜索时因超时而失败。

### 3.2 Kibana的使用
操作界面
![](http://7qnaj2.com1.z0.glb.clouddn.com/Kibana%E7%95%8C%E9%9D%A2.jpeg)


#### 3.2.1 时间范围的选取：

Kibana界面的右上角有详细的Time range选取，并且可以在dashboard的时间曲线上进行框选。例如想观察某段峰值附近的数据，则直接框选峰值附近的时间段即可，此时整个Kibana的时间段都会设置为该值。

#### 3.2.2 日志的搜索分析：

1. 选取好时间段后（可以relative，如最近15分钟等；也可以指定时间段，如几点到几点），点击Kibana左边栏的Discover
2. 在弹出的搜索框中，首先与下方的深灰色下拉菜单中选取sense-face-log（这是本次日志数据存储的index）
3. 直接输入想要搜索的内容，需要注意的是，目前ES的倒排索引是以单词为单位的，故在搜索时如无法得到结果，可以用正则表达式星号"*"包围搜索词，做一个最大匹配。具体可自行体验。
4. 搜索的结果会高亮
5. 左边栏可以选取关注的字段名称，如添加PushFrame作为观察字段，则鼠标移至PushFrame一栏，并点击右边的add。其中，在左边栏中单击某个字段名，可以得到该字段的一些统计信息。
6. 更高级的搜索语法可以参考： https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl.html


#### 3.2.3 Kibana搜索示例
在进行相关搜索时，点击左边栏的Discover，并首先选取指定的index，然后在搜索框写入条件，需要注意右上角的时间范围，搜索例子：
选择`sf_web_log_service`作为目标index，并输入：

```
"fields.log_tag: sf_web_log_service_sfcl && cut"
```
即可搜索出包含“cut”字样的所有sfcl下（fields.log的值为`sf_web_log_service_sfcl`）的日志。


## 四、遇到的坑们
在参照网上他人的教程进行安装时，我遇到了很多问题，在此将各种坑及解决方法一一记录下来，方便后来者。

### 4.1 Elasticsearch

基础概念 http://www.qixing318.com/article/the-concept-of-elasticsearch.html

安装参考：http://www.cnblogs.com/hanyinglong/p/5409003.html

#### 4.1.1 bootstrap check
elastic search的bin启动时，如果设置的虚拟机内存不够大，就会启用bootstrap check，其中经常性地会卡在file descriptor check上。解决的方法即是在系统设置中增加相应的参数值。

**解决参考**：https://www.elastic.co/guide/en/elasticsearch/reference/current/file-descriptors.html
http://www.cnblogs.com/xing901022/p/6030296.html

#### 4.1.2 elasticsearch安装x-pack后需要输入密码
参考http://blog.csdn.net/gamer_gyt/article/details/53016426
默认的superuser为elastic:changeme
出于测试目的，仅适用默认的超级用户elastic。具体的更改位置应为相应的config/*.yml文件。

### 4.2 Elasticsearch Plugin——Head
head是一个可视化管理elasticsearch集群的插件，在此记录与ES 5.X相关的问题

参考安装教程：

1. http://www.cnblogs.com/xing901022/p/6030296.html

2. http://www.sojson.com/blog/85.html

#### 4.2.1 按照Head官方的文档，搭建standalone版本的elasticsearch-head，npm instal时出现无法找到node命令的问题
这个应该是部分ubuntu版本的系统环境问题，本人通过在运行/usr/bin中cp nodejs node解决，这是因为在某个版本中node.js的默认启动命令从node变成了nodejs

#### 4.2.2 成功grunt server后，在`http://xxxxx:9100/`中，可以看到页面，但是无法连接elasticsearch集群
目测是x-pack增加的加密认证所致。
神坑X-pack，因为5.0官方不推荐site_plugin了，所以在加密上会存在问题。
在kibana和elasticsearch中卸载（或disable）x-pack就可以用了。暂时这样吧。

### 4.3 Kibana
#### 4.3.1 初次启动kibana时，需要设置index pattern，但是无法点击确定，显示unable to fetch mapping, do you have indices matching the pattern?

这是因为输入的index pattern无法匹配在elasticsearch中现有的index，请确认修改的index pattern是否正确，或者检查指定的index是否有数据存入。

### 4.4 Logstash

#### 4.4.1 logstash的output设置为elasticsearch后，遇到error 401的问题
应该是没有正确设置ES密码
参考https://www.iamle.com/archives/2103.html
在logstash的pipeline conf中，于output项内添加elasticsearch的user和password

#### 4.4.2 设置密码后仍旧401或者出现configuration error
我遇到的问题是在pipeline中的output项填入了多个输出项（elasticsearch和stdout）

#### 4.4.3 在配合kafka进行使用时，由于消费者处理的时间过长，出现“WARN org.apache.kafka.clients.consumer.internals.ConsumerCoordinator - Auto offset commit failed for group kafka-es-sink”等问题。
按照提示，在Logstash的pipeline.conf中，input{kafka{}}项内增加session.timeout.ms => "60000"  （任意增加）
根据提示，有两种方法解决，一是增加timeout门限，二是减少max.poll.records
其中需要注意的是，session.timeout.ms需要小于request.timeout.ms的设定值

### 4.5 Filebeats
Filebeats高级配置： http://blog.csdn.net/a464057216/article/details/51233375

Beats系列作为ELK底层的数据源工具，具有轻量、功能强大的特点，并且可以不局限于ELK栈，实属数据采集之利器。
主要遇到的是多输入源的问题：在进行多输入源是，使用`-`符来分割input项内的各大数据源的配置内容。



### 4.6 Kafka
本次，我们选取kafka作为中间的数据缓冲层

#### 4.6.1 kafka的python client选取
基本的kafka操作可以在安装了Zookeeper的机器上使用kafka目录下的/bin/kafka-*.sh完成，但是我们现在需要写一个脚本对ELK进行监控，并将结果传入kafka。所以为了方便起见，使用kafka client作为工具是很必要的。
目前python下的kafka client主要有两个：[pykafka](https://github.com/Parsely/pykafka)和[kafka-python](https://github.com/dpkp/kafka-python)。
相关的对比讨论可见：https://github.com/Parsely/pykafka/issues/334

一个简明的对比讨论总结即是：

1. pykafka支持更多的python版本，而kafka-python取消了对python 3.4+的支持
2. kafka重启后，pykafka不需重启，而kafka-python需要。
3. 多consumer的负载管理上，pykafka效果更好
4. 单个producer/consumer的情况下，pykafka的producer速度更快，kafka-python的consumer速度更快





### 4.6.2 kafka使用过程中的坑
1. 一定要记得在使用的机器上修改hosts文件！否则会造成无法连接kafka的情况。因为在某些情况下，连接是直接使用hostname进行的。
2. 重点解决kafka消费者无法commit offset的问题，该问题是因为logstash无法在限定时间内消费完所有的数据，超出了kafka端设定的session timeout，导致session挂掉，且之前消费过的数据offset未能返回给kafka。在kafka端会认为该数据没有正确消费，并进行重新partition。logstash端超时后会重新建立consumer进行数据拉取，而kafka端会因为offset的问题重新发送之前“消费失败”的数据。
因此，这造成了ES一直有数据入库，在Kibana端却无法查询到最新日志。
解决方法：
    1. 在logstash的pipeline conf文件中增大session timeout值，需要注意`session_timeout_ms`值不能大于或等于`request_timeout_ms`值
    2. 尝试建立多topic多消费threads



## 总结
本文的部署目标仅仅是建立一个初级的分布式日志系统，还有更多高级的用法在此没有赘述，有兴趣者可以自行搜索得到更详细的介绍。
ELK作为一款被广泛使用的日志系统，具有部署简单、配置文件内容明了、集成功能强大等特点。在工业界，ELK 2.x版本已有非常成熟的应用，然而要升级至5.x版本会存在不小的问题。
本文尝试总结了ELK 5.x版本的各组件安装方法，并记录展示了在部署过程中遇到的坑。
凡事先查官方文档，查不到再进行相关网络搜索（如StackOverflow及Google）与请教他人。