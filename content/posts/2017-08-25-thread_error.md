---
title: 记一次线程不同步的坑
author: haohongfan
type: post
date: 2017-08-25T11:52:58+00:00
url: /index.php/2017/08/25/thread_error/
categories:
  - c++

---
![memcpy_Image.png](http://images.haohongfan.com/1595110-8031dcc84055fd0b.png?imageView2/2/w/800/h/500)

这段简单的代码, 总是在某些时候, 就会出现`p_cur_ctx->GetCallbackContent()`不为空, 但是总是memcpy失败的问题. 这下无语了, 系统库有问题了吗 ?! WTF, 遇见鬼了, 从来没有见过如此诡异的问题.

![gdb p_cur_ctx->GetCallbackContent()](http://images.haohongfan.com/1595110-541975b74d377fe7.png?imageView2/2/w/800/h/500)

![gdb tmp](http://images.haohongfan.com/1595110-b16483b7c352bb02.png?imageView2/2/w/800/h/500)

#### _怎么看, 这都是不可能发生的事情!!!_, 为什么memcpy会失败呢?

再看GetCallbackContent()的代码实现

![GetCallbackContent实现](http://images.haohongfan.com/1595110-a0a318ab6fad4285.png?imageView2/2/w/800/h/500)

感觉上没有什么问题, 这个问题就诡异了.为什么以前程序跑的好好的, 突然间连memcpy都不靠谱了.这个世界是怎么了.

坐下来仔细想想, 发现这个原来是自己设计框架时候留下的`坑`.

![坑](http://images.haohongfan.com/1595110-3fffae669f4b3cd9.png?imageView2/2/w/800/h/500)

当时设计的线程安全的单例模式, 然后也没有仔细考虑两个线程的同步的问题, 是粗心了.

修改后的程序
  
![Paste_Image.png](http://images.haohongfan.com/1595110-3db75e583e605ad4.png?imageView2/2/w/800/h/500)

再也没有出现这个问题. 哎, 到处都是坑.