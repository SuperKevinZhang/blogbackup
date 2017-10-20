---
title: Excel导入数据库超级DLL
date: 2009-05-15 12:01:56
tags: [Excel,C#]
---

前言:
 项目之余,发现很多项目都需要Excel导入导出的功能,每次重复代码的Coding以令我忍无可忍,
终于在一个"寂寞难耐"的周末,完成了一个Excel导入数据库(支持Sql Server 2000,2005;
Access,Oracle未测试)的程序.闲暇时间测试了一下,功能和效率(1000条3-4秒)方面还不错.
此DLL在做导入程序至少节省您50%的工作量,BUG等也会减少很,闲暇出来的时间大家可以喝喝(其实我从不喝咖啡)
咖啡(我从来不喝),看看新闻或者做其他更重要的事情.痛苦的重复工作就这样成了您的闲暇时间,
工作有时候也需要"偷懒"的.

<!--more-->

**功能:**
 最主要功能:Excel导入只需配置几下xml文件,调用个方法即可导入数据库.

  1. 支持生成SQL语句的逐条导入.
   可以配置主键,如果是主键存在,则进行更新操作,否则进行插入操作.
  2. 支持使用DataAdapter.Update()的批量导入.
   高效率的导入方式,但是导入时主键不能重复.
  3. 支持Excel的格式验证.
  4. 支持Excel中特定列值是否存在的业务验证.
  5. 支持简单的业务验证.
  6. 支持Excel中某列由Name转换为Value或ID的导入.
  7. 支持Excel中单个Sheet导入多张表的事物.
  8. 支持返回详细的错误消息(例:某行某列格式错误,或者在系统中不存在)
  9. 支持对数据库访问接口的扩展.
    10.支持格式错误或业务错误的多种方式回滚方式(单个回滚,全部回滚)
     11.支持共通字段的配置,比如创建人,创建日期等Excel不存在的列信息.
     12.代码是开源的,大家可以根据自己的要求而进行不同的更改.

不支持功能:
 1.多个Sheet同时导入数据库(后续版本可以考虑支持,实际中用到的好像不多,就没有做)
 2.不支持单条记录做为一个事物(原因:效率太低,测试1000条,耗时20秒),及导入数据库出现异常时的单条记录回顾.
使用说明:
 1.首先在您的工程中需要引入Excel2BD工程. 
 2.其次,在Config里面需要配置一下信息:

```xml
 <appSettings>

   <!--EXCEL2DB访问数据库的DLL-->

   <add key="EXCEL2DB_DLL" value="Excel2DB"/>

   <!--EXCEL2DB访问数据的类-->

   <add key="EXCEL2DB_CLASS" value="TestSqlDll"/>

   !--导入时的执行方式SQL:生成SQL语句导入,DATASET:生成DataSet整体导入--

   <add key="EXEC_TYPE" value="DATASET"/>

  </appSettings>

```


 3.配置您的导入文件(.xml文件)详细配置信息参照Sample
  注:共通字段也需要配置在您的配置文件中.
 4.程序调用.

``` c#
	  string fileName ="c://agnet.xls";

        string[] sheetName = new string[] { };

        Hashtable antherColumn = new Hashtable();

        antherColumn.Add("Import_Created", "1");

        antherColumn.Add("Created_By","Kevin");

        antherColumn.Add("Created_Date",DateTime.Now.ToString("yyyy-MM-dd"));

        string xmlfileAddress=@"D:/Project/自开发小工具/Excel2DB/web/";

        Hashtable xmlName = new Hashtable();

        xmlName.Add("MS_DEALER", "MS_DEALER");

        //xmlName.Add("CC", "C_table");

        ArrayList Messages = new ArrayList();

        Excel2DB.Excel2DB  e2b = new Excel2DB.Excel2DB();

        e2b.MainFun(fileName,sheetName,antherColumn,xmlfileAddress,xmlName,ref Messages);	
```

欢迎大家在项目中多多交流,继续完善我们的功能.

交流:
 Email:442364698@qq.com

结束语:
 相信科技,但是不要迷信于科技,从汇编语言到面向对象的第四代语言,编程效率不知提高和多少倍,
 但是项目中还是有无尽的加班. 

代码+Sample下载地址:<http://download.csdn.net/source/1312731>
