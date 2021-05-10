---
title: 使用System V信号量同步引起的Interrupted system call坑
author: haohongfan
type: post
date: 2017-08-25T11:54:11+00:00
url: /index.php/2017/08/25/ipc_interrupted_system_call/
categories:
  - c++

---
使用System V信号量不是那么熟练. 写了一个Monitor监测线程, 一个实际执行的Product线程.
  
本来的想法两个线程一个一个的调, 但是我忘了把另外一个线程注释掉了.然后**坑**就这么产生了.

先看Monitor代码:

![](http://images.haohongfan.com/1595110-3a9524f1f428ae78.png?imageView2/2/w/800/h/500)

然后gdb去调试的时候, 总是出现

![error](http://images.haohongfan.com/1595110-f1ac6dee7f647d67.png?imageView2/2/w/800/h/500)

我一开始以为我的程序出错了, 因为我System V信号量用的不是很熟练, 但是我发现我写的没有错呀, 都是这样用的呀. 一番查找下, 我就确定我这样用是没错的, 就去Google了一下, 发现了真相.

原来GDB在调试的时候是会发生这种情况的.[具体的内容自己查看](https://sourceware.org/gdb/onlinedocs/gdb/Interrupted-System-Calls.html)

### GDB 引起Interrupted system call原因

* 必须是多线程
* 其中一个线程必须因为system call阻塞(例如程序中的semop)
* 而另外一个程序由于断点或者其他什么原因, 就会引起正在阻塞的线程提前结束

文档中还说明了, GDB使用内核中断的方式监测线程库, 例如线程的创建、销毁等情况. 如果当这些情况发生时,尽管程序在我们的预想中不能结束, 但是系统调用可能提前结束, 就导致出现Interrupted system call

尽管文档没有详细说明引起的原因, 但是已经够了. 我知道了我的程序出错的原因, 我把另外一个线程注释掉, 然后再调这个线程, 这没有这个问题发生了.

PS:如果不注释掉另外一个线程, 但是不用GDB去跑, 也不会发生Interrupted system call, 这和文档说明的情况是一样的.

**add:2016-9-22**
  
今天才发现原来《UNIX网络编程--进程间通信》[P230]中也有说明
  
"当一个线程被投入睡眠以等待某个信号量操作完成之时(我们将看到该线程既可以等待改信号量的值变为0, 也可等待它变为大于0), 如果它捕获了一个信号, 那么其信号处理程序的返回将中断引起睡眠的semop函数, 该函数于是返回一个EINTR错误.按照UNPv1的术语定义, **semop是需被所捕获的信号中断的慢系统调用**"

### 解决方法:

解决方法其实很简单,就是我们要接受一下Interrupted system call(EINTR)信号就可以了
  
比如上面的代码,修改如下

![](http://images.haohongfan.com/1595110-0992de709b14ca83.png?imageView2/2/w/800/h/500)

### semop用法

`int semop(int semid, struct sembuf *opstr, size_t nops)`
    
> semval: 信号量当前值 

```
struct sembuf {
    short sem_num;
    short sem_op;
    sem_flag;
}
```
  
* sem_op是正数, 其值就加到semval上, 对应于释放由某个信号控制的资源
* sem_op是0, 如果调用者希望等待到semval为0, 如果semval已经是0, 那么立即返回; 如果semval不为0, 调用线程则被阻塞到semval变为0
* 如果sem_op是负数,那么调用者希望等待semval变为大于或者等于semop的绝对值.
* 如果semval大于或者等于sem_op的绝对值, 那就从semval中减掉sem_op的绝对值.
* 如果semval小于sem_op的绝对值, 那么调用线程就阻塞到semval变为大于或者等于sem_op的绝对值. 到线程解除阻塞, 还将从semval中减掉sem_op的绝对值.

### 代码

有关详细封装的代码请查看我的[github](https://github.com/georgehao/wheel/tree/master/shared_buffer)