---
title: 数据库技术交流--函数篇
date: 2007-08-14 14:03:36
tags: [数据库]
---

数据库技术交流--函数篇:
   在我们平时的数据开发时，像其他的程序开发一样，也经常会遇到很多共通问题，怎么解决这些共通问题呢，我在这里向大家推荐一种方法。
   1：函数（function）如果开发里面有很多共通部分，这时候你会发现用函数就是很方便了，例如：我们程序里面经常要用到除法等问题，我们知道当分母为零时会出异常，在程序里判断当然也可以，但是要考虑程序的可扩展性和可维护性，最好的方法就是我们再自定义一个函数用来判断除数为零时的情况，
下面就是我们创建的除法函数：

<!--more-->

``` sql
-- =============================================

-- Author:        <zhangzeshuai>

-- Create date: <2007/08/10>

-- Description:    <除数为零时判断>

-- =============================================

CREATE FUNCTION [dbo].[fn_zone]

(

     @numerator FLOAT,    --分子

     @denominator FLOAT   --分母

)

RETURNS NUMERIC(13,2)

AS

BEGIN

    DECLARE @Result NUMERIC(13,2)  --结果

    IF(@denominator=0 OR @denominator IS NULL)  --分母为零或空时

     BEGIN

                 SET @Result = 0   --返回结果，也可设置为其他

     END

        ELSE

         BEGIN

        SET @Result = @numerator/@denominator  --返回结果

         END

    RETURN @Result

END

GO



```



使用时直接用 SELECT [dbo].[fn_zone](4,0) 就可以出结果了，而且更改时只需我们更改函数，而不是去每个sql文里去找，测试时也只是测试该函数就可以了。比每个程序里判断简单多了，最重要的是易于维护和扩展。
​                                									Kevin
​                                									2007/08/14
