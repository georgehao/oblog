---
title: LD_LIBRARY_PATH与-L的关系以及延伸
author: haohongfan
type: post
date: 2017-08-25T11:56:24+00:00
categories:
  - c++
  - cmake

---
最近跟同学讨论c++在编译时g++ -L.. 和LD_LIBRARY_PATH的问题，今天在做一个东西的时候发现，我对这两个东西的理解是错误的，经过一番研究，写下我对这些东西的想法，如果有不对的地方，欢迎指正。
 
我遇到的问题：

`g++ multiple.cpp -L/usr/local/lib -lboost_program_options`编译完后，`ldd ./a.out`发现`libboost_program_options.so.1.55.0 => not found`

但是，当我在.bashrc里面写入：export LD_LIBRARY_PATH=/usr/local/lib，然后再编译，我发现可以了，这是为什么呢？
 
LD_LIBRARY_PATH是一个环境变量，它的作用是让动态链接库加载器(ld.so)在运行时有一个额外的选项，即增加一个搜索路径列表。注意，LD_LIBRARY_PATH是在运行时，才起作用。这个环境变量中，可以存储多个路径，用冒号分隔。它的厉害之处在于，搜索LD_LIBRARY_PATH所列路径的顺序，先于嵌入到二进制文件中的运行时搜索路径，也先于系统默认加载路径(如/usr/lib) 摘自：[http://www.ituring.com.cn/article/22101]
 
g++ -L:是在编译的时候，去-L指定的地方找库，-l库的名字上面问题的解释：我在编译的时候，用-L -l使程序正确编译了，但是没有指定运行时库去哪些地方寻找库，而默认的地方不包括/usr/loca/lib，这就造成了./a.out找不到libboost_program_options.so库，也就出现了上面的问题。
 
**缺点**
LD_LIBRARY_PATH是库一般是在run-time linking搜索的路径，但是使用这种方法不管在compiling linking还是run-time linking都是有严重的副作用, gcc/g++ -L在compiling linking搜索的路径，保证你所编译所需的符号是存在的, 如果设置了LD_LIBRARY_PATH，那么你设置的路径会优先寻找，而且比默认的标准路径还要早。