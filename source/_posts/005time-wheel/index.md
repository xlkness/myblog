---
title: C时间轮
date: 2018-06-15 15:41:11
categories:
- 开发
tags:
- C/C++
---

看完了《linux高性能服务器编程》对里面的定时器很感兴趣。书中提到三种定时器，分别是：基于升序链表的定时器，基于时间轮的定时器，基于时间堆的定时器。三种定时器的实现书中均是给了C++代码，不过我对C++不太感兴趣，虽然现在在做C++开发，因此写了C版本的。书中定时器只给了封装的定时器类，没有给调用层代码，我是估摸着写了调用层代码。这里做个总结，以后可以翻翻：

基于升序链表的定时器没太大难度，因此也懒得总结了。

说一下时间轮，下面是截的书中的图片：

![time_wheel]

时间轮，像轮子一样滚动定时，每滚一个刻度，指针就走一个滴答，滚完一圈，就进入下一圈。因此有了这个概念，时间轮的结构也就出来了：1.齿轮（槽slot），用来标识一个滴答；2.槽间隔（slot interval ），当前槽经过多长时间到下一个槽；3.一圈的槽数量（N）；4.当前指针，走一个滴答加一，走完一圈又回到初始位置。

再深入一点，定时器以什么方式添加到槽上？可以看图，每一个槽其实就是一个链表头结点，定时器即添加到所属槽的链表后。这样我们可以对时间轮性能进行分析，SI越小，定时精度越高，如果SI=10s，那么我们指定的定时器只能是10s的倍数；如果N越大，定时器效率越高，这也很好理解，N越小，一圈槽数量越少，那么我们同样添加100个定时器，分配到每个头结点的定时器越多，每一次滴答到时，就遍历当前槽，遍历一次所花时间越多。

如何确定定时器位置？根据定时器到时时间可以计算，例如：定时器超时时间timeout=21s(即21s后触发定时器)，当前间隔SI=2s，一圈槽数量N=70，当前指针cur_slot指向第5个槽，我们可以计算出定时器放置的位置，这里需要两个变量，一个rotation指定定时器处于第几圈，一个slot指定定时器处于第几个槽，因此slot = ( cur_slot + timeout / SI ) % N = 15, rotation = timeout / SI / N = 0，即此定时器被放置于15槽的链表后，至于是链表头插还是尾插这个随意，指针滴答到了15槽即触发15槽到时，遍历15槽链表，若rotation=0的表示为当前该触发定时器，若rotation>0的定时器对rotation–（其实很好理解，cur_slot在转当前轮，则不处理后面的轮，只对它的rotation减一就跳过，等到cur_slot转下一圈再判断此定时器）。根据这个计算，如果其它参数不变，现在有一个timeout=161s的定时器，cur_slot=5,我们可以计算出这个定时器的slot=15，rotation=1，正好处于第15槽，但是是下一转触发该触发。

也就是说，如果我们根据以上参数，同时添加一个15s和一个161s定时器，他们都会随时间轮轮转触发到，只不过指针第一次只想15槽时，判断15s的定时器rotation为0，则触发定时器，然后删除定时器，遍历到161s定时器时，rotation=1，执行减1，跳过继续轮转，当cur_slot=70的时候也就是时间轮走过65*2=130s时，时间轮转一圈，cur_slot=0，继续下一圈开始，再走过14*2=28s后，到达15槽，判断161s定时器，rotation=0，触发定时器。

有了这些分析，下面直接贴代码：

    #include <stdio.h>
    #include <string.h>
    #include <time.h>
    #include <unistd.h>
    #include <signal.h>
    #include <stdlib.h>

    typedef struct client_data {
        int fd;
        time_t tt;
        char buf[512];
        void* data;
    }client_data;
    typedef struct tw_timer {
        //处于时间轮第几转，即时间轮转多少转
        //此定时器可以处于当前转，若再加上槽
        //即可确定此定时器所处时间轮位置
        int rotation;

        //处于当前时间轮转的第几个槽
        int slot;

        //定时器到时执行的回调函数
        void* (*cb_func)( void* param );

        //用户数据，触发回调任务函数的参数
        struct client_data c_data;

        //这里只需要单向不循环链表即可
        //struct tw_timer* prev;
        struct tw_timer* next;
    }tw_timer;
    typedef struct timer_manager {
        //时间轮当前槽，每经过一个间隔时间，加一实现轮转动，
        //超过总槽数即归零表示当前轮转完
        int cur_slot;

        //时间轮一转的总槽数，总槽数越大槽链表越短，效率越高
        int slot_num_r;

        //相邻时间槽间隔时间，即时间轮转到下一个槽需要时间，
        //间隔时间越短，精度越高，例如10s，表示定时器支持10s
        //间隔定时器添加，最小支持1s
        int slot_interval;

        //每个时间槽链表头结点，即一个槽管理一条链表，链表
        //添加相同槽数的结点，但转数可能不同
        struct tw_timer* slots_head[512];
    }timer_manager;

    timer_manager tmanager;

    void* ontime_func( void* param )
    {
        client_data* data = (client_data*)param;
        time_t tt = time(NULL);
        printf("\n----------------------------------------------------\n");
        printf("\tontime,interval:%d\n", (int)(tt - data->tt));
        printf("\told time:%s", ctime(&data->tt));
        printf("\t%s", data->buf);
        printf("\tcur time:%s", ctime(&tt));
        //getchar();
        printf("----------------------------------------------------\n");

        return NULL;
    }
    int add_timer( timer_manager* tmanager,
        int timeout, client_data* c_data )
    {
        if ( timeout < 0 || !tmanager )
            return -1;

        int tick = 0;           //转动几个槽触发
        int rotation = 0;       //处于时间轮第几转
        int slot = 0;           //距离当前槽相差几个槽

        if ( timeout < tmanager->slot_interval )
            tick = 0;
        else
            tick = timeout / tmanager->slot_interval;

        rotation = tick / tmanager->slot_num_r;
        slot = ( tmanager->cur_slot + tick % tmanager->slot_num_r )
                    % tmanager->slot_num_r - 1;

        printf("addtimer-->timeout:%d, rotation:%d,slot:%d\n",
            timeout, rotation, slot);

        tw_timer* tmp_t = (tw_timer*)malloc(sizeof(tw_timer));
        tmp_t->rotation = rotation;

        char buf[100] = {0};
        time_t tt = time(NULL) + timeout;

        sprintf( buf, "set time:%s", ctime(&tt));
        memset( tmp_t->c_data.buf, 0, sizeof(tmp_t->c_data.buf));
        strcpy( tmp_t->c_data.buf, buf );
        tmp_t->slot = slot;
        tmp_t->c_data.tt = time(NULL);
        tmp_t->cb_func = ontime_func;

        if ( !tmanager->slots_head[slot] )
        {
            tmanager->slots_head[slot] = tmp_t;
            tmp_t->next = NULL;
            //printf("[line]:%d\n", __LINE__);
            return 0;
        }
        //printf("[line]:%d\n", __LINE__);
        tmp_t->next = tmanager->slots_head[slot]->next;
        tmanager->slots_head[slot]->next = tmp_t;

        return 0;
    }
    int del_all_timer( timer_manager* tmanager )
    {
        //清除、释放所有定时器，懒得写了
    }
    int tick( timer_manager* tmanager )
    {
        if ( !tmanager )
            return -1;

        tw_timer* tmp = tmanager->slots_head[tmanager->cur_slot];
        tw_timer* p_tmp;

        while ( tmp )
        {
            //rotation减一，当前时间轮转不起作用
            //假设这个tmp指向第0个槽的头，链中某个结点的rotaion为下一圈，
            //即rotation=1，所以这个定时器不起作用，而因为cur_slot不断
            //走动，tmp在当前转不可能再指向这个定时器，下一圈cur_slot
            //为0时能继续判断这个定时器，故实现了定时器处于不同转的判断
            if ( tmp->rotation > 0 )
            {
                tmp->rotation--;
                p_tmp = tmp;
                tmp = tmp->next;
            }
            else
            {
                //否则定时器到时，触发回调函数
                tmp->cb_func( &tmp->c_data );

                //删除此定时器结点
                //吃了没用双向链表的亏，写这么low
                if ( tmp == tmanager->slots_head[tmanager->cur_slot] )
                {
                    //printf("[line]:%d\n", __LINE__);
                    tmanager->slots_head[tmanager->cur_slot] = tmp->next;
                    p_tmp = tmp;
                    tmp = tmp->next;
                    free( p_tmp );
                    p_tmp = NULL;
                    p_tmp = tmp;
                    //printf("[line]:%d\n", __LINE__);
                }
                else
                {
                    p_tmp->next = p_tmp->next->next;
                    free( tmp );
                    tmp = NULL;
                    tmp = p_tmp->next;
                }
            }
        }
        //更新时间轮，转动一个槽，转一圈又从开始转
        tmanager->cur_slot = ++tmanager->cur_slot % tmanager->slot_num_r;

        return 0;
    }
    int init_t_manager( timer_manager* tmanager,
        int slot_num_r, int slot_interval )
    {
        tmanager->cur_slot = 0;
        tmanager->slot_num_r = slot_num_r;
        tmanager->slot_interval = slot_interval;

        return 0;
    }
    //自己试着写的调用层代码
    void alarm_handler( int sig )
    {
        time_t tt = time(NULL);
        //printf("timer tick:%s", ctime(&tt));

        int ret = tick( &tmanager );
        if ( ret < 0 )
            printf("tick error\n");

        alarm( tmanager.slot_interval );
    }
    int main()
    {
        time_t tt = time(NULL);

        signal( SIGALRM, alarm_handler );

        //init_t_manager( &tmanager, 60, 10 );
        init_t_manager( &tmanager, 60, 1 );

        add_timer( &tmanager, 6, NULL );
        add_timer( &tmanager, 11, NULL );
        add_timer( &tmanager, 22, NULL );
        add_timer( &tmanager, 33, NULL );
        add_timer( &tmanager, 44, NULL );
        add_timer( &tmanager, 55, NULL );
        add_timer( &tmanager, 66, NULL );
        add_timer( &tmanager, 77, NULL );
        add_timer( &tmanager, 88, NULL );
        add_timer( &tmanager, 99, NULL );
        add_timer( &tmanager, 111, NULL );
        add_timer( &tmanager, 122, NULL );
        add_timer( &tmanager, 133, NULL );
        add_timer( &tmanager, 144, NULL );

        printf("start time:%s\n", ctime(&tt));
        alarm( tmanager.slot_interval );

        while ( 1 )
            sleep( 5 );

        return 0;
    }

看以上代码，main函数开始即指定了SI=1s，N=60，并添加了很多定时器，然后开始以SI执行定时，每一次到时就触发滴答函数tick()，如此循环定时触发到时信号就实现了时间轮轮转。

关于代码的思考：这里用了SIGALRM信号，每一次到时，主线程暂停，去执行信号函数内容，如果信号SIGALRM的处理函数太庞大，会影响主线程的任务卡顿，虽然以上代码执行量不大，但为了扩展，我觉得可以将定时器触发执行的操作改为添加任务结点到任务链，这样配合线程池效率会高一点，线程池本身会从任务链取任务结点执行，如果我们的定时处理函数只是往任务链放任务，那性能会高很多，而不是往cb_func里执行具体业务逻辑。

下一篇上时间堆。

[time_wheel]:1.jpeg "time_wheel"