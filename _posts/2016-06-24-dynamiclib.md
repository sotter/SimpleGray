
---
layout: post
title: 动态库问题记录
category: default
---

#### 0\. 两个有用的命令nm ldd

nm可以查看库或者可行文件打包的命令和函数，ldd可以查看可执行文件的动态库引用；

在实际应用中，可以有下面的用处：
- mogu_app_server引用了一个静态库，但是又不知道引用的是静态库的哪个版本，如果两个版本有函数名不一样，可以通过nm mogu_app_server|grep function_name, 来看看打包进了哪个函数名；

- ldd可以查看动态库的引用情况, 如mogu_app_server，需要注意的是编译链接的时候链接的动态库，跟运营时加载的是不一样的；


另外还有strings，可以把库和可执行文件的字符搜索出来，几个工具配合使用，一般各种库的依赖和编译问题都能搞定；


#### 1\. 动态库编译连接问题：如果a.out调用了liba.so的函数，liba.so的又引用了libb.so中函数：

编译问题：
	如果使用liba.so在生成一个动态库或者静态库，是不需要的指定-la -lb, 因为没有生成可执行文件，没有经过链接阶段；

	但如果是生成可执行文件，需要同时指定-la -lb

#### 2\. 对一个静态库来说，如果一个函数在调用链中没有被调用是否编译进a.out中

这个问题目前我也没有找到统一的结论，测试后观察到以下现象：

- 可执行程序并不是把静态库中的所有函数都打包进去；
- 原先是猜测是根据main函数然后向下推导调用链，在这个调用链上所有的函数都打包进去，后来验证，如果一个静态库中的函数没有任何人调用，也打包进可行程序中了；
- 后面猜测是根据头文件的函数向下推导调用链打包：实验中发现头文件中有的函数打包进去了，有的没有打包进去；

今天了买了一本书专门研究这链接和装载的，有结论了再跟大家分享；

#### 3\. 动态库编译的版本号

如下所示，mogu_app_server为什么链接指向的是libcdnbalance.so.1.2 而不是1.2.2 或是1.3或是其他；

```
	[root@guomai013030 /usr/local/im-cloud/run] 21:30
	$ ldd mogu_app_server
		linux-vdso.so.1 =>  (0x00007fff96360000)
		libpthread.so.0 => /lib64/libpthread.so.0 (0x0000003135000000)
		librt.so.1 => /lib64/librt.so.1 (0x0000003135800000)
		libdl.so.2 => /lib64/libdl.so.2 (0x0000003135400000)
		libcdnbalance.so.1.2 => /opt/im-cloud-server-thirdparty/linux/lib/libcdnbalance.so.1.2 (0x00007f08cfeba000)
```

实际上程序在编译链接阶段就决定了，链接器找的是这个库的SONAME，具体可参考下面的文章：
[Linux程序编译链接动态库版本的问题](http://littlewhite.us/archives/301)

所以在实际应用用就会产生这样的问题： 一个程序从一台机器拷贝到另一台机器后：

- 根据SONAME找不到REALNAME，导致运行不了；
- 两台机器的库的小版本号不一致，导致同样的程序在不同的机器上运行结果又差异；


#### 4\. glib库不兼容的问题：

程序运行不了，LDD查看如下所示；

```
	./netflow_server: /usr/lib64/libstdc++.so.6: version `CXXABI_1.3.8' not found (required by ./netflow_server)
	./netflow_server: /usr/lib64/libstdc++.so.6: version `GLIBCXX_3.4.17' not found (required by ./netflow_server)
	./netflow_server: /usr/lib64/libstdc++.so.6: version `GLIBCXX_3.4.18' not found (required by ./netflow_server)
	./netflow_server: /usr/lib64/libstdc++.so.6: version `GLIBCXX_3.4.19' not found (required by ./netflow_server)
	./netflow_server: /usr/lib64/libstdc++.so.6: version `CXXABI_1.3.5' not found (required by ./netflow_server)
	./netflow_server: /usr/lib64/libstdc++.so.6: version `GLIBCXX_3.4.14' not found (required by ./netflow_server)
	./netflow_server: /usr/lib64/libstdc++.so.6: version `GLIBCXX_3.4.15' not found (required by ./netflow_server)
	./netflow_server: /usr/lib64/libstdc++.so.6: version `GLIBCXX_3.4.20' not found (required by ./netflow_server)
```

具体可参考下面的文章： 
https://github.com/qiwsir/ITArticles/blob/master/Linux/How_to_solve_GLIBCXX_3.4.19.md


#### 总结：

1. 我们的机器环境各种各样，每次搭建基本上都是各种库的依赖问题，有的操作系统中安装的，有的是third-party中装的；同时third-party中的静态库又有链接静态库的依赖；

2. 动态库和静态库的有缺点：
	- 静态库链接进去后，在新环境中运行不需要再安装库方便，缺点是如果是有版本的兼容性问题不容易看出来；比如一个mogu_app_server拷贝过来，到底是引用的1.2.2 cdn 还是1.2.3版本的cdn看不出来；

	- 动态库：可执行程序占用空间下，能方便的知道可执行程序依赖的版本号；缺点就是我们目前的现状，1. 机器环境不统一 2. 外部库不都是从third-party中过来的

	建议以后还是用动态库，把我们程序依赖的库放到同一的地方deps种，然后start.sh脚本中自动设置LD_LIBRARY_PATH;

推荐这本书： [程序员的自我修养：链接、装载与库](http://item.jd.com/10067200.html)   今天到货，一睹为快！


