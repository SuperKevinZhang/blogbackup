---
title: Android 自动化测试脚本初探
date: 2018-01-31 17:32:24
tags: shell,Python
---

​	最近做一个仓库APP的项目,其中一些测试的体力活,数据造起来非常麻烦,需要造条码,需要疯狂扫描测试效率,以及永远都无法避免的回归测试.烦恼之余,想想为什么不使用自动化脚本来解脱轮回呢.

<!--more-->

​	自动化工具网上已经很多,已经有无数个轮子,但使用起来有些麻烦.我想要测试的只是最简单的模拟录入.不想那么的去调查和学习自动化工具等比较重的东西.就想到了通过脚本模拟一些录入.最初的困难来自仓库收货扫描的时候需要采集SN,我造了400个SN条码,扫了100次,已经开始扫的头晕脑胀了,这些简单重复的工作在我眼中不应该由人去做的,于是开始想办法如何自动化模拟输入;

​	由于知识狭隘,最初使用的是 adb shell命令,这个命令因为开发android的时候经常用到,所以第一个想到了它,模拟400SN,扫描录入并确定上架.本来枯燥的输入工作,瞬间变的简单,命令行执行起脚本.去趟厕所回来,结果执行的差不多了.再次回归测试,只需要改几个数据即可.是不是很方便.

```shell
# 订单 9504 sku 10000001  循环400次 
intCount=900100 
stringDefalut='286172018012'
adb shell input text 000000009504
adb shell input keyevent 66
sleep 1
for ((i=1;i<=400;i++))
do
    adb shell input text ${stringDefalut}$[${intCount}+${i}]
    # 66 回车
    adb shell input keyevent 66
    sleep 1
    # 最下面的button
    adb shell input tap 240 750
    #seleep 1
done
```

​	脚本发给了同事,发现shell命令在windows平台下执行效果不好.不兼容.花几分钟时间瞬间改成Python版本,兼容各种平台了;

```python
import time
import os

def execute(cmd):
    os.system(cmd)
def text(content):
    os.system('adb shell input text ' + content)
def enter():
    os.system('adb shell input keyevent 66')
    time.sleep(1)
# 订单 9504 sku 10000001  循环400次
intCount = 900100
stringDefault = '286172018012'
# inputCmd = 'adb shell input text 000000009504'
# text('000000009504')
# enter()
for i in range(50, 51):
    intCount += i
    text(stringDefault + str(intCount))
    enter()
    execute('adb shell input tap 240 750')

```

​	当然以上的脚本只是最简单的脚本,模拟录入,没有和UI交互.如果想更深入的自动化测试,这个时候就需要使用各种自动化测试工具了.

​	adb shell 和 python真是个好东西,万能胶水语言python,我要好好学学了;