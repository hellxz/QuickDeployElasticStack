# Docker极简部署Kafka+Zookeeper+ElasticStack
之前写ELK部分时有朋友问有没有能一键部署的Kafka+ELK，写本文主要是填这个坑，基本上配置已经集中在一两个文件中了，理论上此配置支持ElasticStack 7.x所有版本



本文所有配置与代码均在本人Github中可以找到：<https://github.com/hellxz/QuickDeployElasticStack>

## 测试环境

- Ubuntu 18.04 LTS
- Docker 18.09.7
- docker-compose 1.24.0
- 主机IP：192.168.87.139
- ElasticStack 7.5.2

## 端口号占用表

| 服务名称      | 默认端口号                |
| ------------- | ------------------------- |
| elasticsearch | 9200                      |
| logstash      | 9600                      |
| kibana        | 5601                      |
| zookeeper     | 2181                      |
| kafka         | 9090                      |
| kafka-manager | 9001（防止与cerebro冲突） |

> 以上端口号本来想再提供自定义配置到 `.env` 文件中的，有过度设计的嫌疑，所以就不放上去了

##  部署ElasticStack的主机的配置

**为ES修改内存限制**

```bash
$ sudo vim /etc/security/limits.conf
#添加如下内容并保存退出
* soft memlock unlimited
* hard memlock unlimited
```

**修改系统限制文件打开数、线程数等**

```bash
$ sudo vim /etc/systemd/system.conf
#最下方添加，参数值可以更大些
DefaultLimitNOFILE=65536
DefaultLimitNPROC=32000
DefaultLimitMEMLOCK=infinity
```

**重启主机** 或 **执行命令**：`systemctl daemon-reexec`

**修改mmap计数的操作系统限制 并 禁用swap**

```bash
$ sudo vim /etc/sysctl.conf
#添加如下内容，如有配置，请修改
vm.max_map_count=262144
vm.swappiness=0
#保存退出
$ sudo sysctl -p
#立即永久生效
```

> 有兴趣的可以写Shell脚本自动配置，本人觉得这些配置最好还是对系统管理员或部署人员可见为好



## Clone 仓库到本地

```shell
git clone https://github.com/hellxz/QuickDeployElasticStack.git
cd QuickDeployElasticStack
```

文件结构如下

```
.
├── docker-compose.yml
├── logstash-pipeline
│   └── logstash.conf
└── README.md
```



## 修改配置文件.env

按需修改.env文件

```shell
#docker-compose.yml引用变量，便于单机部署ElasticStack
#Author: Hellxz

#=========================== 宿主机 配置项 ===================================
#宿主机ip
LOCALHOST_IP=192.168.87.139

#=========================== ElasticStack 共用配置 ===========================
#ELK Docker镜像版本号
ELASTIC_STACK_VERSION=7.5.2

#=========================== Elasticsearch 配置项 ============================
#Elastsearch JVM设置, Xms与Xmx保持相同，最大不要超过32G
ES_JVM_OPTS=-Xms8g -Xmx8g

#Elastsearch数据持载目录与日志目录，需要映射到主机上
ES_DATA_DIR=/data/elk/es-data
ES_LOGS_DIR=/data/elk/es-logs

#=========================== Logstash 配置项 =================================
#Logstash 流水线工作线程数
LOGSTASH_PIPELINE_WORKERS=5

#Logstash JVM设置
LS_JAVA_OPTS=-Xms4g -Xmx4g

#=========================== Kafka 配置项 ====================================
#Kafka主机名
#外部访问kafka时，只需将客户端主机hosts添加Kafka宿主机ip与此主机名的映射
#例如，"10.2.6.63 kafka1"
KAFKA_HOSTNAME=kafka1

#Kafka数据目录
KAFKA_DATA_DIR=/data/elk/kafka-data

#Kafka JVM设置
KAFKA_JVM_OPTS=-Xms4g -Xmx4g

#Kafka启动时创建的Topics
#格式为"topic名称:分区数:副本数[:清理策略]", 多个topic以','分开
KAFKA_BOOTSTRAP_CREATE_TOPICS=logsTopic:5:1:compact

#=========================== KafkaManager 配置项 =============================
#自定义KafkaManager端口号
KAFKA_MANAGER_PORT=9001

#============================ 自定义参数 =====================================
```



## 修改logstash-pipeline/logstash.conf

```bash
input {
  kafka {
    bootstrap_servers => "kafka1:9090" #替换为kafka映射hosts名称
    topics => ["logsTopic"]
    consumer_threads => 3
    group_id => "logstash"
    decorate_events => true
    codec => json
  }
}

filter {
	#这里留给大家自由发挥
}

output {
  elasticsearch {
    hosts => ["192.168.87.139:9200"] #es地址
    index => "logs-%{+YYYY.MM.dd}"
    #user => "elastic"
    #password => "changeme"
  }
  stdout {
    codec => rubydebug
  }
}

```

## 快速部署

配置完前边的部分后，只需要在docker-compose.yml的文件夹下执行一行命令即可启动

```shell
docker-compose up -d
```



## 部署后修改配置

部署后修改配置的问题再所难免，处理也较简单

修改 `docker-compose.yml` 的话，执行部署命令会检测变更

```shell
docker-compose up -d
```

非上述文件的话，需要判断变动配置会影响哪些容器，如：

- Kafka依赖Zookeeper

- Logstash依赖Es与Kafka

- Kibana依赖Es

  分别按依赖关系重启容器即可（被依赖的如果变动需要先重启）不被依赖的直接重启

```shell
docker restart 容器ID或Name
```







**后续**

本文源码直接去Github上看吧，技术含量不高，主要是为了方便大家部署ELK  

最后，如果本文对你有帮助，欢迎推荐、评论，转载请注明出处。

源码地址：<https://github.com/hellxz/QuickDeployElasticStack>