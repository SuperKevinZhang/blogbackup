---
title: 客户现场数据库优化--PKP效率优化
date: 2013-10-23 19:06:31
tags: 数据库
---


​     今天在客户现场待了一天优化PKP问题.解决了部分问题.sp_importpkp的执行效率从20"-120"提高到了6"-15"之间,平均应该在10"以内,基本能满足客户的10"以内的要求.但是至于操作还会不会卡,还有待于观察,我估计操作卡,不是PKP造成的,到底是不是,需要一段时间观察.

<!--more-->

​     过几天我们会继续观察日志,以及了解客户操作速度情况.

​     解决过程如下:(以后大家也按照这个流程去解决效率问题)
​     1:找到执行的job,查看历史,了解存储过程执行情况,1-3分钟能够执行完毕
​     2:找到存储过程,查看每个阶段的日志,sql见附1
​          发现问题:
​          2.1 PKP insert 耗时1"-2"
​          2.2 CWB insert 耗时 2"-10"
​          2.3 TB_PREPRINT update 耗时2"左右
​          2.4 TT_upload_collection_update 耗时 30"-60"
​          2.5 其他耗时较少的不关注
​     3:分析耗时原因,2.1 2.2 ,是由于基础表大导致,2.2,2.3可以修改,估计效果也会太明显,前三个表的数量级都比较大,而且索引较多,所以慢可以理解.但是2.4 70多万条的记录,不应该这么慢.分析sql(附2 注释部分),发现在update使用到子查询的时候,会导致全表扫描,消耗cpu是不使用子查询的几千倍. 至于为什么update 条件里面使用子查询会导致全表扫描没用索引,我也没搞懂,也没查到愿意.只是从执行计划里面看出来了.

​     4:更改 将update使用的子查询更换为只读,只next游标. 耗时0/2"-0.9"之间

​     5:测试,ipmort_pkp 大概在6秒左右,由于不是业务高峰期,还不知道那时多少,从以前的日志中可以推断出,应该在15"以内.

​     这个时候解决了部分问题.还需要做的是
​     A:继续监控几天日志,追踪客户操情况
​     B:检查存储过程是否有类似操作,在我的检查中,发现有几处类似的操作,见附3
​     C:确认监控日志结果,如果情况得到改善,把B处问题更改掉.如果情况还继续卡,说明另有他因,需要重新系统的排查.

今天改了一个存储过程, sp_importpkp, dos的存储过程也存在这个问题.今天监控一晚上后,卫峰明天把DOS库[dbo].[proc_HHTCLTBATCH]也改掉,具体该的地方和参加,见附2和附3.
如果有改善,那么周五会有一个大改善.


job:new DOS clt batch, import pkp (包含两个存储过程[dbo].[proc_HHTCLTBATCH], sp_importpkp)

附1:

```sql
SELECT

               --avg(datediff (ss, a.batch_on, b.batch_on))

            a.batch_on as 'begindate',b.batch_on as 'enddate',

            datediff (ss, a.batch_on, b.batch_on) as 'diff'

  FROM    (SELECT *

             FROM dbo.LG_PROCESS

            WHERE     batch_type = 'ic'

                  AND batch_on > '2013-10-18'

                  AND process_message = 'TT_UPLOAD_COLLECTION update') A

       INNER JOIN

          (SELECT *

             FROM dbo.LG_PROCESS

            WHERE     batch_type = 'ic'

                  AND batch_on > '2013-10-18'

                  AND process_message = 'µ¼ÈëPKP¼ÇÂ¼½áÊø') B

       ON a.batch_id = b.batch_id

               -- group by   a.batch_on

               order by a.batch_on

```




