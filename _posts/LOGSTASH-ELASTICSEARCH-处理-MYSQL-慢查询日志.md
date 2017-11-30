---
title: LOGSTASH+ELASTICSEARCH 处理 MYSQL 慢查询日志
date: 2016-11-04 10:08:23
categories:
- Linux
tags: 
- ELK
- MySQL
---


# LOGSTASH+ELASTICSEARCH 处理 MYSQL 慢查询日志


先确认慢查询日志是否开启, 然后找到日志文件的位置

```
> show variables like '%slow%';
+---------------------+-------------------------------------+
| Variable_name       | Value                               |
+---------------------+-------------------------------------+
| log_slow_queries    | ON                                  |
| slow_launch_time    | 2                                   |
| slow_query_log      | ON                                  |
| slow_query_log_file | /data/mysqllog/20000/slow-query.log |
+---------------------+-------------------------------------+
```

<!--more-->

## 2. 慢查询日志
格式基本如下, 当然, 格式如果有差异, 需要根据具体格式进行小的修改

```
# Time: 160524  5:12:29
# User@Host: user_a[xxxx] @  [10.166.140.109]
# Query_time: 1.711086  Lock_time: 0.000040 Rows_sent: 385489  Rows_examined: 385489
use dbname;
SET timestamp=1464037949;
SELECT 1 from dbname;
```

## 3. 使用 logstash 采集
采集, 无非是用multiline进行多行解析
但是, 需要处理的
第一, 去除没用的信息
第二, 慢查询 sql, 是会反复出现的, 所以, 执行次数成了一个很重要的指标. 我们要做的, 就是降噪(将参数去掉, 涉及带引号的内容 + 数字), 将参数类信息过滤掉, 留下核心的 sql, 然后计算出一个 hash, 这样就可以在查询, 根据这个字段进行聚合. 这里用到了 mutate 以及 checksum
 

```
# calculate unique hash
  mutate {
    add_field => {"sql_for_hash" => "%{sql}"}
  }
  mutate {
    gsub => [
        "sql_for_hash", "'.+?'", "",
        "sql_for_hash", "-?\d*\.{0,1}\d+", ""
    ]
  }
  checksum {
    algorithm => "md5"
    keys => ["sql_for_hash"]
  }
```

最后算出来的 md5, 放入了logstash_checksum
完整的 logstash 配置文件 (具体使用可能需要根据自身日志格式做些小调整) 注意, 里面的 patternALLWORD [\s\S]*

```
input {
  file {
    path => ["/data/mysqllog/20000/slow-query.log"]
    sincedb_path => "/data/LogNew/logstash/sincedb/mysql.sincedb"
    type => "mysql-slow-log"
    add_field => ["env", "PRODUCT"]
    codec => multiline {
      pattern => "^# User@Host:"
      negate => true
      what => previous
    }
  }}filter {
  grok {
    # User@Host: logstash[logstash] @ localhost [127.0.0.1]
    # User@Host: logstash[logstash] @  [127.0.0.1]
    match => [ "message", "^# User@Host: %{ALLWORD:user}\[%{ALLWAORD}\] @ %{ALLWORD:dbhost}? \[%{IP:ip}\]" ]
  }
  grok {
    # Query_time: 102.413328  Lock_time: 0.000167 Rows_sent: 0  Rows_examined: 1970
    match => [ "message", "^# Query_time: %{NUMBER:duration:float}%{SPACE}Lock_time: %{NUMBER:lock_wait:float}%{SPACE}Rows_sent: %{NUMBER:results:int}%{SPACE}Rows_examined:%{SPACE}%{NUMBER:scanned:int}%{ALLWORD:sql}"]
  }
	// remove useless data
  mutate {
    gsub => [
        "sql", "\nSET timestamp=\d+?;\n", "",
        "sql", "\nuse [a-zA-Z0-9\-\_]+?;", "",
        "sql", "\n# Time: \d+\s+\d+:\d+:\d+", "",
        "sql", "\n/usr/local/mysql/bin/mysqld.+$", "",
        "sql", "\nTcp port:.+$", "",
        "sql", "\nTime .+$", ""
    ]
  }
	# Capture the time the query happened
  grok {
    match => [ "message", "^SET timestamp=%{NUMBER:timestamp};" ]
  }
  date {
    match => [ "timestamp", "UNIX" ]
  }
	# calculate unique hash
  mutate {
    add_field => {"sql_for_hash" => "%{sql}"}
  }
  mutate {
    gsub => [
        "sql_for_hash", "'.+?'", "",
        "sql_for_hash", "-?\d*\.{0,1}\d+", ""
    ]
  }
  checksum {
    algorithm => "md5"
    keys => ["sql_for_hash"]
  }
	# Drop the captured timestamp field since it has been moved to the time of the event
  mutate {
    # TODO: remove the message field
    remove_field => ["timestamp", "message", "sql_for_hash"]
  }}output {
    #stdout{
    #    codec => rubydebug
    #}
    #if ("_grokparsefailure" not in [tags]) {
    #    stdout{
    #        codec => rubydebug
    #    }
    #}
    if ("_grokparsefailure" not in [tags]) {
        elasticsearch {
          hosts => ["192.168.1.1:9200"]
          index => "logstash-slowlog"
        }
    }}
采集进去的内容
	{
           "@timestamp" => "2016-05-23T21:12:59.000Z",
             "@version" => "1",
                 "tags" => [
        [0] "multiline"
    ],
                 "path" => "/Users/ken/tx/elk/logstash/data/slow_sql.log",
                 "host" => "Luna-mac-2.local",
                 "type" => "mysql-slow",
                  "env" => "PRODUCT",
                 "user" => "dba_bak_all_sel",
                   "ip" => "10.166.140.109",
             "duration" => 28.812601,
            "lock_wait" => 0.000132,
              "results" => 749414,
              "scanned" => 749414,
                  "sql" => "SELECT /*!40001 SQL_NO_CACHE */ * FROM `xxxxx`;",
    "logstash_checksum" => "3e3ccb89ee792de882a57e2bef6c5371"
}
```

## 4. 写查询
查询, 我们需要按logstash_checksum进行聚合, 然后按照次数由多到少降序展示, 同时, 每个logstash_checksum需要有一条具体的 sql 进行展示
通过 es 的 Top hits Aggregation 可以完美地解决这个查询需求
查询的 query

```
	body = {
    "from": 0,
    "size": 0,
    "query": {
        "filtered": {
            "query": {
                "match": {
                    "user": "test"
                }
            },
            "filter": {
                "range": {
                    "@timestamp": {
                        "gte": "now-1d",
                        "lte": "now"
                    }
                }
            }
        }
    },
    "aggs": {
        "top_errors": {
            "terms": {
                "field": "logstash_checksum",
                "size": 20
            },
            "aggs": {
                "top_error_hits": {
                    "top_hits": {
                        "sort": [
                            {
                                "@timestamp":{
                                    "order": "desc"
                                }
                            }
                        ],
                        "_source": {
                            "include": [
                               "user" , "sql", "logstash_checksum", "@timestamp", "duration", "lock_wait", "results", "scanned"
                            ]
                        },
                        "size" : 1
                    }
                }
            }
        }
    }
}
```

跟这个写法相关的几个参考链接: Terms Aggregation / Elasticsearch filter document group by field
## 5. 渲染页面
python 的后台, 使用sqlparse包, 将 sql 进行格式化 (换行 / 缩进 / 大小写), 再往前端传. sqlparse

```
>>> sql = 'select * from foo where id in (select id from bar);'
>>> print sqlparse.format(sql, reindent=True, keyword_case='upper')
SELECT *
FROM foo
WHERE id IN
  (SELECT id
   FROM bar);
```


