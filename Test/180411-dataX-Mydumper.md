---
title: DataX 入门测试
date: 2018-04-01 21:40:52
updated: 2018-04-10 21:40:52
categories:
  - test
tags:
  - MyDumper
  - DataX
  - TiDB
---
# DataX 入门测试

## 入门阅读

- [云栖社区](https://yq.aliyun.com/articles/59373)
- [Quick Start](https://github.com/alibaba/DataX)
- [Github repo](https://github.com/alibaba/DataX/blob/master/userGuid.md)

## MyDumper 与 DataX 对比

Project | MyDumper | DataX
--------|----------|------
数据导出 | MyDumper 是静态全量多线程导出，只能细化到表 | DataX 是流式多线程导出，可只导出指定字段
数据导入 | MyDumper 本身不支持导入功能，需要使用其他工具，消耗时间 导入+导出 | DataX 流式导入导出，数据理论上消耗 1倍，实际可能时 1.2 - 1.5 倍(取决下游插入速度)。
速率 | MyDumper 没有速率控制  | DataX 区分channel 进行控制速率，没有断点续传，脏数据与模式有关，insert 与 replace
环境准备 | MyDumper 准备 linux 环境 | DataX 需要准备 python 与 jdk 环境
配置 | MyDumper 有配置文件与命令行参数，loader 使用得 toml 方式配置 | DataX 使用的 json 语句，编写上没有语法检查很容易出错

## DataX 使用问题

- DataX 错误信息内容十分详细

```bash
经DataX智能分析,该任务最可能的错误原因是:
com.alibaba.datax.common.exception.DataXException: Code:[Framework-14], 
Description:[DataX传输脏数据超过用户预期，该错误通常是由于源端数据存在较多业务脏数据导致，请仔细检查DataX汇报的脏数据日志信息, 或者您可以适当调大脏数据阈值 .].  
- 脏数据条数检查不通过，限制是[0]条，但实际上捕获了[5727]条.
```

- DataX 启动信息
w
  ```bash
  2018-04-09 16:15:36.730 [0-0-0-reader] INFO  CommonRdbmsReader$Task - Begin to read record by Sql: [select * from sbtest1 
  ] jdbcUrl:[jdbc:mysql://127.0.0.1:34000/sbtest?yearIsDateType=false&zeroDateTimeBehavior=convertToNull&tinyInt1isBit=false&rewriteBatchedStatements=true].
  2018-04-09 16:15:46.708 [job-0] INFO  StandAloneJobContainerCommunicator - Total 0 records, 0 bytes | Speed 0B/s, 0 records/s | Error 0 records, 0 bytes |  All Task WaitWriterTime 0.000s |  All Task WaitReaderTime 0.000s | Percentage 0.00%

  2018-04-09 14:35:11.836 [job-0] INFO  StandAloneJobContainerCommunicator - Total 46548384 records, 9058041406 bytes | Speed 40.57MB/s, 218076 records/s | Error 0 records, 0 bytes |  All Task WaitWriterTime 11.739s |  All Task WaitReaderTime 289.222s | Percentage 0.00%


  2018-04-09 14:47:15.183 [job-0] INFO  StandAloneJobContainerCommunicator - Total 204485568 records, 39962122299 bytes | Speed 6.80MB/s, 36352 records/s | Error 0 records, 0 bytes |  All Task WaitWriterTime 45.961s |  All Task WaitReaderTime 997.914s | Percentage 0.00%
  ```

- DataX 隔一段时间会输出资源使用情况

```bash
2018-03-29 16:05:46.273 [job-0] INFO  VMInfo - 
	 [delta cpu info] => 
		curDeltaCpu                    | averageCpu                     | maxDeltaCpu                    | minDeltaCpu                    
		-1.00%                         | -1.00%                         | -1.00%                         | -1.00%
                        

	 [delta memory info] => 
		 NAME                           | used_size                      | used_percent                   | max_used_size                  | max_percent                    
		 PS Eden Space                  | 75.57MB                        | 23.36%                         | 280.53MB                       | 87.94%                         
		 Code Cache                     | 6.82MB                         | 90.13%                         | 7.14MB                         | 99.32%                         
		 Compressed Class Space         | 1.80MB                         | 90.02%                         | 1.80MB                         | 90.02%                         
		 PS Survivor Space              | 1.72MB                         | 20.22%                         | 3.48MB                         | 27.81%                         
		 PS Old Gen                     | 5.58MB                         | 0.82%                          | 5.58MB                         | 0.82%                          
		 Metaspace                      | 18.59MB                        | 97.87%                         | 18.59MB                        | 97.87%                         

	 [delta gc info] => 
		 NAME                 | curDeltaGCCount    | totalGCCount       | maxDeltaGCCount    | minDeltaGCCount    | curDeltaGCTime     | totalGCTime        | maxDeltaGCTime     | minDeltaGCTime     
		 PS MarkSweep         | 0                  | 0                  | 0                  | 0                  | 0.000s             | 0.000s             | 0.000s             | 0.000s             
		 PS Scavenge          | 7                  | 27                 | 7                  | 6                  | 0.034s             | 0.186s             | 0.094s             | 0.027s             
```

## DataX 使用案例

- `python datax.py -r mysqlreader -w mysqlwriter`
- `python datax.py ../job/job.json

```json
{
  "job": {
    "content": [
      {
        "reader": {
          "name": "streamreader",
          "parameter": {
            "sliceRecordCount": 10,
            "column": [
              {
                "type": "long",
                "value": "10"
              },
              {
                "type": "string",
                "value": "hello，你好，世界-DataX"
              }
            ]
          }
        },
        "writer": {
          "name": "streamwriter",
          "parameter": {
            "encoding": "UTF-8",
            "print": true
          }
        }
      }
    ],
    "setting": {
      "speed": {
        "channel": 5
       }
    }
  }
}
```

- ` python datax.py -r mysqlreader -w mysqlwriter`

```json
{
    "job": {
        "setting": {
            "speed": {
                 "channel": 10
            },
            "errorLimit": {
                "record": 0,
                "percentage": 0.02
            }
        },
        "content": [
            {
                "reader": {
                    "name": "mysqlreader",
                    "parameter": {
                        "username": "root",
                        "password": "root",
                        "column": [
                            "id",
                            "k",
                            "c",
                            "pad"
                        ],
                        "splitPk": "db_id",
                        "connection": [
                            {
                                "table": [
                                    "sbtest1"
                                ],
                                "jdbcUrl": [
     
                                ]
                            }
                        ]
                    }
                },
               "writer": {
                    "name": "streamwriter",
                    "parameter": {
                        "print":true
                    }
                }
            }
        ]
    }
}
```


```json
{
    "job": {
        "content": [
            {
                "reader": {
                    "name": "mysqlreader", 
                    "parameter": {
                        "username": "root",
                        "password": "root",
                        "column": ['*'], 
                        "splitPk": "id",
                        "connection": [
                            {
                                "jdbcUrl": ["jdbc:mysql://127.0.0.1:33/sbtest2"], 
                                "table": ["sbtest1"]
                            }
                        ],
                    }
                }, 
                "writer": {
                    "name": "mysqlwriter", 
                    "parameter": {
                        "column": ['*'], 
                        "connection": [
                            {
                                "jdbcUrl": "jdbc:mysql://127.0.0.1:4000/test", 
                                "table": ["sbtest1"]
                            }
                        ], 
                        "password": "", 
                        "preSql": [], 
                        "session": [], 
                        "username": "root", 
                        "writeMode": "insert"
                    }
                }
            }
        ], 
        "setting": {
            "speed": {
                "channel": "10"
            },
            "errorLimit": {
                "record": 0,
                "percentage": 0.02
            }
        }
    }
}
```

## DataX 导出问题

- 使用 DataX 从 TiDB 导出过程中，dataX 空跑，因为事务超过了默认 10 分钟限制，然后整段垮掉，dataX 自动重试，因此持续不断的再跑

  ```log
  2018/04/09 16:38:36.753 conn.go:481: [warning] [4] dispatch error:
  id:4, addr:127.0.0.1:50926 status:1, collation:utf8_general_ci, user:datax
  "select * from sbtest1 "
  [tikv:9006]GC life time is shorter than transaction duration
  ```
