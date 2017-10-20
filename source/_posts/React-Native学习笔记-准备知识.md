---
title: React Native学习笔记--准备知识
date: 2016-1-22 18:58:53
tags: React Native
---

学习React Native,需要哪些知识

React Native 不是一种语言,是各种工具及语言的集合,但他也有自己的语法JSX,所以学习他,需要有一定的知识

**1:需要掌握的知识:**

-    Javascript语法

-    React/JSX的语法

-    ES6的新特性语法

-    些许的iOS或者Android的原生开发的知识

  <!--more-->

  **2:硬件准备**

- Mac机器,为什么用mac机器,因为Facebook的团队也是用mac开发的ReactNative,所以Mac机器对React Native 的支持比较好,其次它能显著提高你的开发效率

- 苹果手机,为什么是苹果手机,因为我接下来用它来学习,android出现的莫名问题较多,初学阶段不好定位错误

**3:需要了解的知识:**

[Homebrew](http://brew.sh/)

- [Homebrew](http://brew.sh/) 是 Mac 中的一个包管理器。没有安装的话，[点击这里安装](http://brew.sh/) 我的版本如下:

  ```
  mac-2:~ srain$ brew -v
  Homebrew 0.9.5 (git revision ac9a7; last commit 2015-09-21)
  ```



  ```

  版本过低将会导致无法安装后续几个组件。可用 `brew update` 升级。

  ```
  mac-2:react-native srain$ brew update
  Updated Homebrew from ac9a71c8 to 4257c3da.



  ```

- [Node.js](https://nodejs.org/) 和 [npm](https://docs.npmjs.com/)

  [Node.js](https://nodejs.org/) 需要 `4.0` 及其以上版本。安装好之后，[npm](https://docs.npmjs.com/) 也有了。

- - 通过 [nvm](https://github.com/creationix/nvm#installation) 安装 [Node.js](https://nodejs.org/)

    [nvm](https://github.com/creationix/nvm#installation) 是 [Node.js](https://nodejs.org/) 的版本管理器，可以轻松安装各个版本的 [Node.js](https://nodejs.org/) 版本。

    安装 [nvm](https://github.com/creationix/nvm#installation) 可以通过 [Homebrew](http://brew.sh/) 安装:

  ```
    brew install nvm



    ​```

    或者 [按照这里的方法](https://github.com/creationix/nvm#installation) 安装。

    然后安装 [Node.js](https://nodejs.org/)：

    ​```
    nvm install node && nvm alias default node



    ​```

  - 也可以直接下载安装 [Node.js](https://nodejs.org/)： <https://nodejs.org/en/download/>

- ```
  mac-2:react-native srain$ node -v
  v4.0.0
  mac-2:react-native srain$ npm -v
  2.14.2
  ```



  ```

  安装好之后，如下：

- 安装 [watchman](https://facebook.github.io/watchman/docs/install.html) 和 [flow](http://www.flowtype.org/)

  这两个包分别是监控文件变化和类型检查的。安装如下：

  ```
  brew install watchman
  brew install flow


  ```

  ```