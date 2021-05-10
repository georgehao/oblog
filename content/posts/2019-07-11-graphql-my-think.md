---
title: "Graphql 之我所想"
date: 2020-03-31T11:45:50+08:00
lastmod: 2020-03-31T11:45:50+08:00
keywords: [graphql]
tags: [graphql]
categories: [graphql]
draft: true
---

![graphql](http://images.haohongfan.com/graphql.png?imageView2/1/w/1000/h/550)

上家公司主项目由于某些今天看来非常次要的原因, 服务器接口从 rest api 切换到 graqhql. 从

## Graphql

一种用于 API 的查询语言, 起源于facebook. GraphQL 既是一种用于 API 的查询语言也是一个满足你数据查询的运行时。 GraphQL 对你的 API 中的数据提供了一套易于理解的完整描述，使得客户端能够准确地获得它需要的数据，而且没有任何冗余，也让 API 更容易地随着时间推移而演进，还能用于构建强大的开发者工具

* 请求你所要的数据不多不少. 客户端可以自定义需要的字段
* 获取多个资源只用一个请求. graphql只有一个endpoint, 只有post请求, 一次请求可以获取很多接口的数据
* 描述所有的可能类型系统. graphql是强类型约束, 避免rest api中客户端与服务端类型不一致导致app崩溃
* 强大的开发者工具GraphiQL. 实际上这个GraphiQL挺鸡肋的, 建议使用[insomnia](https://github.com/getinsomnia/insomnia)
* API 演进无需划分版本. 在rest api版本划分的问题, 如果处理不好这个确实挺麻烦的
* 使用你现有的数据和代码
* 强大的背书. github, facebook

一切看起来都是那么完美

## 殇之泪 