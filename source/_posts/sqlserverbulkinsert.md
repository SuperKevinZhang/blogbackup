---
title: 大批量导入数据库 BULK INSERT
date: 2007-08-01 11:43:26
tags: [数据库,大批量导入,BULK INSERT]
---

**大批量导入数据库 自己写的存储过程**

很多年前做的一个项目,上线时需要将多个几百兆甚至上G的txt文件导入到sql server数据库中,使用buck 语法,可以非常快的速度导入,主要不要开启事物,否则会非常的慢

<!--more-->

***

``` sql
CREATE PROC [dbo].[sp_Import_TT]

@TableName NVARCHAR(50) ,  --导入的表名称

@DataFile NVARCHAR(100) , --数据源文件

@FormatFile NVARCHAR(100), --格式文件

@LogFile NVARCHAR(100) --日志文件

AS

 EXEC('BULK INSERT '+@TableName+'

                          FROM '''+@DataFile+'''

                          WITH 

                          (

                             FORMATFILE = '''+@FormatFile+''',  

                             ROWS_PER_BATCH =10,                --错误数回滚，如果错误超过此数，将整个事务回滚

                             ERRORFILE = '''+@LogFile+'''                     

                           )'

      )



```



**附：格式文件类型**
9.0                         
9                          
1    SQLCHAR    0    20    ""    1    CWB_NO    "" 
2    SQLCHAR    0    20    ""    2    AcctID    "" 
3    SQLCHAR    0    10    ""    3    SERVICE_ID    "" 
4    SQLCHAR    0    3    ""    4    DestSZMCode    "" 
5    SQLCHAR    0    10    ""    5    ActualWeight    ""
6    SQLCHAR    0    10    ""    6    VolumeWeight    ""
7    SQLCHAR    0    2    ""    7    CountryCode    "" 
8    SQLCHAR    0    8    ""    8    Clt_Datetime    ""
9    SQLCHAR    0    14    "/r/n"    47    PICKUP_TIME    ""
 

**bcp导出数据:**

``` sql
EXEC master..xp_cmdshell 'BCP OCS_Links_sha.DBO.tb_customer out D:/DATA.txt -c -F"2" -S"192.168.12.129/ocs" -U"sa" -P"111111"'
```



**bcp 导出format文件:**

``` sql
EXEC master..xp_cmdshell 'BCP OCS_Links_sha.DBO.tb_customer format nul -f D:/FMT.txt -c -S"192.168.12.129/ocs" -U"sa" -P"111111"'
```



 ``` sql
  --,如果遇到阻止Shell,请使用以下SQL启用shell

  EXEC sp_configure 'show advanced options', 1;RECONFIGURE;EXEC sp_configure 'xp_cmdshell', 1;RECONFIGURE

 ```



  
