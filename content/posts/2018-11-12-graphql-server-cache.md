---
title: "GraphQL Server Cache"
date: 2018-11-12T18:43:15+08:00
lastmod: 2018-11-12T18:43:15+08:00
draft: true
keywords: [graphql, cache]
categories: [graphql]
author: ""
---

# 为什么要使用缓存

API缓存能够减轻服务端压力, 加快响应速度, 加快客户端渲染速度. 但是GraphQL的请求都是发送同同一个endpiont, 所以现有的HTTP缓存方案对于GraphQL是无效的.

# 在什么层次使用缓存

强烈不建议将GraphQL查询的内容作为缓存的一部分, 因为查询内容的一小部分变化可能会导致缓存的激增. 一般缓存都是位于GraphQL之下, 而不是在GraphQL之上.

# 只使用缓存方案



# 参考
1. [Add support for caching](https://github.com/Folkloreatelier/laravel-graphql/issues/302)
2. [Server caching and cache invalidation](https://github.com/graphql/graphql-js/issues/819)