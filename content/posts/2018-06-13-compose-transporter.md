---
title: "Compose Transporter -- 基于阿里云MongoDB和ES之间的数据同步" 
date: 2018-06-13T21:59:26+08:00
lastmod: 2018-06-13T21:59:26+08:00
keywords: [compose, transporter, elasticsearch, mongodb]
tags: [compose, transporter, elasticsearch, mongodb]
categories: [mongodb, elasticsearch]
author: "GeorgeHao"
---

## 工具选型

主要查看[这篇文章](https://code.likeagirl.io/5-different-ways-to-synchronize-data-from-mongodb-to-elasticsearch-d8456b83d44f)

* [mongo-connector](https://github.com/mongodb-labs/mongo-connector)
* [elasticsearch-river-mongodb](https://github.com/richardwilly98/elasticsearch-river-mongodb)
* [Logstash](https://www.elastic.co/guide/en/logstash/current/plugins-inputs-jdbc.html)
* [transporter](https://github.com/compose/transporter)
* [Mongoosastic](https://www.compose.com/articles/mongoosastic-the-power-of-mongodb-and-elasticsearch-together/)

`mongo-connector`是[mongodb-labs](https://github.com/mongodb-labs/mongo-connector)官方的开源项目. 但是官方不在支持, 关闭了`issue`, 导致很多问题无法找到答案, 也就无法解决遇到的问题

> mongo-connector is not currently supported by MongoDB, Inc. If any community members would like to take over maintenance, please contact seth.payne@mongodb.com`

`elasticsearch-river-mongodb`不支持`elasticsearch`5.0以版本.

最终选择`Transporter`.


## 选择`Transporter`原因

1. 项目活跃度较高
2. 实时同步
3. 配置简单
4. `golang`写的(golang忠实粉丝)
5. IBM公司产品

开头文章提到的问题:

> It’s important to know that the transporter synchronizing only once. When the job is done, the transporter comes to its end.

这个问题已经不存在了. 查看这个[文章](https://www.compose.com/articles/transporter-mongodb-and-synchronization/)

> And the Transporter starts copying the collection... and when it's done it stops running. That's great for this collection because of its historical data. But what if that collection was live data; how would we manage to copy it consistently. With the MongoDB adaptor, there's an option to tail the oplog, MongoDB's replication trail and this lets programs see changes in real time. Turn the option on and when the initial copying has finished, the Transporter stays running listening to the oplog and creating new messages which contain documents with all the changes. So all you need to add to the source properties is "tail": true.

可以看出来, `transporter`和`mongo-connector`都是监听`oplog`的变化来实现数据的实时同步. 

## Oplog

### 查看`oplog`大小

```
db.printReplicationInfo()
db.oplog.rs.stats().maxSize 单位k
```

一般复制集的oplog size 默认是磁盘容量的5%(最小1G，最大50G）

### 如何估算oplog大小

http://www.mongoing.com/blog/oplog-size

### 如何修改oplog大小

官方文档: https://docs.mongodb.com/v3.4/tutorial/change-oplog-size/

不过阿里云官方不支持修改oplog, 所以如果遇到oplog过小导致同步失败的问题, 只能增加磁盘大小

## transporter使用

官方: https://github.com/compose/transporter 

博客: https://www.compose.com/articles/search/?s=transporter

WIKI: https://github.com/compose/transporter/wiki

Compose官网: https://www.compose.com/

### 命令

```
run       run pipeline loaded from a file
test      display the compiled nodes without starting a pipeline
about     show information about available adaptors
init      initialize a config and pipeline file based from provided adaptors
xlog      manage the commit log
offset    manage the offset for sinks
```

### 查看支持的adaptors

```
root@364293f04e45:~# ./transporter about
rabbitmq - an adaptor that handles publish/subscribe messaging with RabbitMQ
rethinkdb - a rethinkdb adaptor that functions as both a source and a sink
elasticsearch - an elasticsearch sink adaptor
file - an adaptor that reads / writes files
mongodb - a mongodb adaptor that functions as both a source and a sink
postgres - a postgres adaptor that functions as both a source and a sink
```

### 初始化配置文件

```
root@364293f04e45:~# ./transporter init mongodb elasticsearch
Writing pipeline.js...
```
会生成一个`pipeline.js`

### 配置文件

默认:

```
var source = mongodb({
  "uri": "${MONGODB_URI}"
  // "timeout": "30s",
  // "tail": false,
  // "ssl": false,
  // "cacerts": ["/path/to/cert.pem"],
  // "wc": 1,
  // "fsync": false,
  // "bulk": false,
  // "collection_filters": "{}",
  // "read_preference": "Primary"
})

var sink = elasticsearch({
  "uri": "${ELASTICSEARCH_URI}"
  // "timeout": "10s", // defaults to 30s
  // "aws_access_key": "ABCDEF", // used for signing requests to AWS Elasticsearch service
  // "aws_access_secret": "ABCDEF" // used for signing requests to AWS Elasticsearch service
  // "parent_id": "elastic_parent" // defaults to "elastic_parent" parent identifier for Elasticsearch
})

t.Source("source", source, "/.*/").Save("sink", sink, "/.*/")
```

主要参考[Configuration](https://github.com/compose/transporter/wiki/Configuration), [Pipelines](https://github.com/compose/transporter/wiki/Pipelines), [Transformers](https://github.com/compose/transporter/wiki/Transformers) 进行配置.

`t.Source("source", source, "/.*/").Save("sink", sink, "/.*/")`的`namespaces`(`/.*/`)是一个正则表达式. 以`^`开头并以`$`结尾

关于这一块查看issue的时候要注意, 尽量不看`2017年`之前的`issue`, 因为在[0.3.0](https://www.compose.com/articles/transporter-0-3-0-released-transporter-streamlined/)改变了`transporter`的使用方式. 尽量按照wiki的方式来. 

我的配置文件
```
var source = mongodb({
  "uri": "mongodb://**:**@dds-2ze007b62390d8241.mongodb.rds.aliyuncs.com:3717/story?authSource=admin"
  // "timeout": "30s",
  "tail": true,
  // "ssl": false,
  // "cacerts": ["/path/to/cert.pem"],
  // "wc": 1,
  // "fsync": false,
  // "bulk": false,
  //"collection_filters": "{ }",
  // "read_preference": "Primary"
})


var sink = elasticsearch({
  "uri": "http://***:***@es-cn-mp90mbej1000dyum9.elasticsearch.aliyuncs.com:9200/story"
  // "timeout": "10s", // defaults to 30s
  // "aws_access_key": "ABCDEF", // used for signing requests to AWS Elasticsearch service
  // "aws_access_secret": "ABCDEF" // used for signing requests to AWS Elasticsearch service
  // "parent_id": "elastic_parent" // defaults to "elastic_parent" parent identifier for Elasticsearch
})

t.Source("source", source, "/^tracks$|^albums$|^picture_books$|^picture_books_items$/").Transform(omit({"fields":["code"]})).Save("sink", sink, "/^tracks$|^albums$|^picture_books$|^picture_books_items$/")
```

还存在两个问题:

1. 如何`mongodb://**:**@dds-2ze007b62390d8241.mongodb.rds.aliyuncs.com:3717/story?authSource=admin`换成MongoDB connection string?
2. 如何在同一脚本中同时同步多个数据库?

### 测试配置文件是否正常

```
root@364293f04e45:~# ./transporter test
Transporter:
 - Source:         source                                   mongodb         ^tracks$|^albums$|^picture_books$|^picture_books_items$
  - Sink:          sink                                     elasticsearch   ^tracks$|^albums$|^picture_books$|^picture_books_items$
```

### 运行

```
root@364293f04e45:~# ./transporter run
INFO[0000] adaptor Listening...                          name=sink path="source/sink" type=elasticsearch
INFO[0000] starting with metadata map[]                  name=source path=source type=mongodb
INFO[0000] adaptor Starting...                           name=source path=source type=mongodb
INFO[0000] boot map[source:mongodb sink:elasticsearch]   ts=1529296920584180591
INFO[0000] testing oplog access                          uri="mongodb://**:**@dds-2ze007b62390d8241.mongodb.rds.aliyuncs.com:3717/story?authSource=admin"
INFO[0000] oplog access good
INFO[0000] starting Read func                            db=story
INFO[0000] Establishing new connection to dds-2ze007b62390d8241.mongodb.rds.aliyuncs.com:3717 (timeout=1h0m0s)...
INFO[0000] Connection to dds-2ze007b62390d8241.mongodb.rds.aliyuncs.com:3717 established.
INFO[0000] collection count                              db=story num_collections=39
INFO[0000] skipping iteration...                         collection=abc db=story
INFO[0000] adding for iteration...                       collection=albums db=story
INFO[0000] skipping iteration...                         collection="cartoon_albums" db=story
INFO[0000] skipping iteration...                         collection=categories db=story
INFO[0000] skipping iteration...                         collection="category_groups" db=story
INFO[0000] skipping iteration...                         collection="child_ugcs" db=story
INFO[0000] skipping iteration...                         collection="feedback_categories" db=story
INFO[0000] skipping iteration...                         collection=feedbacks db=story
INFO[0000] skipping iteration...                         collection=handpicks db=story
INFO[0000] skipping iteration...                         collection=languages db=story
INFO[0000] skipping iteration...                         collection="picture_book_items" db=story
INFO[0000] adding for iteration...                       collection="picture_books" db=story
INFO[0000] skipping iteration...                         collection=system.profile db=story
INFO[0000] skipping iteration...                         collection=tags db=story
INFO[0000] skipping iteration...                         collection=testcoll db=story
INFO[0000] adding for iteration...                       collection=tracks db=story
INFO[0000] done iterating collections                    db=story
INFO[0000] iterating...                                  collection=albums
INFO[0000] Establishing new connection to dds-2ze007b62390d8241.mongodb.rds.aliyuncs.com:3717 (timeout=1h0m0s)...
INFO[0000] Connection to dds-2ze007b62390d8241.mongodb.rds.aliyuncs.com:3717 established.
INFO[0000] iterating complete                            collection=albums db=story
INFO[0000] oplog start timestamp: 6568280257273528320    collection=albums
INFO[0000] tailing oplog with query map[ns:story.albums ts:map[$gte:6568280257273528320]]  db=story
INFO[0000] iterating...                                  collection="picture_books"
INFO[0000] Establishing new connection to dds-2ze007b62390d8241.mongodb.rds.aliyuncs.com:3717 (timeout=1h0m0s)...
INFO[0000] Connection to dds-2ze007b62390d8241.mongodb.rds.aliyuncs.com:3717 established.
^CINFO[0000] adaptor Stopping...                           name=source path=source type=mongo
```

默认打印`INFO`级别的log, 其他级别的还有`debug`,`error`, 只需要加上参数: `-log.level "debug"`

### Docker运行

https://github.com/compose/transporter/wiki/Running-with-Docker

```
docker pull quay.io/compose/transporter
docker run --rm -v $(pwd):/workdir -w /workdir quay.io/compose/transporter:latest transporter init mongodb file
#!/bin/sh
docker run --rm -v $(pwd):/workdir -w /workdir quay.io/compose/transporter:latest transporter run
```

### 创建mapping

在使用`transporter`的时候最好要提前创建好index, 这样通过到ES的数据才会按照我们想要的格式建立索引. 如果不提前设置创建index, 那么ES需要打开`自动创建索引`选项, 不建议这么做.

#### 创建index
```
{
    "index":{
        "analysis":{
            "analyzer":{
                "by_smart":{
                    "type":"custom",
                    "tokenizer":"ik_smart",
                    "filter":[
                        "by_tfr",
                        "by_sfr"
                    ],
                    "char_filter":[
                        "by_cfr"
                    ]
                },
                "by_max_word":{
                    "type":"custom",
                    "tokenizer":"ik_max_word",
                    "filter":[
                        "by_tfr",
                        "by_sfr"
                    ],
                    "char_filter":[
                        "by_cfr"
                    ]
                }
            },
            "filter":{
                "by_tfr":{
                    "type":"stop",
                    "stopwords":[
                        " "
                    ]
                },
                "by_sfr":{
                    "type":"synonym",
                    "synonyms_path":"analysis/synonyms.txt"
                }
            },
            "char_filter":{
                "by_cfr":{
                    "type":"mapping",
                    "mappings":[
                        "| => |"
                    ]
                }
            }
        }
    }
}
```

#### 创建type

```
{
	"properties": {
	    "title": {
	    	"type": "text",
	    	"fields": {
		    	"keyword": {
		        	"type": "keyword",
		        	"ignore_above": 256
		    	}
	    	},
			"index": "analyzed",
		    "analyzer": "by_max_word",
		    "search_analyzer": "by_smart"
    	},
    	"authors": {
    		"properties":{
    			"name": {
    				"type": "text",
			    	"fields": {
				    	"keyword": {
				        	"type": "keyword",
				        	"ignore_above": 256
				    	}
			    	},
					"index": "analyzed",
				    "analyzer": "by_max_word",
				    "search_analyzer": "by_smart"		
    			}
    		}
    	},
    	"nickname": {
	    	"type": "text",
	    	"fields": {
		    	"keyword": {
		        	"type": "keyword",
		        	"ignore_above": 256
		    	}
	    	},
			"index": "analyzed",
		    "analyzer": "by_max_word",
		    "search_analyzer": "by_smart"
    	}
	}
}
```