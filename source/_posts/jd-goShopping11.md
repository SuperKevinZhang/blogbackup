---
title: 京东抢购秒杀助手—小桃子版
date: 2017-11-03 14:35:29
tags: [秒杀,抢购,京东]
---

​	双十一当天考试,没办法去抢购东西,闲暇之余写了一个京东秒杀抢购助手,帮助没有时间的人抢一些想要的东西;

​	为什么叫小桃子版,因为头像的小公主就是叫小桃子.@_@本来想做把脚本做成chrome扩展插件的形式,让大家用起来方便些,扩展程序注入监控画面比较麻烦,试了两次失败了,平时比较忙,来不及实现了.使用脚本抢购,就当锻炼锻炼自己的动手能力吧;

​	如果做成Chrome插件,就是这个样子

![如果做成chrome就是这个样子](http://img.blog.csdn.net/20171103152352169?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvemhhbmd6ZXNodWFp/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)



​	**使用方式:**

​	1:打开要抢购商品的详细画面,请注意网络地址一定要是 item.jd.com开头的详细画面,摁F12,打开开发者模式,选择Console选项卡,把以下代码粘贴在Console里面,回车执行;

<!--more-->

​	*打开开发者模式*

​	![这里写图片描述](http://img.blog.csdn.net/20171103145829718?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvemhhbmd6ZXNodWFp/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)



​	**特别注意:被监控的画面标签页一定要在前台 **
![这里写图片描述](http://img.blog.csdn.net/20171103145848977?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvemhhbmd6ZXNodWFp/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)





代码:

``` javascript
var nIntervId;
var count = 1;
var goDate;
function go() {
    console.log("小桃子 ^_^ 正在帮你抢购＊＊＊＊＊＊＊＊＊＊＊＊＊ 刷新" + count + "次");
    //console.log("host:" + window.location.hostname);
    count++;
    // iPhone X go set time
    if (Date.now() >= new Date("2017-11-03 15:59:59")) {
        console.log("开始抢购iPhone X" + Date.now());

        // iphoneX 抢购
        if ($(parent.frames[0].document).find("#choose-btn-ko").length == 1) {
            console.log("(++++++++++++iphoneX 抢购");
            var sku = window.location.pathname.replace(/[^0-9]/ig, "");
            var ref = "//cart.jd.com/gate.action?pid=" + sku + "&pcount=1&ptype=1";
            console.log("https:" + ref);
            //5089237
            $(parent.frames[0].document).find("#choose-btn-ko").attr("href", ref);//                 
            parent.frames[0].document.getElementById("choose-btn-ko").click();
            return;
        }

        //预约抢购
        if ($(parent.frames[0].document).find("#btn-reservation").length == 1) {
            console.log("(++++++++++++正在预约抢购");

            parent.frames[0].document.getElementById("btn-reservation").click();
            return;
        }

        //秒杀   
        if ($(parent.frames[0].document).find("#InitCartUrl").length == 1) {
            console.log("(++++++++++++正在秒杀");
            parent.frames[0].document.getElementById("InitCartUrl").click();
            return;
        }

        //去购物车结算
        if ($(parent.frames[0].document).find("#GotoShoppingCart").length == 1) {
            console.log("(++++++++++++正在去购物车结算");
            parent.frames[0].document.getElementById("GotoShoppingCart").click();
        }

        //去结算        
        if ($(parent.frames[0].document).find(".submit-btn").length == 1) {
            console.log("(++++++++++++正在去结算");

            //只提交我抢购的商品
            //var sku = window.location.pathname.replace(/[^0-9]/ig, "");                 

            //$("#toggle-checkboxes_up").trigger("click");
            //全不选择
            //parent.frames[0].document.getElementById("toggle-checkboxes_up").click();

            //$(parent.frames[0].document).find('input:checkbox').attr("checked",false);
            //$(parent.frames[0].document).find("input:checkbox[value^='"+sku+"']").trigger("click");

            //$(parent.frames[0].document).find("input:checkbox[value^='"+sku+"']").attr("checked",true);

            parent.frames[0].document.getElementsByClassName("submit-btn")[0].click();
        }
        //提交订单order-submit
        if ($(parent.frames[0].document).find("#order-submit").length == 1) {
            console.log("(++++++++++++正在提交订单");
            //$(parent.frames[0].document).find(".payment-item item-selected online-payment")

            //在线支付
            parent.frames[0].document.getElementById("order-submit").click();
        }
    }
}
function rewrite(current) {
    fr4me = '<frameset cols=\'*\'>\n<frame src=\'' + current + '\'/>';
    fr4me += '</frameset>';
    with (document) { write(fr4me); void (close()) };
}


//注入sql
rewrite(window.location.href);

//这里需要注意的是，prompt有两个参数，前面是提示的话，后面是当对话框出来后，在对话框里的默认值
var d = prompt("请输入抢购开始时间", "2017-11-03 15:59:59");
//如果返回的有内容
if (d) {
    try {
        goDate = new Date(d);
        console.log("设定时间成功:" + goDate);
        
        alert("监控期间,请保持标签页在最前面");
        //go(); 0.25秒执行一次
        nIntervId = setInterval("go()", 250);
    }
    catch (e) {
        alert("时间格式不正确,请使用yyyy-MM-dd hh:mm:ss格式,精确到秒, 请重试");
    }
}
else {
    alert("请抢购时间, 请重重试");

}


/*
    clearInterval(nIntervId);//停止监控
    */

```



