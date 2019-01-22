---
layout: post
title: "Sbt 依赖配置"
date: 2019-01-21 17:57:57 +0800
author: "尹天宇"
tags:
    - Scala
    - sbt
---


# sbt依赖配置

## 前言
相信大家在使用chisel项目的时候都有遇到过**库依赖**的问题，比如从[rocket-chip](https://github.com/freechipsproject/rocket-chip)中clone下来的项目在import到IntelliJ Idea的过程中会出现

![](/img/blog-sbt-error.png)

这种错误。这其实就是由于在sbt构建项目的过程中找不到chisel3这个库（你在写代码的时候会`import chisel3`)。

那么如何解决这个问题呢？

## sbt的库依赖

sbt的库依赖有两种方式，一种是**非托管依赖**，一种是**托管依赖**。

非托管依赖比较简单，只要将打包好的`.jar`文件放置在`lib`文件夹下，它们就会被自动添加到项目的classpath中。

一般我们使用的库都是托管依赖的，托管依赖配置在构建定义(即build.sbt)中，在sbt构建项目的时候会自动从仓库(repository)中下载。

本文主要讲托管依赖。

### 托管依赖

sbt使用[Apache Ivy](http://ant.apache.org/ivy/)来实现托管依赖。

大多数时候我们在libraryDependencides设置项中列出我们需要的依赖。也可以通过Maven POM文件或者Ivy配置文件来配置依赖，而且可以通过sbt来调用这些外部的配置文件。

定义一个依赖的方法如下，其中groupId, artifactId和revision都是字符串。

`libraryDependicies += groupId % artifactId % revision`

有的时候需要在依赖库的名称中加上版本号，这时一般通过%%方法来实现，比如

`libraryDependencies += "org.scala-tools" % "scala-stm_2.11.1" % "0.3"`

在scalaVersion为2.11.1的情况下和下面这一行是等效的

`libraryDependencies += "org.scala-tools" %% "scala-stm" % "0.3"`

当然如果你需要不同scala版本的库同时工作，%%就没那么智能了，需要手动配置这些版本号。

## 发布
上一节中写好的依赖，如果是sbt能找到的，那就会去网上自动下载。但是rocket-chip使用的许多库都是需要自己[publish](https://www.scala-sbt.org/1.x/docs/Publishing.html)的。

sbt有publish和publishLocal这两个操作，其中publish是发布到互联网上，需要指定仓库，而publishLocal是发布到本地~/.ivy2/local中，只供本地使用。

由于rocket-chip及相关的项目只使用publishLocal，故本文只介绍publishLocal。

publishLocal命令把项目发布到本地的Ivy仓库中，默认位置是~/.ivy2/local。这样本地其它的项目就可以使用这些依赖。比如说，当你发布一个配置信息如下的项目后

```
name := "My Project"
organization := "org.me"
version := "0.1-SNAPSHOT"
```

你就可以在别的项目中这样来使用它

```
libaryDependencies += "org.me" %% "my-project" % "0.1-S"
```



## 实作
我们现在来尝试解决刚才遇到的rocket-chip无法构建的问题。

本文的测试环境：

 * 操作系统: WSL(Windows Subsystem for Linux) Ubuntu 16.04(本人也在真实的Ubuntu 16.04、CentOS 7以及macOS Mojave上测试通过，Windows理论上也没有问题)
 * sbt 1.2.8
 
当出现在*前言*中图所示的问题的时候，就意味着sbt无法找到chisel3_2.11的库，也就是我们需要去rocket-chip项目的子模块chisel3中手工publish它。

在我今天(2019年1月21日)下载的代码中，chisel3的build.sbt中设置的版本为`scalaVersion := "2.12.6"`。这样一来，如果直接在chisel3目录下执行`sbt publishLocal`就会在~/.ivy/local/edu.berkeley.cs目录下创建chisel3_2.12这个目录，包含了这个库的所有内容。

但是图中显示我们需要的是chisel3_2.11的库，而创建的是chisel3_2.12的库，这样即使发布了这个库还是不能被rocket-chip使用。

因此我们需要修改chisel3的build.sbt，将对应行修改为`scalaVersion := "2.11.12"`，再次`sbt publishLocal`即可。

修改之后在发布chisel3_2.11时由出现了firrtl_2.11找不到的问题，只要如法炮制，将firrtl的build.sbt中对应行修改为`scalaVersion := "2.11.12"`，再将firrtl发布一下。之后再发布chisel3_2.11。

此后再遇到问题只要重复上述步骤，缺少什么库就发布什么库，同时注意版本即可(有的时候同一个库的2.11和2.12两个版本都被使用，需要发布两次)。

