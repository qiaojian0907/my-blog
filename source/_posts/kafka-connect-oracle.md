---
title: kafka-connect-oracle
date: 2022-07-28 17:15:05
index_img: /image/oracle/img.png
banner_img: /image/oracle/img.png
tags: [oracle]
categories: "数据库"
---

## 1.基础环境配置 ##



> 安装zookeeper、kafka

linux下安装zookeeper  [https://www.cnblogs.com/expiator/p/9853378.html](https://www.cnblogs.com/expiator/p/9853378.html)

linux下安装kafka  [https://juejin.cn/post/6844903745772322824](https://juejin.cn/post/6844903745772322824)


## 2.oracle安装及配置 ##

### 以oracle12c为例 ###

- 使用docker安装oracle12c

[https://blog.csdn.net/Damionew/article/details/84566718](https://blog.csdn.net/Damionew/article/details/84566718)

- oracle开启LogMiner

检查数据库日志模式

	select log_mode from v$database;
	
	LOG_MODE
	------------
	ARCHIVELOG

开启LogMiner需要用拥有DBA权限的用户登录Oracle进行开启。


    sqlplus / as sysdba
    SQL>shutdown immediate;
    SQL>startup mount;
    SQL>alter database archivelog;
    SQL>alter database open;

启用补充日志记录

    sqlplus / as sysdba
    SQL>alter database add supplemental log data (all) columns;

为了成功执行连接器，必须使用特权Oracle用户启动连接器。

	create role <role_name>;
	grant create session to <role_name>;
	grant  execute_catalog_role,select any transaction ,select any dictionary,logmining to <role_name>;
	create user <user_name> identified by <user_pwd>;
	grant  <role_name> to <user_name>;
	alter user<user_name> quota unlimited on users ;

添加建表权限：

	grant create any table to <user_name>;


## 3.安装kafka-connect-oracle ##

### 3.1 安装包获取###



- 官方地址：

[https://github.com/erdemcer/kafka-connect-oracle](https://github.com/erdemcer/kafka-connect-oracle)

下载项目后mvn clean package 打成kafka-connect-oracle-1.0.jar。

- 下载oracle的jdbc驱动包

- 将oracle驱动包及kafka-connect-oracle-1.0.jar放至kafka安装路径的lib目录下。

- 将github项目里面的config/OracleSourceConnector.properties文件拷贝到kafak/config。

### 3.2 OracleSourceConnector.properties配置文件详解： ###

    name=oracle-logminer-connector
    connector.class=com.ecer.kafka.connect.oracle.OracleSourceConnector
    db.name.alias=test
    tasks.max=1
    topic=test-oracle-hdc
    db.name=xe
    db.hostname=172.18.26.194
    db.port=1521
    db.user=hdc
    db.user.password=hdc
    db.fetch.size=1
    table.whitelist=HDC.*
    parse.dml.data=true
    reset.offset=true
    start.scn=
    multitenant=false
    table.blacklist=

详情参考：[https://github.com/erdemcer/kafka-connect-oracle](https://github.com/erdemcer/kafka-connect-oracle)

### 3.3 启动 ###

启动connector：

Kafka connect的工作模式分为两种，分别是standalone模式和distributed模式。


standalone模式启动：

    bin/connect-standalone.sh -daemon config/connect-standalone.properties config/OracleSourceConnector.properties

distributed模式启动：

	bin/connect-distributed.sh -daemon config/connect-distributed.properties

在connect-distributed.properties的配置文件中，其实并没有配置了你的connector的信息，因为在distributed模式下，启动不需要传递connector的参数，而是通过REST API来对kafka connect进行管理，包括启动、暂停、重启、恢复和查看状态的操作。

在启动kafkaconnect的distributed模式之前，首先需要创建三个主题，这三个主题的配置分别对应connect-distributed.properties文件中config.storage.topic(default connect-configs)、offset.storage.topic (default connect-offsets) 、status.storage.topic (default connect-status)的配置。



* config.storage.topic：用以保存connector和task的配置信息，需要注意的是这个主题的分区数只能是1，而且是有多副本的。（推荐partition 1，replica 1）

* offset.storage.topic:用以保存offset信息。（推荐partition50，replica 1）

* status.storage.topic:用以保存connetor的状态信息。（推荐partition10，replica 1）

以下是创建主题命令：

##### config.storage.topic=connect-configs

	bin/kafka-topics.sh --create --zookeeper localhost:2181 --topic connect-configs --replication-factor 1 --partitions 1 --config cleanup.policy=compact

##### offset.storage.topic=connect-offsets

	bin/kafka-topics.sh --create --zookeeper localhost:2181 --topic connect-offsets --replication-factor 1 --partitions 50 --config cleanup.policy=compact

##### status.storage.topic=connect-status

	bin/kafka-topics.sh --create --zookeeper localhost:2181 --topic connect-status --replication-factor 1 --partitions 10 --config cleanup.policy=compact


kafka-comsumer消费：

	bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --from-beginning --topic test-oracle-hdc

topic常用命令:

	1、创建topic
	
	bin/kafka-topics.sh --create --bootstrap-server localhost:9092 --replication-factor 1 --partitions 1 --topic test1
	
	2、查看所有topic
	
	bin/kafka-topics.sh --list --bootstrap-server localhost:9092
	
	3、产生console消息
	
	bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test
	
	4、消费console消息
	
	bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test1 --from-beginning
	
	5、显示test1 topic详情
	
	bin/kafka-topics.sh --describe --bootstrap-server localhost:9092 --topic test1
	
	6、删除topic test1
	
	bin/kafka-topics.sh --delete --bootstrap-server localhost:9092 --topic test1

执行结果：

DDL：

	{
	"schema": {
	"type": "struct",
	"fields": [
	  {
	"type": "int64",
	"optional": false,
	"field": "SCN"
	  },
	  {
	"type": "string",
	"optional": false,
	"field": "SEG_OWNER"
	  },
	  {
	"type": "string",
	"optional": false,
	"field": "TABLE_NAME"
	  },
	  {
	"type": "int64",
	"optional": false,
	"name": "org.apache.kafka.connect.data.Timestamp",
	"version": 1,
	"field": "TIMESTAMP"
	  },
	  {
	"type": "string",
	"optional": false,
	"field": "SQL_REDO"
	  },
	  {
	"type": "string",
	"optional": false,
	"field": "OPERATION"
	  },
	  {
	"type": "struct",
	"fields": [
	  
	],
	"optional": true,
	"field": "data"
	  },
	  {
	"type": "struct",
	"fields": [
	  
	],
	"optional": true,
	"field": "before"
	  }
	],
	"optional": false,
	"name": "test.hdc.user_info.row"
	  	},
	"payload": {
	"SCN": 1879834,
	"SEG_OWNER": "HDC",
	"TABLE_NAME": "_GENERIC_DDL",
	"TIMESTAMP": 1622138532000,
	"SQL_REDO": "create table USER_INFO\n(\n  ID   NUMBER(11) not null,   \n  NAME   VARCHAR2(50) not null,   \n  AGE   NUMBER(5) not null\n  \n)",
	"OPERATION": "DDL",
	"data": null,
	"before": null
	  }
	}


DML:

insert：

    {
      "schema": {
    "type": "struct",
    "fields": [
      {
    "type": "int64",
    "optional": false,
    "field": "SCN"
      },
      {
    "type": "string",
    "optional": false,
    "field": "SEG_OWNER"
      },
      {
    "type": "string",
    "optional": false,
    "field": "TABLE_NAME"
      },
      {
    "type": "int64",
    "optional": false,
    "name": "org.apache.kafka.connect.data.Timestamp",
    "version": 1,
    "field": "TIMESTAMP"
      },
      {
    "type": "string",
    "optional": false,
    "field": "SQL_REDO"
      },
      {
    "type": "string",
    "optional": false,
    "field": "OPERATION"
      },
      {
    "type": "struct",
    "fields": [
      {
    "type": "int64",
    "optional": false,
    "field": "ID"
      },
      {
    "type": "string",
    "optional": false,
    "field": "NAME"
      },
      {
    "type": "int32",
    "optional": false,
    "field": "AGE"
      }
    ],
    "optional": true,
    "name": "value",
    "field": "data"
      },
      {
    "type": "struct",
    "fields": [
      {
    "type": "int64",
    "optional": false,
    "field": "ID"
      },
      {
    "type": "string",
    "optional": false,
    "field": "NAME"
      },
      {
    "type": "int32",
    "optional": false,
    "field": "AGE"
      }
    ],
    "optional": true,
    "name": "value",
    "field": "before"
      }
    ],
    "optional": false,
    "name": "test.hdc.user_info.row"
      },
      "payload": {
    "SCN": 1880255,
    "SEG_OWNER": "HDC",
    "TABLE_NAME": "USER_INFO",
    "TIMESTAMP": 1622139201000,
    "SQL_REDO": "insert into \"HDC\".\"USER_INFO\"(\"ID\",\"NAME\",\"AGE\") values (2,'test1',22)",
    "OPERATION": "INSERT",
    "data": {
      "ID": 2,
      "NAME": "test1",
      "AGE": 22
    },
    "before": null
      }
    }

update：

    {
      "schema": {
    "type": "struct",
    "fields": [
      {
    "type": "int64",
    "optional": false,
    "field": "SCN"
      },
      {
    "type": "string",
    "optional": false,
    "field": "SEG_OWNER"
      },
      {
    "type": "string",
    "optional": false,
    "field": "TABLE_NAME"
      },
      {
    "type": "int64",
    "optional": false,
    "name": "org.apache.kafka.connect.data.Timestamp",
    "version": 1,
    "field": "TIMESTAMP"
      },
      {
    "type": "string",
    "optional": false,
    "field": "SQL_REDO"
      },
      {
    "type": "string",
    "optional": false,
    "field": "OPERATION"
      },
      {
    "type": "struct",
    "fields": [
      {
    "type": "int64",
    "optional": false,
    "field": "ID"
      },
      {
    "type": "string",
    "optional": false,
    "field": "NAME"
      },
      {
    "type": "int32",
    "optional": false,
    "field": "AGE"
      }
    ],
    "optional": true,
    "name": "value",
    "field": "data"
      },
      {
    "type": "struct",
    "fields": [
      {
    "type": "int64",
    "optional": false,
    "field": "ID"
      },
      {
    "type": "string",
    "optional": false,
    "field": "NAME"
      },
      {
    "type": "int32",
    "optional": false,
    "field": "AGE"
      }
    ],
    "optional": true,
    "name": "value",
    "field": "before"
      }
    ],
    "optional": false,
    "name": "test.hdc.user_info.row"
      },
      "payload": {
    "SCN": 1882426,
    "SEG_OWNER": "HDC",
    "TABLE_NAME": "USER_INFO",
    "TIMESTAMP": 1622143353000,
    "SQL_REDO": "update \"HDC\".\"USER_INFO\" set \"NAME\" = 'test-3' where \"ID\" = 3 and \"NAME\" = 'test3' and \"AGE\" = 33",
    "OPERATION": "UPDATE",
    "data": {
      "ID": 3,
      "NAME": "test-3",
      "AGE": 33
    },
    "before": {
      "ID": 3,
      "NAME": "test3",
      "AGE": 33
    }
      }
    }


## 4.通过rest api管理connector ##

因为kafka connect的意图是以服务的方式去运行，所以它提供了REST API去管理connectors，默认的端口是8083，你也可以在启动kafka connect之前在配置文件中添加rest.port配置。

- GET /connectors – 返回所有正在运行的connector名
- POST /connectors – 新建一个connector; 请求体必须是json格式并且需要包含name字段和config字段，name是connector的名字，config是json格式，必须包含你的connector的配置信息。
- GET /connectors/{name} – 获取指定connetor的信息
- GET /connectors/{name}/config – 获取指定connector的配置信息
- PUT /connectors/{name}/config – 更新指定connector的配置信息
- GET /connectors/{name}/status – 获取指定connector的状态，包括它是否在运行、停止、或者失败，如果发生错误，还会列出错误的具体信息。
- GET /connectors/{name}/tasks – 获取指定connector正在运行的task。
- GET /connectors/{name}/tasks/{taskid}/status – 获取指定connector的task的状态信息
- PUT /connectors/{name}/pause – 暂停connector和它的task，停止数据处理知道它被恢复。
- PUT /connectors/{name}/resume – 恢复一个被暂停的connector
- POST /connectors/{name}/restart – 重启一个connector，尤其是在一个connector运行失败的情况下比较常用
- POST /connectors/{name}/tasks/{taskId}/restart – 重启一个task，一般是因为它运行失败才这样做。
- DELETE /connectors/{name} – 删除一个connector，停止它的所有task并删除配置。