---
title: Mac 环境下 react-native4.9 android studio3.0 开发实录
date: 2017-10-18 18:25:18
tags: [react native,android studio]
---

​	心情一激动,使用了最新版的android studio 和react native的最新版本开发程序;一地坑,记录下遇到问题的点点滴滴;

​	个人比较倾向于用次新版的东西,最新版的东西坑太多,出错了有可能找不到解决方案.用太旧的东西,可能会有些性能问题,后期升级更比较麻烦;

<!--more-->

**问题::Error:(1, 1) A problem occurred evaluating project ':app'.**

> java.lang.UnsupportedClassVersionError: com/android/build/gradle/AppPlugin : Unsupported major.minor version 52.0

**解决方法**

​	这个问题是jdk版本过低导致,我电脑上装了1.7 和1.8 两个版本,android studio 指定的是1.7.本来以为 java -version 是1.8 就是使用了默认的1.8,实际上需要重新指定;

​	指定方法,打开 file -> Project Structure ->sdk location ,jdk location ,更改为1.8 的版本;

​       注意,这个设置jdk的地方不在setting里面,刚开始在setting里面找半天也没找到;

**问题2:**

```java
Error:Failed to complete Gradle execution.

Cause:

The version of Gradle you are using (3.3) does not support the forTasks() method on BuildActionExecuter. Support for this is available in Gradle 3.5 and all later versions.

```



解决方法:

从[Stack Overflow](https://stackoverflow.com/questions/44206433/gradle-version-3-3-does-not-support-fortask-method-on-buildactionexecuter)找到了答案:

将”gradle-wrapper.properties”中的：



> distributionUrl=https\://services.gradle.org/distributions/gradle-3.3-all.zip

更改为：

> distributionUrl=https\://services.gradle.org/distributions/gradle-4.0-milestone-1-all.zip