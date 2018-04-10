---
title: HttpRuntime.Cache 与 static Dictionary 缓存
date: 2017-11-28 17:27:52
tags: [缓存,Cache]
---

​	HttpRuntime.Cache 与 static Dictionary缓存用法

​	***简介:***

​	本文设计系统采用C#语言,webapi技术,数据库连接使用EntityFramework6.0技术.数据库采用mysql,业务领域为仓储领域;使用到的缓存技术有ORM缓存(EntityFramework自带),HttpRuntime.Cache缓存,static Dictiona缓存.内容为开放式讨论,本文介绍的缓存技术不一定是最合适的技术,笔者对于缓存技术也处于摸索状态,后续会根据实际情况调整缓存解决方案;

<!--more-->

​	***问题现状:***

​	仓库出入库系统中,扫描过程中会有大量的数据库操作,对于一个全国具有2000万平的仓库和一个集中式部署的仓库系统,如果提高出入库的业务吞吐量是一个必需解决的问题.客户要求可以同时支持5000人同时操作,且扫描处理不超过1秒钟的系统要求.虽然我不认为会有一家公司能有5000个人同时去进行库存操作.但是对于技术要求,还是需要达到的;

​	鉴于仓库系统操作的特殊性,我们采用分布式部署api的形式,因为不存在一个仓库去操作另外一个仓库内的数据,所以不存在分布式一致性问题,那么这样的分布式和缓存方案就比较简单了;我只需要指定华东区仓库去访问华东区固定的api地址,华南区仓库访问华南区固定api地址,这些api访问一个中心库即可;

​	这样设计的好处是没有复杂的分布式事物处理,没有数据同步的问题,也不存在分布式一致性问题,缺点是所有的压力都集中到中心数据库上了;所以优化的核心工作在中心库这里;

​	***解决方案***

​	解决方案当然是减少数据库的访问量,加快数据写入速度;当然这些都是废话;处理这些问题,我的一般思路是首先使用现成的东西,既然别人有了轮子,所以我也就不需要自己造了.

​	第一个想到的解决方案当然是**EntityFramework的ORM缓存技术**了,最简单,配置一下就行了.当然如果这个技术能解决所有的问题,也就没有下面的文章了;ORM缓存虽然简单,但是数据何时更新,何时使用cache,何时使用数据的情况不好解决(<u>也许只是我不知道怎么解决</u>),查询等不常用的cache,或者数据更新后不太需要及时通知app的数据,我使用ORM缓存技术;但当订单状态发生改变了,接口不知道变化的结果.就引出了第二个问题;

​	第二个想到的解决方案是**HttpRuntime.Cache**,这个缓存网上的介绍很多,是.net里面自带的一个缓存类,也是基于key,value形式的,可以存放一些web中的一些不太复杂的缓存,如果不熟悉的话可以看[这里](https://www.cnblogs.com/fish-li/archive/2011/12/27/2304063.html).HttpRuntime.Cache存放的是订单实体缓存,CacheHelper.Set()形式,当我的订单等内容发送变化是,其他接口通过调用Remove接口清除缓存,下次访问的时候也就是最新的缓存了;这里大家使用的时候请注意缓存穿透和缓存雪崩;这种情况也是存储不经常发生变化的东西,但是扫描的时候每次扫描的数量都会发生变化,如何处理变化的数据,又会引出第三个问题;

​	第三个解决方案是通过**静态Dictionary**方式来解决每次访问都会变化的问题;这个解决方法有些妖.后面我们说为什么妖.我们先说我们的静态缓存;静态变量作为缓存的效率是最高的,对于一个每次访问都要改变的缓存,当然直接使用Dictionary作为缓存也是可以的,但是对于一个web系统,直接使用静态变量会引发无数个不可预知的问题,所以我们必需封装一个特殊的方法来处理这个问题.自定义缓存需要考虑这两大问题,线程安全,缓存过期,其他问题与HttpRuntime.Cache用法就一致了;对于线程安全使用了ConcurrentDictionary容器来保证线程安全,并在操作缓存的时候,锁住当前的key,缓存过期策略使用**LRU策略(Least recently used，最近最少使用)**,通过解决这两个问题来解决静态变量的缺点,顺便还加了一个缓存容量,防止无限增长;

​	为什么妖,因为每次扫描的时候,计算扫描的数量并不是每次都从数据库读取,而是在缓存里面取到后add扫描的数量,然后再把这个数量缓存到字典里面.这里面虽然考虑到了数据库事物,线程并发等问题,但项目还没经过实践.不知道会不会出现数据不准问题;

```c#
		/// <summary>
        /// 获取订单的扫描数量
        /// </summary>
        /// <param name="orderCode"></param>
        /// <returns></returns>
        public long getOrderScanCount(string orderCode, long ScanCount = 0)
        {
            string qtyKey = "ScanQty" + orderCode;
            //先读取
            var qty = cacheScanQty.Get(qtyKey);
            if (qty != null && qty >0 )//cache 存在
            {
                lock (this)
                {
                    qty = qty + ScanCount ;
                    cacheScanQty.Set(qtyKey, qty);
                    Console.WriteLine(String.Format("读Cache"));

                    return qty;
                }
            }
            else
            {
                Console.WriteLine(String.Format("读数据库"));

                //订单总体已扫描数量/件
                //long count = db.tb_inbound_scan_detail.Where(s => s.OrderNo == orderCode && s.DeleteFlag == false && s.UomCode == "PCS").
                //                                        AsNoTracking().
                //                                        Sum(s => s.Qty);

                long count = 1;
                cacheScanQty.Set(qtyKey, count);//缓存LRU
                //cache.Get(QtyCache);
                return count;
            }
        }
```





​	**HttpRuntime.Cache 缓存帮助类**

```c#
	/// <summary>
    /// CacheHelper HttpRuntime.Cache
    /// </summary>
    public static class CacheHelper
    {
        #region 缓存Helper
        /// <summary>
        /// 获取数据缓存
        /// </summary>
        /// <param name="CacheKey">键</param>
        public static object Get(string CacheKey)
        {
            System.Web.Caching.Cache objCache = HttpRuntime.Cache;
            return objCache[CacheKey];
        }
        /// <summary>
        /// 设置数据缓存(慎用)
        /// </summary>
        /// <param name="CacheKey">Key</param>
        /// <param name="objObject">Value</param>
        public static void Set(string CacheKey, object objObject)
        {
            System.Web.Caching.Cache objCache = HttpRuntime.Cache;
            objCache.Insert(CacheKey, objObject);
        }
       /// <summary>
        /// 设置数据缓存
       /// </summary>
       /// <param name="CacheKey">Key</param>
       /// <param name="objObject">Value</param>
       /// <param name="Timeout">时间跨度</param>
        public static void Set(string CacheKey, object objObject, TimeSpan Timeout)
        {
            System.Web.Caching.Cache objCache = HttpRuntime.Cache;
            objCache.Insert(CacheKey, objObject, null, DateTime.MaxValue, Timeout, System.Web.Caching.CacheItemPriority.NotRemovable, null);
        }
        /// <summary>
        /// 设置数据缓存
        /// </summary>
        /// <param name="CacheKey">Key</param>
        /// <param name="objObject">Value</param>
        /// <param name="absoluteExpiration">绝对过期时间</param>
        /// <param name="slidingExpiration">时间跨度</param>
        public static void Set(string CacheKey, object objObject, DateTime absoluteExpiration, TimeSpan slidingExpiration)
        {
            System.Web.Caching.Cache objCache = HttpRuntime.Cache;
            objCache.Insert(CacheKey, objObject, null, absoluteExpiration, slidingExpiration);
        }
        /// <summary>
        /// 移除指定数据缓存
        /// </summary>
        /// <param name="CacheKey">Key</param>
        public static void RemoveAllCache(string CacheKey)
        {
            System.Web.Caching.Cache _cache = HttpRuntime.Cache;
            _cache.Remove(CacheKey);
        }
        /// <summary>
        /// 移除全部缓存
        /// </summary>
        public static void RemoveAllCache()
        {
            System.Web.Caching.Cache _cache = HttpRuntime.Cache;
            IDictionaryEnumerator CacheEnum = _cache.GetEnumerator();
            while (CacheEnum.MoveNext())
            {
                _cache.Remove(CacheEnum.Key.ToString());
            }
        }
        #endregion
    }
```

**基于RLU算法的static Dictiona**

```c#
/// <summary>
    /// LRU缓存 Static缓存
    /// </summary>
    /// <typeparam name="TValue"></typeparam>
    public class LRUCache<TValue> : IEnumerable<KeyValuePair<string, TValue>>
    {
        private long ageToDiscard = 0;  //淘汰的年龄起点
        private long currentAge = 0;        //当前缓存最新年龄
        private int maxSize = 0;          //缓存最大容量

        //使用ConcurrentDictionary来作为我们的缓存容器，并能保证线程安全
        private readonly ConcurrentDictionary<string, TrackValue> cache;

        private TimeSpan maxTime;

        //实现IEnumerable方法一
        public System.Collections.Generic.IEnumerator<KeyValuePair<string, TValue>> GetEnumerator()
        {
            return this.GetEnumerator();
        }
        //实现IEnumerable方法二
        System.Collections.IEnumerator System.Collections.IEnumerable.GetEnumerator()
        {
            return GetEnumerator();//IEnumerator<T>继承自IEnumerator
        }

        /// <summary>
        /// 初始化
        /// </summary>
        /// <param name="maxKeySize">缓存最大size</param>
        public LRUCache(int maxKeySize)
        {
            cache = new ConcurrentDictionary<string, TrackValue>();
            maxSize = maxKeySize;
        }
        /// <summary>
        /// 初始化
        /// </summary>
        /// <param name="maxKeySize">缓存最大size</param>
        /// <param name="maxExpireTime">时间跨度</param>
        public LRUCache(int maxKeySize, TimeSpan maxExpireTime)
        {
            cache = new ConcurrentDictionary<string, TrackValue>();
            maxSize = maxKeySize;
            maxTime = maxExpireTime;
        }

        /// <summary>
        /// 新增缓存
        /// </summary>
        /// <param name="key">key</param>
        /// <param name="value">value</param>
        public void Set(string key, TValue value)
        {
            Adjust(key);
            var result = new TrackValue(this, value);
            cache.AddOrUpdate(key, result, (k, o) => result);
        }
        /// <summary>
        /// 跟踪值
        /// </summary>
        public class TrackValue
        {
            public readonly TValue Value;
            public long Age;

            //TrackValue增加创建时间和过期时间
            public readonly DateTime CreateTime;
            public readonly TimeSpan ExpireTime;

            public TrackValue(LRUCache<TValue> lv, TValue tv)
            {
                Age = Interlocked.Increment(ref lv.currentAge);
                Value = tv;
            }
        }
        /// <summary>
        /// 调整
        /// </summary>
        /// <param name="key"></param>
        public void Adjust(string key)
        {
            while (cache.Count >= maxSize)
            {
                long ageToDelete = Interlocked.Increment(ref ageToDiscard);
                var toDiscard =
                      cache.FirstOrDefault(p => p.Value.Age == ageToDelete);
                if (toDiscard.Key == null)
                    continue;
                TrackValue old;
                cache.TryRemove(toDiscard.Key, out old);
            }
        }

        public TValue Get(string key)
        {
            try
            {
                TrackValue value = null;
                if (cache.TryGetValue(key, out value))
                {
                    value.Age = Interlocked.Increment(ref currentAge);
                }
                return value.Value;
            }
            catch (Exception ex)
            {

                return default(TValue);
            }
           
        }
        /*****************************************************/
        public Tuple<TrackValue, bool> CheckExpire(string key)
        {
            TrackValue result;
            if (cache.TryGetValue(key, out result))
            {
                var age = DateTime.Now.Subtract(result.CreateTime);
                if (age >= maxTime || age >= result.ExpireTime)
                {
                    TrackValue old;
                    cache.TryRemove(key, out old);
                    return Tuple.Create(default(TrackValue), false);
                }
            }
            return Tuple.Create(result, true);
        }
        /// <summary>
        /// 检查缓存是否过期,过期则删除 需要通过定时任务或者线程调用
        /// </summary>
        public void Inspection()
        {
            foreach (var item in this)
            {
                CheckExpire(item.Key);
            }
        }
    }
```



参考:

​	[知乎连接](https://www.zhihu.com/question/37747712)