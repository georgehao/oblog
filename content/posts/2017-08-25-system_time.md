---
title: 修改系统时间导致的坑
author: haohongfan
type: post
date: 2017-08-25T11:53:30+00:00
url: /index.php/2017/08/25/system_time/
categories:
  - c++

---
有一天测试人员对我说, 我怎么测试`10点开站会`这个功能呢? 当时也没有经过脑子, 直接对她说, 你把系统时间修改一下吧.

好嘛, 麻烦来了. 测试对我说, 你新开发的程序有BUG, 程序没反应了. 我晕, 哥已经测过的, 怎么会有问题呢? 然后我就做在那开发排地雷. 后来经过仔细排查, 排查到一个别人封装的接口, 我把那个程序大致的样子写出来.

![sleep_for.png](http://images.haohongfan.com/1595110-357257f58ddb1a40.png?imageView2/2/w/800/h/500)

程序总是在这个函数中阻塞住. 为什么会阻塞呢? 然后我就在这段代码中到处加log打印

直到我把`std::cout << sleep before << std::endl`加在23行时, 我终于发现问题了, 发现程序阻塞在24行

`boost::this_thread::sleep_for`是不能修改系统时间的!!!
