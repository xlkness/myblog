---
title: c半同步半异步进程池模型之cgi服务器
date: 2018-06-15 18:47:28
categories:
- 开发
tags:
- C/C++
---

对半同步半异步进程池模型垂涎已久，这次中秋放假撸了下代码。

模型的简单介绍：主进程创建几个子进程作为工作进程，主进程监听客户端connect事件，一旦有连接事件，即通过round robin(简单轮询)选取一个子进程， 通过父子进程间的通信管道通知子进程有连接事件，子进程epoll监听到管道通信时间，即知道有客户端连接，因此进程accept，并将客户端连接套接字加入epoll监听事件，客户端浏览器发送get请求，客户端监听到请求事件，即调用封装好的客户端事件进行处理，本例子的客户端处理为recv客户的请求，从中提取文件名(now.cgi)，并重定向客户端连接套接字到stdout，然后执行对应cgi文件，cgi文件打印html页面字符串，因为重定向的缘故，打印的html字符串发送给客户端，客户端浏览器即显示了html页面。

代码写了几个模块，分别是：

`util`：封装了套接字创建、unix族socket管道创建、中断信号、简单屏幕输出(可自行替换为日志文件输出)

`epoll_wrapper`：封装了epoll相关操作包括创建epfd、添加epoll监听事件、删除epoll监听事件

`myhshappool`(我的半同步半异步进程池 - -!…)：封装了进程池初始化、启动进程池进行事件监听

`client_handle`：进程池监听到客户事件、即调用client_handle封装的处理事件，这里封装的是执行cgi文件向客户端浏览器返回服务器时间(最近在看unix网络编程，里面都是时间获取的服务器，借鉴下拿来搞事，当然，嵌入式里拿来控制个灯泡开关想来特别带劲，用android做个网页app，板子接wifi模块接智能灯，cgi负责开关灯泡 。。)

`cgisrv`：入口，初始化进程池，启动进程池

代码快1k行，不知道一个博客文章能不能写下，这种玩具demo代码就不往github放了。代码中凑合写了注释(有时候不想切换中英文因此用了蹩脚的英文注释)，限(wo)于(tai)篇(lan)幅(le)没有写文件头注释和函数头注释。

`util.h`：

    #ifndef _UTIL_H
    #define _UTIL_H

    #include <stdio.h>
    #include <signal.h>
    #include <string.h>
    #include <errno.h>
    #include <sys/socket.h>
    #include <sys/types.h>
    #include <unistd.h>
    #include <arpa/inet.h>
    #include <stdlib.h>
    #include <fcntl.h>

    //这里不用 #转字符串了，不方便管理等级
    #ifdef LEVELNUMPRINT
        #define LEVEL0 "LEVEL0"
        #define LEVEL1 "LEVEL1"
        #define LEVEL2 "LEVEL2"
        #define LEVEL3 "LEVEL3"
        #define LEVEL4 "LEVEL4"
        #define LEVEL4 "LEVEL5"
    #else
        #define LEVEL0  "DEBUG"
        #define LEVEL1  "INFO"
        #define LEVEL2  "NOTICE"
        #define LEVEL3  "WARN"
        #define LEVEL4  "ERROR"
        #define LEVEL5  "FATAL"
    #endif

    #define PRINTINFO( LEVEL, format, args... ) \
    do { \
        printf("[%s]:(pid:%d)/(file:%s)/(func:%s)/(line:%d)", \
            LEVEL, getpid(), __FILE__, __func__, __LINE__); \
        /*printf("\t" #LEVEL ":");*/ \
        printf("--|--"); \
        printf( format, ##args ); \
        printf("\n"); \
    } while( 0 )

    #define PRINTINFO_ERR( LEVEL, format, args... ) \
    do { \
        printf("[%s]:(pid:%d)/(file:%s)/(func:%s)/(line:%d)", \
            LEVEL, getpid(), __FILE__, __func__, __LINE__); \
        /*printf("\t" #LEVEL ":");*/ \
        printf("--|--"); \
        printf( format, ##args ); \
        printf("(errmsg:%s)", strerror(errno)); \
        printf("\n"); \
    } while( 0 )


    void Add_sig( int sig, void (*handler)(int),
        int restart_syscall );

    void Socketpair( int *pairpipefd );

    int Socket_create( char *ipaddr, int port );

    void Setnonblocking( int fd );

    #endif

`util.c`:

    #include "util.h"


    static int add_sig( int sig, void (*handler)(int),
        int restart_syscall )
    {
        struct sigaction act;
        bzero( &act, sizeof(act) );

        act.sa_handler = handler;
        act.sa_flags = 0;

        //早期unix系统对于进程在执行一个低速系统调用(如ioctl、
        //read、write、wait)而阻塞期间捕捉到一个信号，则系统
        //调用被中断不再执行，该系统调用返回错误，设置errno为
        //EINTR，随后的bsd系统引入了自动重启，即再次进行此系统
        //调用。unix衍生系统默认的方式可能为可选、总是等，类
        //unix系统的linux系统可能默认为不重启，因此添加重启标识
        if ( restart_syscall ) {
            act.sa_flags |= SA_RESTART;
        }

        //宏定义：
        //#define sigfillset(*p) (*p) = ~(0,0)
        sigfillset( &act.sa_mask );

        if ( -1 == sigaction(sig, &act, NULL) ) {
            PRINTINFO_ERR( LEVEL4, "sigaction error" );
            return -1;
        }

        return 0;
    }
    void Add_sig( int sig, void (*handler)(int),
        int restart_syscall )
    {
        if ( add_sig(sig, handler, restart_syscall) < 0 ) {
            PRINTINFO( LEVEL5, "add_sig error" );
            exit( 0 );
        }
    }
    void Socketpair( int *pairpipefd )
    {
        int ret;

        ret = socketpair( PF_UNIX, SOCK_STREAM,
                0, pairpipefd );

        if ( ret < 0 ) {
            PRINTINFO_ERR( LEVEL5, "socketpair error!!" );
            exit( 0 );
        }
    }
    static int socket_create( char *ipaddr, int port,
        int backlog )
    {
        int sockfd;
        int ret;

        sockfd = socket( AF_INET, SOCK_STREAM, 0 );
        if ( sockfd < 0 ) {
            PRINTINFO_ERR( LEVEL4, "socket error!!!" );
            return -1;
        }

        struct sockaddr_in addr;
        bzero( &addr, sizeof(addr) ) ;

        addr.sin_family     = AF_INET;
        addr.sin_port       = htons( port );
        inet_pton( AF_INET, ipaddr, &addr.sin_addr );

        int reuseaddr = 1;
        setsockopt( sockfd, SOL_SOCKET, SO_REUSEADDR,
            &reuseaddr, sizeof(int) );

        ret = bind( sockfd, (struct sockaddr *)&addr, (socklen_t)sizeof(addr) ) ;
        if ( ret < 0 ) {
            PRINTINFO_ERR( LEVEL4, "bind error!!!" );
            return -1;
        }

        ret = listen( sockfd, backlog );
        if ( ret < 0 ) {
            PRINTINFO_ERR( LEVEL4, "listen error!!!" );
            return -1;
        }

        return sockfd;
    }
    int Socket_create( char *ipaddr, int port )
    {
        int ret;

        ret = socket_create( ipaddr, port, 5 );
        if ( ret < 0 ) {
            PRINTINFO( LEVEL5, "socket_creaet error!!!" );
            exit( 0 );
        }

        return ret;
    }
    static int setnonblocking( int fd )
    {
        int old_opt = fcntl( fd, F_GETFL );
        int new_opt = old_opt | O_NONBLOCK;

        fcntl( fd, F_SETFL, new_opt );

        return old_opt;
    }
    void Setnonblocking( int fd )
    {
        setnonblocking( fd );
    }
    int Send ( int socket_fd, const unsigned char * send_buf,
        int buf_size, int flag )
    {
        int snd_bytes = 0;
        int snd_total_bytes = 0;
        int snd_count = 3;

        while ( snd_count -- ) {
            snd_bytes = send( socket_fd, send_buf, buf_size, flag );
            if ( snd_bytes <= 0 ) {
                if ( EAGAIN == errno || EINTR == errno
                  || EWOULDBLOCK == errno ) { //暂时发送失败，需要重复发送
                    usleep( 50 );
                    continue;
                }else {  //连接不正常，返回-1交由上层清理此套接字
                    PRINTINFO_ERR( LEVEL4, "send return error!!!" );
                    return -1;
                }
            }
            snd_total_bytes += snd_bytes;
            if ( snd_total_bytes >= buf_size ) {
                break;
            }
        }
        if ( !snd_count ) {
            PRINTINFO( LEVEL4, "send timeout!!!" );
            return -1;
        }
        return snd_total_bytes;
    }
    #if 0
    int main()
    {
        PRINTINFO( LEVEL0, "likun:%d", 123 );
        PRINTINFO( LEVEL1, "likun:" );
        //PRINTINFO( likun, "likun:" );

        return 0;
    }
    #endif

`epoll_wrapper.h`:

    #ifndef _EPOLL_WRAPPER_H
    #define _EPOLL_WRAPPER_H

    #include <sys/epoll.h>
    #include <stdlib.h>

    int Epoll_create( int size );

    int Epoll_wait( int epfd, struct epoll_event *events,
            int maxevents, int timeout );

    void Epoll_add_fd( int epfd, int fd );

    void Epoll_del_fd( int epfd, int fd );

    #endif

`epoll_wrapper.c`:

    #include "epoll_wrapper.h"
    #include "util.h"

    static int epoll_create0( int size )
    {
        int ret;
        ret = epoll_create( size );
        if ( ret <= 0 ) {
            PRINTINFO_ERR( LEVEL3, "epoll_create error!!!" );
            return -1;
        }

        return ret;
    }
    int Epoll_create( int size )
    {
        int ret;

        if ( (ret = epoll_create0(size)) < 0 ) {
            PRINTINFO( LEVEL5, "epoll_create0 error!!!" );
            exit( 0 );
        }
        return ret;
    }
    int Epoll_wait( int epfd, struct epoll_event *events,
            int maxevents, int timeout )
    {
        return epoll_wait( epfd, events, maxevents, timeout );
    }
    static int epoll_add_fd( int epfd, int fd )
    {
        struct epoll_event event;
        event.data.fd = fd;
        event.events  = EPOLLIN | EPOLLET;

        epoll_ctl( epfd, EPOLL_CTL_ADD, fd, &event );

        Setnonblocking( fd );

        return 0;
    }
    void Epoll_add_fd( int epfd, int fd )
    {
        epoll_add_fd( epfd, fd );
    }
    void Epoll_del_fd( int epfd, int fd )
    {
        epoll_ctl( epfd, EPOLL_CTL_DEL, fd, NULL );
    }

    #if 0
    int main()
    {}
    #endif

`myhshappool.h`:

    #ifndef _MY_HS_HA_P_POOL_H
    #define _MY_HS_HA_P_POOL_H

    #include <sys/types.h>
    #include <sys/socket.h>
    #include <netinet/in.h>
    #include <arpa/inet.h>
    #include <assert.h>
    #include <stdio.h>
    #include <unistd.h>
    #include <errno.h>
    #include <string.h>
    #include <fcntl.h>
    #include <stdlib.h>
    #include <sys/epoll.h>
    #include <signal.h>
    #include <sys/wait.h>
    #include <sys/stat.h>
    #include <libgen.h>

    typedef struct process {
        //当前进程id号
        pid_t   pid;
        //与父进程通信用的管道
        //0端父进程写
        //1端子进程读
        int     pipefd[2];
    } process;

    typedef struct processpool {
        //进程池最大进程数
        int     max_process_num;
        //每个进程最大处理客户数量
        int     max_user_num;
        //epoll最多处理的事件数
        int     max_epoll_event;
        //当前进程池进程总数
        int     cur_process_num;
        //当前子进程在进程池序号，从0开始
        int     index;
        //每个进程一个epoll内核事件表
        int     epollfd;
        //监听socket
        int     listenfd;
        //停止线程池
        int     stop;
        //进程池子进程管理
        struct process *sub_process;
    } processpool;

    int init_process_pool( processpool *ppool, int maxpnum,
        int maxunum, int maxeevent, int curpnum, int listenfd );

    void run( processpool *ppool );

    #endif

`myhshappool.c`:

    #include "myhshappool.h"
    #include "client_handle.h"
    #include "util.h"

    //用于信号中断时主进程通信，
    //统一处理事件，即将客户端连接
    //事件、信号事件都统一用epoll
    //监听处理，0端信号处理函数写，
    //1端进程读
    static int sig_pipefd[2];

    static void sig_handler( int sig )
    {
        //保存旧的errno，对后续的send不
        //进行错误判定，但send假如返回
        //失败会设置errno，信号中断调用
        //结束后影响进程其它模块判断
        int old_errno = errno;
        char signo     = (char)sig;
        send( sig_pipefd[0], (char *)&signo, 1, 0 );
        errno = old_errno;
    }
    static void init_signal( processpool *ppool )
    {
        Socketpair( sig_pipefd );

        Setnonblocking( sig_pipefd[0] );
        //Epoll_add_fd( ppool->epollfd, sig_pipefd[1] );

        Add_sig( SIGCHLD, sig_handler, 1 );
        Add_sig( SIGTERM, sig_handler, 1 );
        Add_sig( SIGINT,  sig_handler, 1 );
        Add_sig( SIGPIPE, SIG_IGN, 1 );
    }

    int init_process_pool( processpool *ppool, int maxpnum,
        int maxunum, int maxeevent, int curpnum, int listenfd )
    {
        if ( !ppool ) {
            PRINTINFO( LEVEL4, "ppool is null!!!" );
            return -1;
        }

        ppool->max_process_num  = maxpnum;
        ppool->max_user_num     = maxunum;
        ppool->max_epoll_event  = maxeevent;
        ppool->cur_process_num  = curpnum;
        ppool->listenfd         = listenfd;
        // ppool->epollfd          = Epoll_create( 5 );
        ppool->index            = -1;
        ppool->stop             = 0;

        ppool->sub_process =
            (process *)calloc( sizeof(process), curpnum );
        if ( !ppool->sub_process ) {
            PRINTINFO( LEVEL4, "sub_process calloc error!!!" );
            return -1;
        }

        int i = 0;
        int pid;

        //先模拟一下进程池创建之前的情况，假设终端
        //bash shell进程id为1000，运行此程序进程id
        //为1001，其父进程为1000，fork之后主进程不
        //变，子进程id为1002，其父进程为1001，因此
        //明白fork的过程，下面可以走一下进程池创建
        //的流程（条件均为以上假设）：
        //第一次fork:创建亲缘进程的管道，父进程1001
        //，子进程1002，其父进程为1001，子进程不再
        //执行for循环，且主进程与1002子进程有单独通
        //信的管道
        //第二次fork:创建亲缘进程的管道，父进程1001
        //，子进程1003，其父进程为1001，子进程不再
        //执行for循环，且主进程与1003子进程有单独通
        //信的管道
        //第三次fork .....1004.....
        //    ........
        //通过以上过程，可以看到for循环次数为创建的
        //子进程数量，且每个子进程可以单独与父进程
        //通信
        //
        //这里进程创建，没有脱离当前终端的会话，
        //我觉得可以setsid()来摆脱终端影响
        for ( ; i < curpnum; i++ ) {
            Socketpair( ppool->sub_process[i].pipefd );
            pid = fork();
            if ( pid > 0 ) { //parent fork
                close( ppool->sub_process[i].pipefd[1] );
                ppool->sub_process[i].pid = pid;
                Setnonblocking( ppool->sub_process[i].pipefd[0] );
                continue;
            } else if ( pid == 0 ) { //child
                ppool->index = i;
                PRINTINFO( LEVEL0, "child(%d):%d\tparent:%d", i + 1, getpid(), getppid() );
                close( ppool->sub_process[i].pipefd[0] );
                //每次只由父进程去创建进程
                break;
            }
            else {
                PRINTINFO( LEVEL5, "fork error!!!" );
                exit( 0 );
            }
        }
    }
    static int client_signal_handle( processpool *ppool,
        char *signals, int signals_num )
    {
        int i = 0;
        for ( ; i < signals_num; i++ ) {
            switch( signals[i] ) {
                case SIGCHLD:
                {
                    PRINTINFO( LEVEL0, "child receive a SIGCHLD signal" );
                    pid_t pid;
                    int stat;
                    //catch SIGCHLD
                    while ( (pid = waitpid(-1, &stat, WNOHANG)) > 0 ) {
                        continue;
                    }
                    break;
                }
                case SIGTERM:
                {
                    PRINTINFO( LEVEL0, "child receive a SIGTERM signal" );
                    ppool->stop = 1;
                    break;
                }
                case SIGINT:
                {
                    PRINTINFO( LEVEL0, "child receive a SIGINT signal" );
                    ppool->stop = 1;
                    break;
                }
                default:
                {
                    break;
                }
            }
        }
    }
    static void run_child( processpool *ppool )
    {
        if ( !ppool ) {
            PRINTINFO( LEVEL4, "ppool is null!!!" );
            return;
        }

        init_signal( ppool );

        ppool->epollfd          = Epoll_create( 5 );

        PRINTINFO( LEVEL0, "child cur process:%d", ppool->cur_process_num );
        int pipefd = ppool->sub_process[ppool->index].pipefd[1];

        Epoll_add_fd( ppool->epollfd, pipefd );
        Epoll_add_fd( ppool->epollfd, sig_pipefd[1] );

        struct epoll_event *events = (struct epoll_event *)
            calloc( sizeof(struct epoll_event), ppool->max_epoll_event );
        if ( !events ) {
            PRINTINFO( LEVEL5, "calloc error!!!" );
            goto _free_source;
        }

        struct client_param *cparam = NULL;
        cparam = (struct client_param *)
                    calloc( sizeof(struct client_param), ppool->max_user_num );
        if ( !cparam ) {
            PRINTINFO( LEVEL5, "calloc error!!!" );
            goto _free_source;
        }

        int event_num, event_fd;
        int i, j, ret, onebyte;

        while ( !ppool->stop ) {
            event_num = Epoll_wait( ppool->epollfd, events, ppool->max_epoll_event, -1);
            //PRINTINFO( LEVEL0, "child event num:%d", event_num );
            if ( (event_num < 0) && (errno != EINTR) ) {
                PRINTINFO_ERR( LEVEL4, "Epoll_wait error!!!" );
                ppool->stop = 1;
                break;
            }

            for ( i = 0; i < event_num; i++ ) {
                event_fd = events[i].data.fd;
                //parent process notify that there is a new client connect to.
                if ( event_fd == pipefd && events[i].events & EPOLLIN ) {
                    PRINTINFO( LEVEL0, "receive signal from parent there is a new client connection" );
                    ret = recv( event_fd, (char *)&onebyte, 1, 0 );
                    if ( ret <= 0 ) {
                        continue;
                    } else {
                        struct sockaddr_in clientaddr;
                        socklen_t addrlen = sizeof(clientaddr);
                        bzero( &clientaddr, addrlen );

                        int connfd = accept( ppool->listenfd,
                                        (struct sockaddr *)&clientaddr, &addrlen );
                        if ( connfd < 0 ) {
                            PRINTINFO_ERR( LEVEL3, "accept a new client error!!!" );
                            continue;
                        }
                        PRINTINFO( LEVEL0, "one client conntect(fd:%d)", connfd );
                        Epoll_add_fd( ppool->epollfd, connfd );
                        client_param_init( &cparam[connfd], connfd, &clientaddr );
                    }
                }
                //process catch a signal
                else if ( event_fd == sig_pipefd[1] && events[i].events & EPOLLIN ) {
                    int sig;
                    char signals[1024] = {0};
                    ret = recv( sig_pipefd[1], signals, sizeof(signals), 0 );
                    if ( ret <= 0 ) {
                        continue;
                    }
                    client_signal_handle( ppool, signals, ret );
                }
                //client socket fd has readable event,maybe a
                //request
                else if ( events[i].events & EPOLLIN ) {
                    client_handle( &cparam[event_fd] );
                }
                else {
                    continue;
                }
            }
        }
    _free_source:
        free( events );
        events = NULL;

        free( cparam );
        cparam = NULL;

        close( pipefd );
        close( ppool->epollfd );
    }
    static int parent_signal_handle( processpool *ppool,
        char *signals, int signals_num )
    {
        int i = 0, j = 0;

        for ( ; i < signals_num; i++ ) {
            switch( signals[i] ) {
                case SIGCHLD:
                {
                    PRINTINFO( LEVEL0, "parent receive SIGCHLD signal" );
                    pid_t pid;
                    int stat;
                    while ( (pid = waitpid(-1, &stat, WNOHANG)) > 0 ) {
                        PRINTINFO( LEVEL0, "parent receive SIGCHLD signal(pid:%d)", pid );
                        for ( j = 0; j < ppool->cur_process_num; j++ ) {
                            if ( ppool->sub_process[j].pid == pid ) {
                                PRINTINFO( LEVEL3, "child process(%d) exit.", pid );
                                close( ppool->sub_process[j].pipefd[1] );
                                ppool->sub_process[j].pid = -1;
                            }
                        }
                    }
                    ppool->stop = 1;
                    for ( j = 0; j < ppool->cur_process_num; j++ ) {
                        PRINTINFO( LEVEL0, "pid:%d", ppool->sub_process[j].pid );
                        if ( ppool->sub_process[j].pid != -1 ) {
                            ppool->stop = 0;
                            break;
                        }
                    }
                    break;
                }
                case SIGTERM:
                case SIGINT:
                {
                    PRINTINFO( LEVEL2, "recv SIGINT/SIGTERM, kill all child process now." );
                    //PRINTINFO( LEVEL0, "cur_process_num:%d", ppool->cur_process_num );
                    for ( i = 0; i < ppool->cur_process_num; i++ ) {
                        int pid = ppool->sub_process[i].pid;
                        if ( pid != -1 ) {
                            PRINTINFO( LEVEL0, "kill process:%d", pid );
                            ppool->sub_process[i].pid = -1;
                            kill( pid, SIGTERM );
                        }
                    }
                    ppool->stop = 1;
                    break;
                }
                default:
                {
                    break;
                }
            }
        }
    }
    static void run_parent( processpool *ppool )
    {
        if ( !ppool ) {
            PRINTINFO( LEVEL4, "ppool is null!!!" );
            return;
        }

        init_signal( ppool );

        ppool->epollfd          = Epoll_create( 5 );

        Epoll_add_fd( ppool->epollfd, ppool->listenfd );
        Epoll_add_fd( ppool->epollfd, sig_pipefd[1] );

        struct epoll_event *events = NULL;
        events = (struct epoll_event *)
            calloc( sizeof(struct epoll_event), ppool->max_epoll_event );
        if ( !events ) {
            PRINTINFO( LEVEL5, "calloc error!!!" );
            char sig = SIGINT;
            parent_signal_handle( ppool, &sig, 1 );
            goto _free_source;
        }

        int event_num;
        int i, onebyte = 1, ret, j;
        //p_idx specifies current dispatched child process.
        //roll_index specifies the next child process.
        int p_idx , roll_index = 0;

        //PRINTINFO( LEVEL0, "epollfd:%d", ppool->epollfd );

        while ( !ppool->stop ) {

            //Specifying  a timeout of -1 causes epoll_wait()
            //to block indefinitely.
            event_num =
                Epoll_wait( ppool->epollfd, events, ppool->max_epoll_event, -1 );

            if ( event_num < 0 && errno != EINTR ) {
                PRINTINFO_ERR( LEVEL5, "Epoll_wait error!!!" );
                ppool->stop = 1;
                break;
            }
            //PRINTINFO( LEVEL0, "event_num:%d", event_num );
            for ( i = 0; i < event_num; i++ ) {
                int event_fd = events[i].data.fd;

                //listenfd,there is a new client connection.
                //notify child process to accept
                if ( event_fd == ppool->listenfd ) {
                    PRINTINFO( LEVEL0, "event:parent listenfd" );
                    //round robin dispatch
                    //easily roll polling
                    p_idx = roll_index;
                    do {
                        if ( ppool->sub_process[p_idx].pid != -1 ) {
                            break;
                        }
                        p_idx = ( p_idx + 1 ) % ppool->cur_process_num;
                    } while ( p_idx != roll_index );

                    //roll polling all the child process,but they are
                    //all run error.so p_idx equals to roll_index.
                    if ( ppool->sub_process[p_idx].pid < 0 ) {
                        ppool->stop = 1;
                        break;
                    }

                    roll_index = ( p_idx + 1 ) % ppool->cur_process_num;

                    if ( Send( ppool->sub_process[p_idx].pipefd[0],
                            (char *)&onebyte, 1, 0 ) < 0 ) {
                        PRINTINFO( LEVEL5, "Send error!!!" );
                        ppool->stop = 1;
                        break;
                    }
                }
                //receive signal from signal handler.
                else if ( (event_fd == sig_pipefd[1])
                    && (events[i].events & EPOLLIN) ) {
                    PRINTINFO( LEVEL0, "event:parent receive signal" );
                    int sig;
                    char signals[1024];
                    ret = recv( sig_pipefd[1], signals, sizeof(signals), 0 );
                    if ( ret <= 0 ) {
                        continue;
                    } else {
                        parent_signal_handle( ppool, signals, ret );
                    }
                }
            }
        }
    _free_source:
        free( events );
        events = NULL;

        close( ppool->epollfd );
    }
    void run( processpool *ppool )
    {
        if ( ppool->index != -1 ) {
            run_child( ppool );
            return;
        }
        run_parent( ppool );
    }


`client_handle.h`:

    #ifndef _CLIENT_HANDLE_H
    #define _CLIENT_HANDLE_H

    #include <stdio.h>
    #include <unistd.h>
    #include <fcntl.h>
    #include <string.h>
    #include <stdlib.h>
    #include <sys/socket.h>
    #include <sys/types.h>
    #include <arpa/inet.h>

    #define MAXRECVBUF  1024

    typedef struct client_param {
        //用于recv返回出错移除sockfd
        int epollfd;
        int sockfd;
        struct sockaddr_in addr;
        char buf[ MAXRECVBUF ];
    } client_param;

    void client_param_init( client_param *cparam,
        int connfd, struct sockaddr_in *addr );

    void client_handle( client_param *cparam );

    #endif

`client_handle.c`:

    #include "client_handle.h"
    #include "util.h"

    void client_param_init( client_param *cparam,
        int connfd, struct sockaddr_in *addr )
    {
        if ( !cparam ) {
            PRINTINFO( LEVEL4, "cparam is null!!!" );
            return;
        }
        cparam->sockfd = connfd;
        memcpy( &cparam->addr, addr, sizeof(struct sockaddr_in) );
        memset( cparam->buf, 0, sizeof(cparam->buf) );
    }
    void client_handle( client_param *cparam )
    {
        int i, ret;

        while ( 1 ) {
            memset( cparam->buf, 0, sizeof(cparam->buf) );
            ret = recv( cparam->sockfd, cparam->buf, sizeof(cparam->buf), 0 );
            if ( ret < 0 ) {
                if ( errno != EAGAIN && errno != EWOULDBLOCK
                    && errno != EINTR ) {
                    Epoll_del_fd( cparam->epollfd, cparam->sockfd );
                }
                close( cparam->sockfd );
                break;
            }
            else if ( 0 == ret ) {
                Epoll_del_fd( cparam->epollfd, cparam->sockfd );
                close( cparam->sockfd );
                break;
            }
            else {
                if ( ret < 15 ) {
                    close( cparam->sockfd );
                    break;
                }
                //PRINTINFO( LEVEL0, "child receive buf:\n%s", cparam->buf );
                fflush(stdout);
                char *p_get = strstr( cparam->buf, "GET" );
                if ( !p_get ) {
                    close( cparam->sockfd );
                    break;
                }
                char *p_http = strstr( cparam->buf, "HTTP" );
                if ( !p_http ) {
                    close( cparam->sockfd );
                    break;
                }

                cparam->buf[ret] = '\0';

                //GET filename HTTP/1.1 .....
                char file_name[20] = {0};
                int file_name_len = p_http - p_get - 6;
                memcpy( file_name, p_get + 5, file_name_len );

                if ( access( file_name, F_OK ) == -1 ) {
                    PRINTINFO( LEVEL3, "file:(%s) dosen't exist!!", file_name );
                    Epoll_del_fd( cparam->epollfd, cparam->sockfd );
                    close( cparam->sockfd );
                    break;
                }
                PRINTINFO( LEVEL0, "file name:%s--", file_name );
                ret = fork();
                if ( ret == -1 ) {
                    Epoll_del_fd( cparam->epollfd, cparam->sockfd );
                    close( cparam->sockfd );
                    break;
                }
                else if ( ret > 0 ) {
                    Epoll_del_fd( cparam->epollfd, cparam->sockfd );
                    close( cparam->sockfd );
                    break;
                }
                else {
                    close( STDOUT_FILENO );
                    //relocate the stdou to sockfd
                    PRINTINFO( LEVEL0, "sockfd:%d", cparam->sockfd );
                    dup( cparam->sockfd );
                    //printf("likun\n");
                    execl( file_name, file_name, NULL );
                    fflush(stdout);
                    Epoll_del_fd( cparam->epollfd, cparam->sockfd );
                    close( cparam->sockfd );
                    exit( 0 );
                }
            }
        }
    }

`cgisrv.c`:

    #include "myhshappool.h"
    #include "client_handle.h"
    #include "util.h"
    #include "epoll_wrapper.h"

    //进程池最大数量
    #define MAXPROCESSNUMBER            16
    //每个进程支持客户连接任务
    #define USERPERPROCESS              65535
    //epool最大支持的监听事件数
    #define MAXEPOLLEVENT               10000

    processpool ppool;

    int main()
    {
        int listenfd = Socket_create( "192.168.1.250", 8888 );

        init_process_pool( &ppool, MAXPROCESSNUMBER,
            USERPERPROCESS, MAXEPOLLEVENT, 5, listenfd );

        run( &ppool );

        close( listenfd );

        return 0;
    }

贴一下makefile：

    CC = gcc
    ROOTDIR = $(shell pwd)
    OBJ = util.o myhshappool.o epoll_wrapper.o \
    client_handle.o cgisrc.o
    BIN = cgisrv.bin
    CFLAG = -Wall -O2 -I./
    LDFLAG += -c

    $(BIN):${OBJ}
        $(CC) $(CFLAG) -o $@ $^

    %:%.c
        $(CC) $(CFLAG) -o $@ $< $(LDFLAG)

    .PHONY:clean
    clean:
        rm $(OBJ) $(BIN) -rf

还有cgi执行程序：

    #include <stdio.h>
    #include <time.h>
    #include <string.h>
    /*
    <!DOCTYPE html>
    <html>
        <head>
            <meta charset="utf-8">
            <title></title>
        </head>
        <body>
            <h1><u><font color=00ff00>..</font></u></h1>
        </body>
    </html>
    */
    int main( int argc, char **argv )
    {
        time_t tt = time(NULL);

        printf("<!DOCTYPE html>");
        printf("<html>");
        printf("<head>");
        printf("<meta charset=\"utf-8\">");
        printf("<title>get now time</title>");
        printf("</head>");
        printf("<body>");
        printf("<h1><u><font color=00ff00>");
        printf("当前服务器时间:%s", ctime(&tt));
        printf("</font></u></h1>");
        printf("</body>");
        printf("</html>");

        return 0;
    }


测试：代码编译了，即可执行cgisrv.bin，主进程处于监听客户端连接情况，打开浏览器输入: `http://xxx.xxx.xxx.xxx:8888/now.cgi，可以看到出现一行加下划线的绿字：当前服务器时间:Sat Sep 17 21:49:07 2016`。

ip、端口、进程池数、最大epoll监听事件数等在入口模块(cgisrv)可以改。

本例子写完，调试了几个地方，运行几个客户端发送get请求就没有做测试了，练手的玩具demo ^_^。