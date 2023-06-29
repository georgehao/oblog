---
title: "话说golang channel"
date: 2019-08-06T23:15:26+08:00
lastmod: 2019-08-06T23:15:26+08:00
draft: true
keywords: [golang, channel]
tags: [golang]
categories: [golang]
---

## 基本用法

## 一些问题

1. make(chan int)的长度是多少?
2. len(chan), cap(chan)是如何计算的
3. make(chan int)的长度为0, 但是为什么能够send数据?
4. 为什么nil channel会不管recv或者send都是deadlock?
5. 为什么channel会阻塞?
6. 非缓冲channel与缓冲channel的区别?

## 原理

### make

### receive

### send