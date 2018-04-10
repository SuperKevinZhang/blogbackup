---
title: webapi优化之MiniProfiler监控 (一)
date: 2017-10-27 19:07:23
tags: [WebApi,MiniProfiler,Entity Framework]
---

​	使用.net开发Web API,Entity Framework是个好东西,可以大大简化操作数据库的复杂度,使很多不会写sql的人也能完成对后台的操作.但凡事都有两面性,EF 带来开发便利性的同时,也带来了效率的低下性; 这里所说的效率低下,并不是说EF真的效率低下,而是由于大家对EF不熟悉,EF功能又非常强大,随便你怎么写都能出现结果,用户用法不规范导致的效率低下;除了开发规范避免一些特别明显的效率写法外,好的调试工具这个时候就成本效率优化必需的东西了,这个时候MiniProfiler插件就排上了用场;

<!--more-->

​	MiniProfiler不是必需的,你也可以通过使用SQL Server Profiler来监控执行的sql,但确实非常必要的,MiniProfiler可以使你的监控变得非常简单; MiniProfiler是[Stack Overflow](http://stackoverflow.com/)团队设计的一款对ASP.NET MVC的性能分析的小程序。可以对一个页面本身，及该页面通过直接引用、Ajax、Iframe形式访问的其它页面进行监控,监控内容包括数据库内容，并可以显示数据库访问的SQL（支持EF、EF CodeFirst等 ）。并且以很友好的方式展现在页面上。

​	该Profiler的一个特别有用的功能是它与数据库框架的集成。除了.NET原生的 DbConnection类，profiler还内置了对实体框架（Entity Framework）以及LINQ to SQL的支持。任何执行的Step都会包括当时查询的次数和所花费的时间。为了检测常见的错误，如N+1反模式，profiler将检测仅有参数值存在差 异的多个查询。



	## MVC Web的监控

​	MiniProfiler 放在nuget上面,安装起来也非常简单,可以通过程序包管理器控制台安装,也可以通过管理解决方案的NuGet程序包安装(两者 都在NuGet包管理器里面),我们这里通过命令行的形式安装;

​	安装准备:

​		1:Entity Framework 6.0

​		2:Visio Studio 2013 

​		3:[MiniProfiler](https://miniprofiler.com/) 3.2.0.157

​	安装步骤

​		1:打开工具->NuGet包管理器->程序包管理器控制台

​			输入

``` sh
PM> Install-Package MiniProfiler
```

​		如果是低版本的EF需要选择对应的Profiler,高版本的建议选用最新版的Profiler,我刚开始选的稍微低一个版本的,监控输入不出来;

​		2:接下来的就是需要配置了;

 			2.1 在Global.asax.cs中配置启动监控信息(这里配置的监控EF6)

``` c#
protected void Application_Start()
        {
            AreaRegistration.RegisterAllAreas();
            GlobalConfiguration.Configure(WebApiConfig.Register);
            FilterConfig.RegisterGlobalFilters(GlobalFilters.Filters);
            RouteConfig.RegisterRoutes(RouteTable.Routes);
            BundleConfig.RegisterBundles(BundleTable.Bundles);
		   //这里是监控EF6 的信息 --监控EF6 初始化
            MiniProfilerEF6.Initialize();

        }

        protected void Application_BeginRequest()
        {
            if (Request.IsLocal)//这里是允许本地访问启动监控,可不写
            {
                //开始监控
                MiniProfiler.Start();

            }
        }

        protected void Application_EndRequest()
        {
            //结束监控
            MiniProfiler.Stop();
        }
```



​		2.2 修改layout文件

​			在Views\Shared\_Layout.cshtml文件的body前面加上一段代码，让监控展示在页面上。

``` html
<head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width" />
    <title>@ViewBag.Title</title>
    @RenderSection("scripts", required: false)
    @using StackExchange.Profiling; /
</head>
<body>
    @RenderBody()
    @MiniProfiler.RenderIncludes();
</body>
```

​	此时再次运行,就可以在左上角看到web画面执行的情况了;

​	好

## WebAPI的监控

​	默认情况下,Profiler并不会监控webapi,如果我们想监控webapi,需要做一些其他的更改;

在[miniProfiler](https://community.miniprofiler.com/t/can-i-use-mini-profiler-for-asp-net-web-api-and-have-results-still-seen-on-url/365/2)论坛上搜索到一个解决方案:需要的请参考



​	https://github.com/mmusket/webapi-profiling

​	http://www.lambdatwist.com/webapi-profiling-with-miniprofiler-swagger/