---
title: "一次服务器 CLOSE_WAIT 引发的血案"
date: 2019-12-20T10:30:16+08:00
lastmod: 2019-12-20T10:30:16+08:00
keywords: [golang, http]
tags: [golang]
categories: [golang]
draft: true
---

## 参考

1. [线上大量CLOSE_WAIT的原因深入分析](https://juejin.im/post/5c0cf1ed6fb9a04a08217fcc)
2. [Go 长连接服务线上一次大量 CLOSE_WAIT 复盘](https://mp.weixin.qq.com/s/FuXLwe7l2qUB76w2K3OYDw)
3. [浅谈CLOSE_WAIT](https://blog.huoding.com/2016/01/19/488)
4. [又见CLOSE_WAIT](https://mp.weixin.qq.com/s?__biz=MzI4MjA4ODU0Ng==&mid=402163560&idx=1&sn=5269044286ce1d142cca1b5fed3efab1&3rd=MzA3MDU4NTYzMw==&scene=6#rd)
5. [Golang 大杀器之性能剖析 PProf](https://segmentfault.com/a/1190000016412013)
6. [pprof 和火焰图](https://xargin.com/pprof-and-flamegraph/)