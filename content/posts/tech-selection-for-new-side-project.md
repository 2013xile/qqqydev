---
title: 新的Golang side project技术选型记录
date: "2023-09-10"
categories:
  - "Golang"
  - "Backend"
---

> 打算利用工作之余的时间，写一个side project, 简单记录一下前期的选型对比。

# 语言 `Go`

这个project主要目标还是一个偏Web类的应用，这类应用用JS来撸肯定是最快最爽的，但是因为工作中的主要语言就是JS，所以想换换口味。在上一份工作里有一半时间是写Go的，离职以后已经很久没用了，也不能荒废了，之前的工作写的也是偏业务功能的代码居多，没有很深入研究Go, 所以想趁此机会捡起来一些。Go用于开发Web应用也是比较不错的。而且我这个项目最终大概率是一个在本地运行居多的东西，我又不太想做成一个桌面应用，理想情况是利用一个可执行文件启动应用服务以后，用本地浏览器做应用载体，当然这个文件最终也可以打包成各个平台的软件包。Go的话利用 `go:embed` 可以很方面地把前端产物打到最终的二进制文件里。综合上述几点原因，最终选择的开发语言是**Go**.

# Web框架 `fiber`

我对Web框架的要求基本上就是能处理 `router` 和 `handler` 就可以，Go在这方面的库和轮子太多了，用过[gin](https://github.com/gin-gonic/gin)和[mux](https://github.com/gorilla/mux), 甚至我觉得简单的场景Go自带的 `http` 库也能用用。好像现在[fiber](https://github.com/gofiber/fiber) 还挺流行的，看了介绍，"Express inspired web framework", 这个我熟，就它了。

# ORM框架 `Bun`

前面都比较顺利，在ORM框架选择上，费了不少功夫。其实在以前我基本上很少用ORM框架，最多用query builder库来拼SQL。我要做的这个应用可能会有兼容多数据库的需要，所以写raw sql不太现实，也不能把时间都花在做sql兼容适配上，所以大概还是需要这么个东西。  
本来我是想用以前我用的挺好的一个query builder的库 [goqu](https://github.com/doug-martin/goqu), 但是看Github上面的代码最新提交已经是2年前了，比较想选择一个相对活跃的库，另外就是我还需要 _migration_ 的功能，这个库也没有。  
 在[awesome-go](https://awesome-go.com/) 上看了一圈，发现query builder的库基本上都不带 _migration_ 功能，也想过用query builder库 + migration库，但是很多单独的 `migration` 库都是通过写raw sql来做的，没有实现不同数据库的兼容。
最后还是只能回头来看ORM, [xorm](https://xorm.io/) 和 [gorm](https://gorm.io/) 我是用过的，基本上我对这两东西是拒绝的，很多东西太重了，还有光定义 `Model` 和转换输出就麻烦。[ent](https://github.com/ent/ent) 算是另一种层面的ORM，但是看了下文档，感觉代码还是很冗杂，本质上没有降低写代码的心智负担。  
反反复复看了看，还真让我找到一个比较合适的—— [bun](https://bun.uptrace.dev/). 用法上基本是一个query builder, 没有各种复杂的Relation需要定义，兼容常见的数据库，支持 _migration_ 功能，更新也比较活跃。不过具体体验还得用了才知道。
