---
title: 有关LD_LIBRARY_PATH与ld.so.conf
author: haohongfan
type: post
date: 2017-08-25T11:55:53+00:00
categories:
  - c++
  - cmake

---

我之前写过一篇关于[LD_LIBRARY_PATH与gcc/g++ -L的关系的文章](http://www.jianshu.com/p/f0f4700d5611). 在用CPACK制作了一个Debian安装包，然后我在/home/.bashrc里添加了export `LD_LIBRARY_PATH=/usr/loca/lib:$LD_LIBRARY_PATH`, 这个不够优美, 经过一番寻找终于找到了---ld.so.conf可以完美解决这个问题。
 
### 为什么LD_LIBRARY_PATH不行？

[可以看看老外是怎么说的](http://xahlee.info/UnixResource_dir/_/ldpath.html)

升级共享库时，替换之前先测试一下类似的，升级后的某个程序可能依赖于一些动态链接库，如果你将某个链接库替换了，程序可能就无法工作了。这时候，你可以使用LD_LIBRARY_PATH指向存有备份的一个目录，然后，你可以没有顾忌地替换系统版本了。万一出错，拷贝回去就是了

感觉LD_LIBRARY_PATH就是临时使用的，为什么呢？

因为LD_LIBRARY_PATH如果设置成全局的话，如果被破坏掉的，那么就会出现大规模的破坏。不要说这个变量不会改，但是这样做就是不妥的办法，网上大家可以找到许许多多的关于LD_LIBRARY_PATH不好的文章。这里就不再多说
 
如果不让用LD_LIBRARY_PATH，我们该怎么办呢？Linux系统为大家已经想好了办法，我们使用ld.so.conf来解决。

我们已经知道linux在加载动态库的时候，会去标准路径下(/lib,/usr/lib)下去寻找应用程序用到的动态库。但对于我们那些不标准的路径下我们安装了lib库，例如我把我的库安装到了/usr/local/lib下了，我们怎么办呢?
 
Linux的通常做法是：将非标准路经加入 /etc/ld.so.conf，然后运行 ldconfig 生成 ld.so.cache。 Linux在加载共享库的时候，会从 ld.so.cache 查找。

所以，我们在安装了库，但是编译程序后，ldd发现链接不到库，那么我们就要查看你安装的库是否是非标准路径，如果是那么就把你的非标准路径加入到/etc/ld.so.conf文件中，然后调用ldconfig生成下ld.so.cache，就可以了。想查看下你的库是否已经在ld.so.cache中，可以这样 ldconfig -p | grep lib**就可以了。
 
对于，Ubuntu来说，还与其他的LINUX系统不一样，在/etc/ld.so.conf中只有一句include /etc/ld.so.conf.d/*.conf，也就是说它我们不能在/etc/ld.so.conf下添加，但是我们可以在/etc/ld.so.conf.d下新建一个*.conf在这里面添加你的非标准路径就可以了，记得调用sudo ldconfig 生成ld.so.cache文件就可以了
 
***补充：***
 
说说今天又遇到的问题：（cmake之后出现的问题）
问题1：`/usr/bin/ld: warning: libboost_system.so.1.55.0, need by /usr/local/lib/libboost_thread.so, may conflict with libboost_system.so.1.48.0`

问题2: `.text._ZN5boost16exceptions_detail10bad_alloc_D2Ev' refereced section .text._ZN5boost16exceptions_detail10bad_alloc_D1Ev'..... of /usr/lib/gcc/x86_64-linux-gnu/4.8.2/../../../libboost_thread.a(thread.o)`
 
这两个问题折腾我了一下午，这两个问题的变现不一样，其实他们都是一个问题造成的。
首先说说为什么会这样？
第一个问题：是由于系统里面在/usr/lib下面有一个libboost_system.so.1.48.0，而在/usr/local/lib下我安装了一个libboost_system.so.1.55.0，造成了g++链接时的冲突
第二个问题：是由于在/usr/lib下有一个libboost_thread.a，而我在/usr/local/lib下安装了libboost_thread.so.1.55.0，造成了g++链接的冲突错误。
 
现在问题来了，他们为什么会冲突呢？为什么呢？我们在编译程序的时候，gcc/g++是怎样搜索链接库呢？
经过仔细寻找，我们使用 ld --verbose | grep SEARCH_DIR会出现下面的内容
SEARCH_DIR("/usr/i686-linux-gnu/lib32"); SEARCH_DIR("=/usr/local/lib32"); SEARCH_DIR("=/lib32"); SEARCH_DIR("=/usr/lib32");SEARCH_DIR("=/usr/local/lib/i386-linux-gnu"); SEARCH_DIR("=/usr/local/lib"); SEARCH_DIR("=/lib/i386-linux-gnu"); SEARCH_DIR("=/lib");SEARCH_DIR("=/usr/lib/i386-linux-gnu"); SEARCH_DIR("=/usr/lib");
 
这就是gcc再编译的时候，所搜寻的库文件所在的路径，所以就出现了上面的问题。当gcc/g++在/usr/lib中发现这些库之后，就不再搜索了，然后就让gcc/g++链接了这些库，而我需要的正确的库应该在/usr/local/lib下，这就造成了链接了错误的库文件。
解决方法就是：删除掉/usr/lib下同名的不需要的或者版本过低的库文件（目前是这样做的，也许有更好的方法，知道的请告诉我）
 
上面查看gcc/g++/ld搜索路径还有其他方法：
``` 
gcc -Wl,  --verbose 
gcc --print-search-dirs
```