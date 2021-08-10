# 数据总线（使用参考手册）

## 一、kafka单机版集群搭建

192.126.124.243服务器上`已有单机版kafka集群，监控工具，和服务（从gitlab.kdsec.org的yizhan也可以获得到）`

### 1、环境要求:

需要有docker和docker-compose环境

### 2、部署

docker-compose -f docker-compose.yml up -d

### 3、查看是否部署成功

docker ps

### 4、关闭单机版服务

docker-compose -f docker-compose.yml down

## 二、监控工具Kafka-Eagle的介绍

### 1、访问地址

`yourip:8048/ke`

### 2、用户名和密码



用户名:`admin`

密 码：`123456` （<font color="red">密码最好换成复杂的</font>）

### 3、官方地址

[kafka-eagle的github地址](https://github.com/smartloli/kafka-eagle)

[kafka-eagel的官方文档](https://www.kafka-eagle.org/articles/docs/documentation.html)

### 4、界面图例

![image-20201228194312444](C:\Users\Dacyuan\AppData\Roaming\Typora\typora-user-images\image-20201228194312444.png)



![image-20201228194827172](C:\Users\Dacyuan\AppData\Roaming\Typora\typora-user-images\image-20201228194827172.png)

## 三、服务接口介绍

**返回格式**：

```json
{
    "code": int,
    "data": list,
    "msg": str
}
```



### 1、生产消息

流程：创建topic -> 向topic发送消息  `or`  创建topic -> 向指定topic的指定分区发送消息

#### (1)、生产消息（`不并行，不需要知道topic分区的情况`）

需要用到的接口：**POST /api/topics/<topic_name>**

```python
"""
Demo:向指定topic发送数据，test已创建
"""
import requests

url = "http://192.168.124.243:8006/api/topics/test"
body = {
    "records": [
    {
        "key": "a2V5",
        "value": "I am a test"
    },
    {
        "value": "My name is test"
    }
    ]
}
requests.post(url,json=body)
```

#### (2)、生产消息（`并行即创建多个生产者，需要知道topic分区的情况`）

需要用到的接口： **POST /api/topics/<topic_name>/partitions/<partition_id>**

```python
'''
Demo:向指定topic的指定分区发送数据，test已创建
test有两个分区，故可以最大的并行生产消息的生产者为2个
一个生产者向test的分区0发送数据
另一个生产者同时向test的分区1发送数据
'''
import requests

url = "http://192.168.124.243:8006/api/topics/test/partitions/0"
body = {
    "records": [
    {
        "key": "a2V5",
        "value": "I am a test"
    },
    {
        "value": "My name is test"
    }
    ]
}
resp = requests.post(url,json=body)
print(resp.text)

```

### 2、消费消息

流程：创建消费者实例 -> 订阅主题（可以订阅多个topic主题） -> 消费数据    `or`   创建消费者实例 -> 手动分配topic的分区消费者实例 -> 设置分区的offset -> 消费数据                                                                                                                                                                                                                                  

#### (1)、消费消息（`不并行，不需要知道topic分区的情况`）

需要用到的接口：

**POST /api/consumers/<group_name>** (创建消费者实例)

**POST /api/consumers/<group_name>/instances/<instance_name>/subscription**（订阅主题)

**GET /api/consumers/<group_name>/instances/<instance_name>/records?max_bytes=**（消费数据）

```python
"""
Demo:消费数据
"""

import requests

#创建消费者例 group6组的my_consumer实例
instance_url = "http://192.168.124.243:8006/api/consumers/group6"

instance_body = {
  "name": "my_consumer",
  "format": "json",
  "auto.offset.reset": "earliest",
  "auto.commit.enable": "false"
}
resp = requests.post(instance_url,json=instance_body)

# 订阅topic：test
subscribe_url = "http://192.168.124.243:8006/api/consumers/group6/instances/my_consumer/subscription" 
subscribe_body = {
  "topics": [
    "test"
  ]
}
resp = requests.post(subscribe_url,json=subscribe_body)

# #消费数据(后面可跟参数max_bytes=******)
consumer_url = "http://192.168.124.243:8006/api/consumers/group6/instances/my_consumer/records"
resp = requests.get(consumer_url)
print(resp.text)

```

#### (2）、消费消息（`并行即创建多个消费者,需要知道topic分区情况且需要手动分配消费分区，可以从自定义的offset处消费，较上面复杂些`）

`当然你也可以使用这些接口不并行的消费数据（只创建一个消费者实例就行了，但为了保证topic的数据都能全部消费到，你得清楚地知道该topic有几个分区，并手动分配给消费者实例该topic的全部分区）`



需要用到的接口:

**POST /api/consumers/<group_name>**(创建消费者实例)

**POST /api/consumers/<group_name>/instances/<instance_name>/assignments**(手动分配消费分区)

**POST /api/consumers/<group_name>/instances/<instance_name>/positions/beginning**(从offset为beginning处消费) or 

**POST /api/consumers/<group_name>/instances/<instance_name>/positions/end**(从offset为end处消费) or

**POST /api/consumers/<group_name>/instances/<instance_name>/positions**(从自定义的offset处消费)

**GET /api/consumers/<group_name>/instances/<instance_name>/records?max_bytes=**(消费数据)

```python
"""
Demo: 最简单的设置：最早的offset处开始消费数据(创建了一个消费者，消费一个分区)
"""
import requests
#创建消费者例
instance_url = "http://192.168.124.243:8006/api/consumers/group6"
instance_body = {
  "name": "my_consumer",
  "format": "json",
  "auto.offset.reset": "earliest",
  "auto.commit.enable": "false"
}
requests.post(instance_url,json=instance_body)


# 手动分配分区
manual_url = "http://192.168.124.243:8006/api/consumers/group6/instances/my_consumer/assignments"
manual_body = {
  "partitions": [
    {
      "topic": "topic-3",
      "partition": 0
    }
  ]
}
requests.post(manual_url,json=manual_body)

#设置最早的offset
offset_url = "http://192.168.124.243:8006/api/consumers/group6/instances/my_consumer/positions/beginning"
offset_body = {
  "partitions": [
    {
      "topic": "topic-3",
      "partition": 0
    }
  ]
}
requests.post(offset_url,json=offset_body)

#消费数据
consumer_url = "http://192.168.124.243:8006/api/consumers/group6/instances/my_consumer/records"
resp = requests.get(consumer_url)
print(resp.text)
```

```python
"""
 Demo并行消费(创建10个消费者实例，并行消费数据 默认从offset为0处开始消费)
"""
import requests

# #创建10个消费者例
instance_url = "http://192.168.124.243:8006/api/consumers/group"
instance_body = {}
for i in range(0, 10):
    instance_body["name"] = "consumer" + str(i);
    instance_body["format"] = "json";
    instance_body["auto.offset.reset"] = "earliest";
    instance_body["auto.commit.enable"] = "false";
    resp = requests.post(instance_url,json=instance_body)

# 手动分配消费者消费哪个topic分区 ，指定consumer0消费test10的0分区，consumer1消费test10的1分区....
arr = []
value = {}
body = {}
for i in range(0, 10):
    value["topic"] = "test10"
    value["partition"] = i
    arr.append(value)
    body = {"partitions":arr}
    assign_url = "http://192.168.124.243:8006/api/consumers/group/instances/" +"consumer" + str(i) +  "/assignments"
    resp = requests.post(assign_url,json=body)


# 每个消费者开始读取对应的topic的分区，即consumer0从test10的分区0里拿数据
for i in range (0, 10):
    consume_url = "http://192.168.124.243:8006/api/consumers/group/instances/consumer"+str(i)+"/records"
    resp = requests.get(consume_url)
    print(resp.text)

```



![](.\1.png)                                                                                                         

### 3、所有接口详细介绍

#### Topics



> GET /api/topics   获取所有的topic	

Example request:

```tex
GET /api/topics
```

Example response:

```json
{
    "code": 200,
    "data": [
        "test",
        "topic-3"
    ],
    "msg": "success"
}
```



> GET /api/topics/<topic_name> 获取特定topic的详细信息

Example request:

```tex
GET /api/topics/topic-3
```

Example response:

```json
{
    "code": 200,
    "data": {
        "configs": {
            "cleanup.policy": "delete",
            "compression.type": "producer",
            "delete.retention.ms": "86400000",
            "file.delete.delay.ms": "60000",
            "flush.messages": "9223372036854775807",
            "flush.ms": "9223372036854775807",
            "follower.replication.throttled.replicas": "",
            "index.interval.bytes": "4096",
            "leader.replication.throttled.replicas": "",
            "max.compaction.lag.ms": "9223372036854775807",
            "max.message.bytes": "1048588",
            "message.downconversion.enable": "true",
            "message.format.version": "2.6-IV0",
            "message.timestamp.difference.max.ms": "9223372036854775807",
            "message.timestamp.type": "CreateTime",
            "min.cleanable.dirty.ratio": "0.5",
            "min.compaction.lag.ms": "0",
            "min.insync.replicas": "1",
            "preallocate": "false",
            "retention.bytes": "-1",
            "retention.ms": "604800000",
            "segment.bytes": "1073741824",
            "segment.index.bytes": "10485760",
            "segment.jitter.ms": "0",
            "segment.ms": "604800000",
            "unclean.leader.election.enable": "false"
        },
        "name": "topic-3",
        "partitions": [
            {
                "leader": 1001,
                "partition": 0,
                "replicas": [
                    {
                        "broker": 1001,
                        "in_sync": true,
                        "leader": true
                    }
                ]
            }
        ]
    },
    "msg": "success"
}
```



> POST /api/topics/<topic_name>  向指定topic发送消息 (`注意body里的格式；key可以没有，value必须要有 `)

Example request:

```tex
POST /api/topics/topic-3

{
  "records": [
    {
      "key": "somekey",
      "value": {"foo": "bar"}
    },
    {
        "value":"I am a test"
    }
  ]
}
```



Example response:

```json
{
  "records": [
    {
      "key": "somekey",
      "value": {"foo": "bar"}
    },
    {
        "value":"I am a test"
    }
  ]
}
```



> POST /api/topics   创建单个或多个topic(`注意body格式，因为是单机版的所以replication_factor必须为1`)  也可以通过kafa-eagle来创建  Topic不要随意创建，慎重

Example request:

```tex
POST /api/topics

{
    "topic_names":["topic-9"],
    "num_partitions":3,
    "replication_factor":1
}
```

Example response:

```json
{
    "code": 200,
    "data": [],
    "msg": "Topic topic-8 created.   "
}
```

> DELETE /api/topics 删除单个或多个topic(`注意body格式`  也可以通过kafka-eagle来创建)   Topic不要随便删除，慎重

Example request:

```tex
DELETE /api/topics
{
	"topic_name":["topic-5"]
}
```

Example response:

```json
{
    "code": 200,
    "data": [],
    "msg": "Topic test-1 deleted.   "
}
```



#### Partitions

> GET /api/topics/<topic_name>/partitions  获取特定topic的分区数及其详细信息

Example request:

```tex
GET /api/topics/test/partitions
```

Example response:

```json
{
    "code": 200,
    "data": [
        {
            "leader": 1001,
            "partition": 0,
            "replicas": [
                {
                    "broker": 1001,
                    "in_sync": true,
                    "leader": true
                }
            ]
        },
        {
            "leader": 1001,
            "partition": 1,
            "replicas": [
                {
                    "broker": 1001,
                    "in_sync": true,
                    "leader": true
                }
            ]
        }
    ],
    "msg": "success"
}
```



> GET /api/topics/<topic_name>/partitions/<partition_id>  获取特定topic的特定分区信息

Example resquest:

```tex
GET /api/topics/test/partitions/1
```

Example response:

```tex
{
    "code": 200,
    "data": {
        "leader": 1001,
        "partition": 1,
        "replicas": [
            {
                "broker": 1001,
                "in_sync": true,
                "leader": true
            }
        ]
    },
    "msg": "success"
}
```



>GET /api/topics/<topic_name>/partitions/<partition_id>/offsets  获取特定topic的特定分区的offset的起始和终止

Example request:

```tex
GET /api/topics/test/partitions/1/offsets
```



Example response:

```json
{
    "code": 200,
    "data": {
        "beginning_offset": 0,
        "end_offset": 739
    },
    "msg": "success"
}
```



> POST /api/topics/<topic_name>/partitions/<partition_id>  向特定topic的特定分区发送消息(`注意body格式`）

Example requst:

```tex
POST /api/topics/test/partitions/partitions/0

{
  "records": [
    {
      "key": "somekey",
      "value": {"foo": "bar"}
    },
    {
      "value": 53.5
    }
  ]
}
```

Example response:

```json
{
    "code": 200,
    "data": [
        {
            "error": null,
            "error_code": null,
            "offset": 555,
            "partition": 0
        },
        {
            "error": null,
            "error_code": null,
            "offset": 556,
            "partition": 0
        }
    ],
    "msg": "success"
}
```

#### Consumers

> POST /api/consumers/<group_name>   创建消费者实例（`非常重要，注意body里的格式，除了消费实例名称,其他最好一样`)

Example request:

```tex
POST /api/consumers/group6

{
  "name": "my_consumer",
  "format": "json",
  "auto.offset.reset": "earliest",
  "auto.commit.enable": "false"
}
```



Example response:

```json
{
    "code": 200,
    "data": {
        "base_uri": "http://rest-proxy:8082/consumers/group6/instances/my_consumer",
        "instance_id": "my_consumer"
    },
    "msg": "success"
}
```



> DELETE /api/consumers/<group_name>/instances/<instance_name>  删除消费者实例

Example request:

```tex
DELETE /api/consumers/group6/instances/my_consumer
```

Example response:

```json
{
    "code": 200,
    "data": [],
    "msg": "success"
}
```



> POST /api/consumers/<group_name>/instances/<instance_name>/subscription 订阅某个主题  

Example request:

```tex
POST /api/consumers/group6/instances/my_consumer/subscription

{
  "topics": [
    "test",
    "topic-3"
  ]
}
```

Example response:

```json
{
    "code": 200,
    "data": [],
    "msg": "success"
}
```



> GET /api/consumers/<group_name>/instances/<instance_name>/subscription  查看消费者实例订阅的主题

Example request:

```tex
GET /api/consumers/group6/instances/my_consumer/subscription
```

```json
{
    "code": 200,
    "data": {
        "topics": [
            "test",
            "topic-3"
        ]
    },
    "msg": "success"
}
```



> DELETE /api/consumers/<group_name>/instances/<instance_name>/subscription  消费者实例取消订阅主题

Example request:

```tex
DELETE /api/consumers/group6/instances/my_consumer/subscription
```

Example response:

```json
{
    "code": 200,
    "data": [],
    "msg": "success"
}
```



> POST /api/consumers/<group_name>/instances/<instance_name>/assignments 手动给消费者实例分配消费分区 (`注意body的格式`)

Example request:

```tex
POST /api/consumers/group6/instances/my_consumer/assignments 

{
  "partitions": [
    {
      "topic": "test",
      "partition": 0
    },
    {
      "topic": "test",
      "partition": 1
    }
  ]
}
```

Example response:

```json
{
    "code": 200,
    "data": [],
    "msg": "success"
}
```



> GET /api/consumers/<group_name>/instances/<instance_name>/assignments 查看手动给消费者实例分配消费分区后的信息

Example request:

```tex
GET /api/consumers/group6/instances/my_consumer/assignments 
```

Example response:

```json
{
    "code": 200,
    "data": {
        "partitions": [
            {
                "partition": 0,
                "topic": "topic-3"
            }
        ]
    },
    "msg": "success"
}
```



> POST  /api/consumers/<group_name>/instances/<instance_name>/positions/beginning  手动设置offset为beginning (`注意body格式`)

Example request:

```tex
POST /api/consumers/group6/instances/my_consumer/positions/beginning

{
  "partitions": [
    {
      "topic": "test",
      "partition": 0
    },
    {
      "topic": "test",
      "partition": 1
    }
  ]
}
```

Example response:

```json
{
    "code": 200,
    "data": [],
    "msg": "success"
}
```



> POST  /api/consumers/<group_name>/instances/<instance_name>/positions/beginning  手动设置offset为end(`注意body格式`)

Example request:

```tex
POST /api/consumers/group6/instances/my_consumer/positions/beginning

{
  "partitions": [
    {
      "topic": "test",
      "partition": 0
    },
    {
      "topic": "test",
      "partition": 1
    }
  ]
}
```

Example response:

```json
{
    "code": 200,
    "data": [],
    "msg": "success"
}
```



> POST /api/consumers/<group_name>/instances/<instance_name>/positions  自定义设置offset（`注意body格式`）

Example request:

```tex
POST /api/consumers/group6/instances/my_consumer/positions

{
  "offsets": [
    {
      "topic": "test",
      "partition": 0,
      "offset": 20
    },
    {
      "topic": "test",
      "partition": 1,
      "offset": 30
    }
  ]
}
```

Example response:

```json
{
    "code": 200,
    "data": [],
    "msg": "success"
}
```



> GET /api/consumers/<group_name>/instances/<instance_name>/records 消费数据

Example request:

```tex
GET /api/consumers/group6/instances/my_consumer/records
```

Example response

```json
{
    "code": 200,
    "data": [
        {
            "key": "somekey",
            "offset": 56,
            "partition": 0,
            "topic": "topic-3",
            "value": {
                "foo": "bar"
            }
        },
        {
            "key": null,
            "offset": 57,
            "partition": 0,
            "topic": "topic-3",
            "value": "I am a test"
        }
    ],
    "msg": "success"
}
```

> POST /api/consumers/<group_name>/instances/<instance_name>/offsets 手动提交offset

Example requst:

```json
当body为空时，默认提交最后一次消费数据的offset位置
当body不为空时 
POST /api/conusmers/group/instances/consumer0/offsets
{
  "offsets": [
    {
      "topic": "test",
      "partition": 0,
      "offset": 20
    },
    {
      "topic": "test",
      "partition": 1,
      "offset": 30
    }
  ]
}
```

Example response:

```json
{
    "code": 200,
    "data": [],
    "msg": "success"
}
```

> GET /api/consumers/<group_name>/instances/<instance_name>/offsets 获取最后一次消费数据的offset位置

Example request:

```json
GET /api/conusmers/group/instances/consumer0/offsets
{
  "partitions": [
    {
      "topic": "test",
      "partition": 0
    },
    {
      "topic": "test",
      "partition": 1
    }

  ]
}
```

Example response:

```json
{
    "code":200,
    "data":{
        "offsets":[
            {
                "metadata":"",
                "offset":330,
                "partition":0,
                "topic":"test"
            }
        ]
    },
    "msg":"success"
}
```

### 4、官方地址

[kafka的github地址](https://github.com/apache/kafka)

[kafka的官方文档](http://kafka.apache.org/)

[zookeeper的github地址](https://github.com/apache/zookeeper)

[zookeeper的官方文档](https://zookeeper.apache.org/)

[kafka-rest的github地址](https://github.com/confluentinc/kafka-rest)

[kafka-rest的官方文档](https://docs.confluent.io/platform/current/kafka-rest/index.html)

