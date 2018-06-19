---
title: Linux daemon守护进程的创建
date: 2018-06-14 21:31:50
categories:
- 开发
tags:
- Linux
---

今天在看《UNIX网络编程》的时候，看到了守护进程的创建，代码中fork了两次，并且第一次fork后对子进程调用setsid()，有些懵。当时搜了下setsid也是看得有点云里雾里。后来折腾了一下午，才算有点明白，这里把自己的一点分析心得写上来，以后忘了可以翻翻：

这里我用ssh登陆ubuntu，在终端用vim编写。

# 第一次fork
我们先写一个简单的代码来fork并创建一个后台程序，代码如下：

    //testdaemon1.c
    #include <stdio.h>
    #include <unistd.h>
    #include <stdlib.h>
    #include <signal.h>

    int main()
    {
        pid_t   pid;

        if ( (pid = fork()) < 0 )
            return -1;
        else if ( pid )
            exit( 0 );
        signal( SIGHUP, SIG_IGN );
        int i = 10;
        while ( i-- )
            sleep( 5 );

        return 0;
    }

编译`gcc -o testdaemon1 testdaemon1.c`
运行`./testdaemon1`

然后我们用ps查看程序:`ps axjf | grep testdaemon1`
>1 28511 28510 28168 pts/18 28168 S 0 0:00 ./testdaemon1

这里看第二列是进程id(28511)，第四列是sessionId(28168)，第五列是控制终端(pts/18)，代码中子进程fork而来，会默认继承父进程的控制终端以及会话组。而如果我们查看linux后台守护进程的话，会发现进程控制终端为“？”，因此这个代码距离标准的守护进程还有距离，后面逐一分析。

什么是会话session呢？我大概的理解就是linux下进程组织结构分很多会话，一个会话有很多进程，会话的首进程会有一个控制终端，一个会话只能有一个控制终端。而我们现在用ssh登陆ubuntu创建了一个会话，此会话首进程id号就是28168,如果我们运行ps aux | grep 28168会输出下面值:
>root 28168 0.0 0.4 6904 4640 pts/18 Ss+ 16:06 0:00 -bash

可以看到这个进程是 bash进程shell终端，其控制终端是pts/18，此后我们在此终端运行的进程都属于以这个首进程id为sessionId的会话组，我们可以运行一个后台的top程序:
`top &`
`ps axjf | grep top`得到下面输出：
>28168 28670 28670 28168 pts/18 28672 T 0 0:00 _ top

再看28168的会话组：

`ps axjf | grep 28168`得到下面输出：

>28141 28168 28168 28168 pts/18 28693 Ss 0 0:00 _ -bash

>28168 28670 28670 28168 pts/18 28693 T 0 0:00 _ top

>28168 28693 28693 28168 pts/18 28693 R+ 0 0:00 _ ps axjf

>28168 28694 28693 28168 pts/18 28693 S+ 0 0:00 _ grep –color=auto 28168

>1 28691 28690 28168 pts/18 28693 S 0 0:00 ./testdaemon1

可以看到运行的几个后台程序sid都是28168。

说了这么多，这个会话组有什么作用呢？我们就要提到SIGHUP信号了，SIGHUP信号会在终端关闭时或者会话首进程退出时发送给会话组的其它进程，当其它进程收到此信号时如果不捕获或者忽略信号的话，默认会退出。
如果现在点击ssh窗口的×关闭终端界面或者kill -9 28618的话，下面所属的进程都会退出。

因此，我们在代码中要调用setsid()，将子进程与父进程会话分离、终端分离，我们加入setsis再编译执行：

`./testdaemon1`
执行`ps axjf | grep testdaemon1`
>1 28891 28891 28891 ? -1 Ss 0 0:00 ./testdaemon1

看到其所属会话组已经变成新的（它自己为首进程）会话组，而终端也变为？，这正是我们看linux下其他守护进程的状态。

# 第二次fork
第一次fork还不够，因为我们setsid之后，第一子进程变为新会话首进程，它有权限重新申请打开一个终端，为了避免这种情况，可以通过使进程不再成为会话组长来禁止进程重新打开控制终端，这就需要第二次调用fork()函数

# 改变工作路径
第二次fork之后，我们在第二子进程里可以进行一些守护进程属性设置了， 改变工作路径可以用chdir(“/”)，为什么要改变工作路径呢？《unix网络编程》中Stevens大神说得很清楚，这里搬过来“守护进程可能是在某个任意的文件系统中启动，如果仍然在其中，那么该文件系统就无法拆卸(unmounting)，除非使用潜在的破坏性的强制措施。”

# 关闭所有打开的文件描述符
守护进程从执行它的进程(当前的shell)继承所有打开的描述符。但没有现成的函数提供来检测打开的文件描述符，Stevens大神的解决办法是关闭前64个，而我在网上找到的有用getdtablesize()，有用NOFILE宏(sys/param.h)的，我的ubuntu前者为1024，后者为256，也不想去深究了。

# 重定向stdin、stdout、stderr
上一步关闭了0开始的描述符，而stdin stdout stderr分别对应0 1 2，因此关闭之后如果不进行重定向，守护进程分配文件描述符会从0开始，因此stevens大神打开/dev/null三次，分配了三次描述符，分别为012，都对应/dev/null


#
贴上整个代码吧（跟《UNIX网络编程》里代码有些出入，某些东西也懒得深究了）：

    #include <stdio.h>
    #include <unistd.h>
    #include <stdlib.h>
    #include <signal.h>

    int main()
    {
        pid_t   pid;

        //原代码的fork用的封装函数Fork，其实多了对fork查错
        if ( (pid = fork()) < 0 )
            return -1;
        else if ( pid )
            exit( 0 );

        setsid();

        //原代码用的封装函数Sianal，这里懒得写了
        //父进程退出，会给所有子进程发一个SIGHUP信号，进程接收到此信号如果不做捕获处理默认会退出
        signal( SIGHUP, SIG_IGN );

        if ( (pid = fork()) < 0 )
            return -1;
        else if ( pid )
            exit( 0 );
        //daemon_proc = 1;//原代码有这句
        chdir("/");

        int i = 0;
        for ( ; i < 64; i++ )
            close( i );

        open("/dev/null", O_RDONLY);
        open("/dev/null", O_RDWR);
        open("/dev/null", O_RDWR);

        //原代码有这句，就是往syslog里写运行本程序的日志，加上进程id，输出等级为facilit(LOG_WARN这样子)
        //openlog(pname, LOG_PID, facility);

        return 0;
    }


