---
title: SQL Server 通过分析执行计划优化视图
date: 2017-12-05 16:32:48
tags: [数据库优化]
---

​						**SQL Server 通过分析执行计划优化视图**

​	最近接到一个优化的咨询,这家公司的系统建立与2004年,目前已经已经运行14之久,这大概是我遇到运行最久的主业务系统了;

​	**项目简介:**

- 物流公司业务基干系统(一个基干系统能用14年,厉害)

- 数据库:SQL Server 2000

  - 开发语言:Visual Basic(注意是VB,不是VB.NET)

  - 系统架构:C/S

    **问题描述:**

    ​       系统太久远,当时开发的人员只剩下了一个,且快到了退休年龄,年轻人已经没几个人会VB了.现在每一个业务改动都非常困难,且系统运行效率特别低.有时候一个操作需要20秒以上;

    ​<!--more-->

    ****解决方案:****

    ​      长远的解决方案当然是重新开发系统,按照最新的公司战略和当下的技术结构重构系统.这这不是一蹴而就的事情,对于解决方案供应商第一任务是取得客户信任,然后再规划将来的事情;对于客户来说,比较重要的事情如何解决目前效率的问题.所以我们做的第一件事情是通过解决客户效率问题取得客户信任.VB的代码牵涉众多,不适合早期的优化,早期优化从优化数据库开始;

    **数据库优化**

    ​       我们拿最常用的一个批货视图开始优化;这个视图拿到的第一眼是晕掉的,查询这么多表,这么多子查询和计算.在我们的测试机上执行时间为52秒;

    ```sql
    SET QUOTED_IDENTIFIER ON 
    GO
    SET ANSI_NULLS ON 
    GO

    ALTER  VIEW 	_Consoled
    AS
    SELECT 	MAWB.MAWB, 	MAWB.FlightDate, CASE WHEN 	Mawb.Agent IS NULL 
          THEN 	MAWB.Flight + 	Mawb.Des3 ELSE 	Mawb.Flight + 	Mawb.Des3 + '('
           + 	Mawb.Agent + ')' END AS Flight, 	MAWB.Flight AS MawbFlight, 
          	MAWB.Dest1, 	MAWB.Dest2, 	HAWB.WareHouseSN, 
          	Customer.CustName, 	MAWB.Agent, 	MAWB.Pieces1, 	MAWB.Gross1, 
          	MAWB.V1, 	MAWB.Weight1, 	MAWB.Pieces2, 	MAWB.Gross2, 
          	MAWB.V2, 	MAWB.Weight2, 
          CASE WHEN MAWB.Status16 = 1 THEN '自报关' ELSE '' END + 	MAWB.OP AS OP, 
          CASE 	MAWB.Printer WHEN '' THEN '' ELSE '总' END AS Printed, 
          	HAWB.Pieces2 AS PCS, 	HAWB.Gross4 AS WGT, 	HAWB.V4 AS VLM, 
          CASE 	HAWB.Num3 WHEN 0 THEN '' ELSE '*' END AS Broken, 
          CASE 	Hawb.HWStatus WHEN 0 THEN ltrim(str(	Hawb.Pieces1)) 
          WHEN 2 THEN ltrim(str(	Hawb.Pieces1)) + '(' + ltrim(str(	Hawb.Pieces2)) 
          + ')' ELSE ltrim(str(	Hawb.Pieces2)) END AS Pieces, 
          CASE 	Hawb.HWStatus WHEN 0 THEN 	Hawb.Gross1 ELSE 	Hawb.Gross4 END
           AS Gross, 
          CASE 	Hawb.HWStatus WHEN 0 THEN 	HAWB.V1 ELSE 	Hawb.V4 END AS V,
           CASE 	Hawb.HWStatus WHEN 0 THEN 	HAWB.Weight1 ELSE 	Hawb.Weight4
           END AS Weight, 	HAWB.HAWB, 	HAWB.BGNote, 
          CASE 	Hawb.Status11 WHEN 0 THEN '' ELSE '*' END AS DCZ, 
          CASE WHEN BGDS.BGDs IS NULL 
          THEN '' ELSE CASE BGDS.Status8 WHEN 0 THEN '*' ELSE '**' END END + 	HAWB.Des4
           + CASE 	Hawb.Num10 WHEN 0 THEN '' ELSE '扣' END + CASE 	Hawb.HWStatus
           WHEN 0 THEN ' ' WHEN 1 THEN 'H' WHEN 2 THEN 'B' WHEN 3 THEN 'C' END + CASE
           	Hawb.Num3 WHEN 0 THEN '' ELSE '破' END + CASE 	HAWB.Status26 WHEN 2
           THEN 'T' ELSE '' END + CASE 	Hawb.DanZhengStatus WHEN 0 THEN ' ' ELSE CASE
           	HAWB.Status19 WHEN 0 THEN 'D' ELSE 'M' END END + CASE 	Hawb.Status14
           WHEN 0 THEN '  ' WHEN 1 THEN '到' WHEN 2 THEN '签' WHEN 3 THEN '拆' WHEN 4 THEN
           '问' WHEN 5 THEN '放' WHEN 6 THEN '审' WHEN 7 THEN '计' WHEN 8 THEN '完' END
           + CASE 	Hawb.Status9 WHEN 0 THEN ' ' ELSE 'S' END + CASE 	Hawb.FStatus WHEN
           0 THEN ' ' ELSE 'A' END + CASE 	Hawb.QStatus WHEN 0 THEN ' ' ELSE 'P' END + CASE
           	Hawb.Status16 WHEN 0 THEN '  ' ELSE '标' END + CASE WHEN Housing.Category
           IS NULL 
          THEN '' ELSE Housing.Category END + CASE MAWB.Status13 WHEN 1 THEN '区内' WHEN
           2 THEN '区外' ELSE '' END + CASE WHEN 	outPlans.WareHouseSN IS NULL 
          THEN '' ELSE '排' END  AS
           Status, 	HAWB.Status14, 	HAWB.Status17, 	HAWB.Status27, 
          Housing.STPosition, Housing.Category, 	MAWB.Status10, 	MAWB.Status11, 
          	MAWB.Status12, 	MAWB.Status13, 	Destination.City, 	AirLines.AirLine, 
          	MAWB.AirPort, 	HAWB.Dest, 	HAWB.Num1, 	HAWB.ProcedureStatus, 
          	HAWB.PrintOut, 	HAWB.PrintRequest, 	MAWB.Status2, 
          	MAWB.Status7, 	MAWB.Status8, 	MAWB.Status4, 	MAWB.Des1, 
          CASE WHEN 	MAWB.DeclareDate IS NULL OR
          	MAWB.DeclareDate = 	MAWB.FlightDate THEN '' ELSE '报关日' + RIGHT(CONVERT(Varchar(10),
           	MAWB.DeclareDate, 120), 5) END AS DeclareDate, 
          	HAWB.Status10 AS HawbStatus10, BGD.BGStatus, 
          CASE WHEN 	SubmitOFFlight.Submit IS NULL 
          THEN '' ELSE 	SubmitOFFlight.Submit END AS Submit, BGDS.PCS AS BGPCS, 
          BGDS.WGT AS BGWGT, 	HAWB.Des2, 	HAWB.Des5, '' as Regular
    FROM 	HAWB RIGHT OUTER JOIN
          	Destination LEFT OUTER JOIN
          	AirLines ON 
          	Destination.AirLine = 	AirLines.AirLineCode RIGHT OUTER JOIN
          	MAWB ON 	Destination.Code = 	MAWB.Dest2 ON 
          	HAWB.MAWB = 	MAWB.MAWB LEFT OUTER JOIN
          	OutPlans ON 
          	HAWB.WareHouseSN = 	OutPlans.WareHouseSN LEFT OUTER JOIN
          	Customer ON 	HAWB.CustID = 	Customer.CustID LEFT OUTER JOIN
              (SELECT 	Store.WareHouseSN, MAX(	Store.STPosition) AS STPosition, 
                   CASE MAX(	WareHouse.Category) 
                   WHEN 1 THEN '新' WHEN 2 THEN '老' WHEN 3 THEN '虹' WHEN 4 THEN '阪' END
                    AS Category
             FROM 	Store LEFT OUTER JOIN
                   	WareHouse ON 
                   	Store.STPosition = 	WareHouse.STPosition
             GROUP BY 	Store.WareHouseSN) Housing ON 
          	HAWB.WareHouseSN = Housing.WareHouseSN LEFT OUTER JOIN
              (SELECT DISTINCT WareHouseSN AS BGStatus
             FROM BGD) BGD ON 
          	HAWB.WareHouseSN = BGD.BGStatus LEFT OUTER JOIN
          	SubmitOFFlight ON LEFT(	MAWB.MAWB, 3) = 	SubmitOFFlight.Code AND 
          	MAWB.Flight = 	SubmitOFFlight.Flight AND 
          	MAWB.AirPort = 	SubmitOFFlight.AirPort LEFT OUTER JOIN
              (SELECT WareHouseSN, COUNT(MK) AS BGDs, MIN(Status8) AS Status8, 
                   SUM(CASE WHEN Copystatus IS NULL AND 
                   Increase = 0 THEN Pieces ELSE 0 END) AS PCS, 
                   SUM(CASE WHEN CopyStatus IS NULL THEN Weight ELSE 0 END) 
                   AS WGT
             FROM BGD
             GROUP BY WarehouseSN) BGDS ON 
          	HAWB.WareHouseSN = BGDS.WareHouseSN 
    ```



**视图旧的执行计划**

![优化前执行计划](../../../../image/oldPlan.png)

通过执行计划分析可以得知以下问题:

1: MAWB 全表扫描 13% 数据 417

```sql
--在MAWB主要的查询时间段FlightDate 添加索引

create index index_mawb_FlightDate on dbo.mawb (FlightDate);
```



2: HWAB 主键索引全表扫描 39% 数据 10万

MAWB.MAWB = HAWB.MAWB
在HWAB中,MAWB 字段没有索引,数据库执行计划判断通过主键进行全部扫描,耗费最多的时间,

```sql
 --关联条件HAWB.MAWB上加索引

 create index index_hawb_hawb on dbo.hawb (MAWB) ;

--处理时间由 26秒 到 1.457秒
```





3: BGD 索引全表扫描 11% 数据 7万 实际数据7万,全部访问
   BGD 索引全表扫描 11% 数据 7万 实际数据7万,全部访问,两次访问

 每次3.5秒左右,两次7秒;

```sql
 --添加索引

 create index index_bgd_WareHouseSN on dbo.bgd (WareHouseSN); 

 --且通过临时表改为访问一次

 --处理时间由 7秒 到 0.355秒
```



4: BGD.WarehouseSN = HAWB.WarehouseSN 7% 实际数据 417,Store 索引全表扫描 7% 实际数据 10万

```sql
--添加复合索引

create index index_store_WareHouseSN_STPosition on dbo.Store (WareHouseSN,STPosition);

--处理时间 5.83 -> 0.295
```



5: 计算列

```
在查询列中有大量计算,影响性能
由于时间关系,以及更改计算列后会影响实际业务逻辑,需要测试,我们在这里没有优化;
优化的思路是:尽量不要在数据库中做列计算,可以将数据查询出来在代码中进行计算;如果非要在数据库计算,尽量计算少的列,且在内存中计算;
```

6: 其他

```sql
不太影响效率的全表扫描
--添加复合索引SubmitOFFlight

 create index index_SubmitOFFlight_Code on dbo.SubmitOFFlight (Code,Flight,AirPort) ;
```

**添加索引后的执行计划**

![新的执行计划](../../../../image/newPlan.png)



​	通过两个执行计划的对比,我们发现之前的全表扫描已经替换成了索引扫描;整体销量也提高了10倍左右;此时较大的花费主要集中在Bookmark Lookup 标签查找上面,由于查询列过多,我们也无法做覆盖索引.如果想提高Bookmark Lookup的查询效率,猜测通过更换SSD硬盘可以提升很多;

​	6秒钟感觉还是很慢,可能sql server 2000的引擎优化不好,相似配置环境下,迁移到sql server 2008的服务器上面(两机器内存均为4G,2008机器E5 cpu 2.4Ghz*2 台式机,2000机器I3 cup 3.07Ghz 服务器,理论上2008的机器配置更低),同样情况之下2秒;如果升级到sql server 2014 ,开启内存模式,应该就可以秒查了;	

​	以上的思路是通过增加视图和升级服务器和硬件的思路.添加视图的方式最简单和成本低,基本上不需要更改任何代码;缺点是可能效率还达不到要求;升级服务器和硬件的方式也简单见效,缺点是费用较高;两者可以结合使用.现在还有第三个思路,就是把视图更改为存储过程,用临时表过滤更多的数据和减少数据库的访问次数.实际情况提升速度并不明显,由视图到存储过程,执行时间大概由6秒下降到3秒,提高100%左右;

​	以上的优化都是基于技术层面,没有牵涉到业务逻辑的优化,实际上通过业务逻辑的优化,也可以大幅增加脚本的查询速度,时间短搞不清业务逻辑的情况下不建议优化逻辑,风险比较大;



**存储过程的思路:**

```sql
--测试的时候,删除缓存
DBCC FREEPROCCACHE
GO
DBCC DROPCLEANBUFFERS
GO

DECLARE @FlightDate   datetime

SET @FlightDate   = '2015-08-01'


SELECT HAWB.WareHouseSN,
       MAWB.MAWB,
       MAWB.FlightDate,
       MAWB.Dest1,
       MAWB.Dest2,
       MAWB.Flight,
       MAWB.AirPort,
       HAWB.CustID,
       Mawb.Agent,
       Mawb.Des3,
       MAWB.Pieces1,
       MAWB.Gross1,
       MAWB.V1,
       MAWB.Weight1,
       MAWB.Pieces2,
       MAWB.Gross2,
       MAWB.V2,
       MAWB.Weight2,
       MAWB.Status16,
       MAWB.Printer,
       HAWB.V4,
       HAWB.Num3,
       Hawb.HWStatus,
       Hawb.Weight4,
       HAWB.BGNote,
       HAWB.Des4,
       HAWB.Status26,
       Hawb.DanZhengStatus,
       HAWB.Status19,
       Hawb.Status14,
       Hawb.Status9,
       Hawb.FStatus,
       Hawb.QStatus,
       HAWB.Status17,
       HAWB.Status27,
       MAWB.Status10,
       MAWB.Status11,
       MAWB.Status12,
       MAWB.Status13,
       HAWB.Dest,
       HAWB.Num1,
       HAWB.ProcedureStatus,
       HAWB.PrintOut,
       HAWB.PrintRequest,
       MAWB.Status2,
       MAWB.Status7,
       MAWB.Status8,
       MAWB.Status4,
       MAWB.Des1,
       MAWB.DeclareDate,
       HAWB.Des5,
       MAWB.OP,
       HAWB.Gross4,
       Hawb.Num10,
       HAWB.Des2,
       HAWB.HAWB
  INTO #AWB
  FROM    MAWB
       INNER JOIN
          HAWB
       ON MAWB.MAWB = HAWB.MAWB
 WHERE MAWB.FlightDate = @FlightDate --and mawb.mawb = '112-2884 6160'

--SELECT * FROM  #AWB

SELECT WareHouseSN, MK, Status8, Copystatus, Increase, Pieces, Weight
  INTO #BGD
  FROM BGD
 WHERE WareHouseSN --in ('SSHA7091914','PVG640054')
       IN (SELECT WareHouseSN
             FROM #AWB)
SELECT AWB.MAWB,
       AWB.FlightDate,
       CASE
          WHEN AWB.Agent IS NULL THEN AWB.Flight + AWB.Des3
          ELSE AWB.Flight + AWB.Des3 + '(' + AWB.Agent + ')'
       END
          AS Flight,
       AWB.Flight AS MawbFlight,
       AWB.Dest1,
       AWB.Dest2,
       AWB.WareHouseSN,
       Customer.CustName,
       AWB.Agent,
       AWB.Pieces1,
       AWB.Gross1,
       AWB.V1,
       AWB.Weight1,
       AWB.Pieces2,
       AWB.Gross2,
       AWB.V2,
       AWB.Weight2,
       CASE WHEN AWB.Status16 = 1 THEN '自报关' ELSE '' END + AWB.OP AS OP,
       CASE AWB.Printer WHEN '' THEN '' ELSE '总' END AS Printed,
       AWB.Pieces2 AS PCS,
       AWB.Gross4 AS WGT,
       AWB.V4 AS VLM,
       CASE AWB.Num3 WHEN 0 THEN '' ELSE '*' END AS Broken,
       CASE AWB.HWStatus
          WHEN 0
          THEN
             ltrim (str (AWB.Pieces1))
          WHEN 2
          THEN
             ltrim (str (AWB.Pieces1)) + '(' + ltrim (str (AWB.Pieces2)) + ')'
          ELSE
             ltrim (str (AWB.Pieces2))
       END
          AS Pieces,
       CASE AWB.HWStatus WHEN 0 THEN AWB.Gross1 ELSE AWB.Gross4 END AS Gross,
       CASE AWB.HWStatus WHEN 0 THEN AWB.V1 ELSE AWB.V4 END AS V,
       CASE AWB.HWStatus WHEN 0 THEN AWB.Weight1 ELSE AWB.Weight4 END
          AS Weight,
       AWB.HAWB,
       AWB.BGNote,
       CASE AWB.Status11 WHEN 0 THEN '' ELSE '*' END AS DCZ,
       CASE
          WHEN BGDS.BGDs IS NULL THEN ''
          ELSE CASE BGDS.Status8 WHEN 0 THEN '*' ELSE '**' END
       END
       + AWB.Des4
       + CASE AWB.Num10 WHEN 0 THEN '' ELSE '扣' END
       + CASE AWB.HWStatus
            WHEN 0 THEN ' '
            WHEN 1 THEN 'H'
            WHEN 2 THEN 'B'
            WHEN 3 THEN 'C'
         END
       + CASE AWB.Num3 WHEN 0 THEN '' ELSE '破' END
       + CASE AWB.Status26 WHEN 2 THEN 'T' ELSE '' END
       + CASE AWB.DanZhengStatus
            WHEN 0 THEN ' '
            ELSE CASE AWB.Status19 WHEN 0 THEN 'D' ELSE 'M' END
         END
       + CASE AWB.Status14
            WHEN 0 THEN '  '
            WHEN 1 THEN '到'
            WHEN 2 THEN '签'
            WHEN 3 THEN '拆'
            WHEN 4 THEN '问'
            WHEN 5 THEN '放'
            WHEN 6 THEN '审'
            WHEN 7 THEN '计'
            WHEN 8 THEN '完'
         END
       + CASE AWB.Status9 WHEN 0 THEN ' ' ELSE 'S' END
       + CASE AWB.FStatus WHEN 0 THEN ' ' ELSE 'A' END
       + CASE AWB.QStatus WHEN 0 THEN ' ' ELSE 'P' END
       + CASE AWB.Status16 WHEN 0 THEN '  ' ELSE '标' END
       + CASE WHEN Housing.Category IS NULL THEN '' ELSE Housing.Category END
       + CASE AWB.Status13
            WHEN 1 THEN '区内'
            WHEN 2 THEN '区外'
            ELSE ''
         END
       + CASE WHEN outPlans.WareHouseSN IS NULL THEN '' ELSE '排' END
          AS Status,
       AWB.Status14,
       AWB.Status17,
       AWB.Status27,
       Housing.STPosition,
       Housing.Category,
       AWB.Status10,
       AWB.Status11,
       AWB.Status12,
       AWB.Status13,
       Destination.City,
       AirLines.AirLine,
       AWB.AirPort,
       AWB.Dest,
       AWB.Num1,
       AWB.ProcedureStatus,
       AWB.PrintOut,
       AWB.PrintRequest,
       AWB.Status2,
       AWB.Status7,
       AWB.Status8,
       AWB.Status4,
       AWB.Des1,
       CASE
          WHEN AWB.DeclareDate IS NULL OR AWB.DeclareDate = AWB.FlightDate
          THEN
             ''
          ELSE
             '报关日'
             + RIGHT (CONVERT (VARCHAR (10), AWB.DeclareDate, 120), 5)
       END
          AS DeclareDate,
       AWB.Status10 AS HawbStatus10,
       BGD.BGStatus,
       CASE
          WHEN SubmitOFFlight.Submit IS NULL THEN ''
          ELSE SubmitOFFlight.Submit
       END
          AS Submit,
       BGDS.PCS AS BGPCS,
       BGDS.WGT AS BGWGT,
       AWB.Des2,
       AWB.Des5,
       '' AS Regular
  FROM                      #AWB AWB
                         RIGHT OUTER JOIN
                               dbo.Destination
                            LEFT OUTER JOIN
                               dbo.AirLines
                            ON dbo.Destination.AirLine =
                                  dbo.AirLines.AirLineCode
                         ON dbo.Destination.Code = AWB.Dest2
                      LEFT OUTER JOIN
                         dbo.OutPlans
                      ON AWB.WareHouseSN = dbo.OutPlans.WareHouseSN
                   LEFT OUTER JOIN
                      dbo.Customer
                   ON AWB.CustID = dbo.Customer.CustID
                LEFT OUTER JOIN
                   (SELECT dbo.Store.WareHouseSN,
                           MAX (dbo.Store.STPosition) AS STPosition,
                           CASE MAX (dbo.WareHouse.Category)
                              WHEN 1 THEN '新'
                              WHEN 2 THEN '老'
                              WHEN 3 THEN '虹'
                              WHEN 4 THEN '阪'
                           END
                              AS Category
                      FROM    dbo.Store
                           LEFT OUTER JOIN
                              dbo.WareHouse
                           ON dbo.Store.STPosition = dbo.WareHouse.STPosition
                     WHERE dbo.Store.WareHouseSN IN (SELECT WareHouseSN
                                                       FROM #AWB)
                    GROUP BY dbo.Store.WareHouseSN) Housing
                ON AWB.WareHouseSN = Housing.WareHouseSN
             LEFT OUTER JOIN
                (SELECT DISTINCT WareHouseSN AS BGStatus
                   FROM #BGD) BGD
             ON AWB.WareHouseSN = BGD.BGStatus
          LEFT OUTER JOIN
             dbo.SubmitOFFlight
          ON     LEFT (AWB.MAWB, 3) = dbo.SubmitOFFlight.Code
             AND AWB.Flight = dbo.SubmitOFFlight.Flight
             AND AWB.AirPort = dbo.SubmitOFFlight.AirPort
       LEFT OUTER JOIN
          (SELECT WareHouseSN,
                  COUNT (MK) AS BGDs,
                  MIN (Status8) AS Status8,
                  SUM(CASE
                         WHEN Copystatus IS NULL AND Increase = 0 THEN Pieces
                         ELSE 0
                      END)
                     AS PCS,
                  SUM (CASE WHEN CopyStatus IS NULL THEN Weight ELSE 0 END)
                     AS WGT
             FROM #BGD
           GROUP BY WarehouseSN) BGDS
       ON AWB.WareHouseSN = BGDS.WareHouseSN
 WHERE AWB.FlightDate = @FlightDate 

DROP TABLE  #AWB
DROP TABLE #BGD
```

