---
title: erlang:now()与os:timestamp()-Erlang源码学习一
date: 2018-06-20 19:44:02
categories:
- 开发
tags:
- Erlang
---

erlang中，关于erlang:now()与os:timestamp()两个接口，查看官方文档的解释：

![erlang_time1]

![erlang_time2]

按官方文档上说erlang:now/0是废弃了的，它可以获取一个持续递增的唯一时间戳。除此也没说讲到更多。

再看erlang:now/0文档给的[时间和时间修正][1]，里面详细描述了erlang对于时间的处理，暂不看。

直接跳到c源码看吧，在这之前可以看看[linux内核时间的管理][2]，明白什么是墙上时间(wall time)、单调递增时间(monotonic time)等。

# erlang:now/0

    erlang:now/0的bif函数对应erlang源码:erts/emulator/beam/bif.c

![erlang_time3]

`now_0()`的获取时间调用`get_now()`函数，位于erts/emulator/beam/erl_time_sup.c

![erlang_time4]

获取时间主要用一个回调`get_time()`，而获取时间之后会与上一次调用产生的值比较，并产生一个新的保证单调递增的唯一值，并加锁修改旧值。关于`get_time()`的初始化要在同文件的`erlang_init_time_sup()`函数


![erlang_time5]

可以看到有一个条件编译宏 ERTS_HAVE_OS_MONOTONIC_TIME_SUPPORT，在没有配置erlang源码时，这些宏都未定义，但如果在源码根目录执行了./configure配置后，会在emulator目录生成一个文件夹（我的是x86_64-unknown-linux-gnu），里面放有config.h配置文件，里面根据操作系统类型做了对应宏定义，对应宏ERTS_HAVE_OS_MONOTONIC_TIME_SUPPORT就是在config.h里，表示这个操作系统有单调递增时间(monotonic time)，那么这里可以看到get_time回调指向了`get_os_grift_corrected_time()`，此函数直接返回`read_corrected_time()`函数结果：

![erlang_time6]

可以看到`read_corrected_time()`函数主要执行了`erts_os_monitonic_time()`和`calc_corrected_erl_mtime()`来获取操作系统的monitonic_time以及修正时间。
`erts_os_monitonic_time()`直接调用`posix_clock_gettime()`函数，参数为MONOTONIC_CLOCK_ID，获取monotonic time：

![erlang_time7]

获取时间的函数实则调用了linux的系统函数`clock_gettime()`，可以man手册看一看；
`calc_corrected_erl_time()`函数：

![erlang_time8]

将获取的当前操作系统monitonic time与最近一次更新的操作系统monitonic time做一个差值计算，然后根据erlang时间的设计来计算一个新的erlang monitonic time。

# os:timestamp/0

`os:timestamp()`函数代码：

![erlang_time9]

代码里调用了`erts_os_system_time()`函数来获取操作系统的时间：

![erlang_time10]

又是熟悉的`posix_clock_gettime()`，并且参数为WALL_CLOCK_ID获取墙上时间。


# 总结

`erlang:now/0`获取的是erlang系统的monotonic time，它从操作系统获取后还要用erlang时间处理的方式在调整为erlang monotonic time，期间几次会对全局变量加锁，故效率会有损耗，而每一次获取时间值后会与上一次获取的值做一个对比，并加一来保证获取值的严格单调递增，所以可以用来作为唯一名(unique name)的生成，但是，erts7.0之后就不建议用这个函数了，可以用`erlang:timestamp/0`替代，如果要生成唯一名可以用`erlang:unique_integer/0`等等。 而`os:timestamp/0`则是获取操作系统的墙上时间(wall time)，并做调整变为erlang system time。

`erlang:now/0`获取时间的文件为erl_time_sup.c，所有的erlang关于时间处理方式的逻辑都定义在里面，包括维护全局变量来处理操作系统转变为erlang规则的内部时间、注册定时器周期检查erlang时间与os时间对比的偏移量等；`os:timestamp/0`获取时间的文件为sys_time.c，是对os获取时间的库函数的封装、以及对获取的时间进行简单调整的文件，比较轻量级。

后记：看了erlang时间的处理，感觉很有趣，后面学习一下erlang源码中对时间的管理代码。


[1]:http://erlang.org/doc/apps/erts/time_correction.html#Dos_and_Donts
[2]:https://blog.csdn.net/hytgxljh/article/details/52440837

[erlang_time1]:/img/erlang_time1.png ""
[erlang_time2]:/img/erlang_time2.png ""
[erlang_time3]:/img/erlang_time3.png ""
[erlang_time4]:/img/erlang_time4.png ""
[erlang_time5]:/img/erlang_time5.png ""
[erlang_time6]:/img/erlang_time6.png ""
[erlang_time7]:/img/erlang_time7.png ""
[erlang_time8]:/img/erlang_time8.png ""
[erlang_time9]:/img/erlang_time9.png ""
[erlang_time10]:/img/erlang_time10.png ""