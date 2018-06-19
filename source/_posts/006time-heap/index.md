---
title: C时间堆
date: 2018-06-15 15:46:30
categories:
- 开发
tags:
- C/C++
---


跟上一篇，这里写一下时间堆：

时间轮的滴答是固定以指定的槽间隔触发，而时间堆是以定时器堆中的最小到期时间做定时，也就是alarm(minTimeout)，一旦定时器被触发，那么就删除此定时器，更新时间堆，将新的最小到期时间定时器作为新的定时。

然后是时间堆的数据结构，这里用了二叉树，将二叉树的根节点作为最小最小到期时间，每次滴答就取根节点，并且是完全二叉树，因此比较容易理解，我也懒得画图了，截了书上的图：


![time_heap]


从图中可以看到，对时间堆的关键操作其实就是上滤和下滤，上滤也就是简单的将完全二叉树待添加的定时器最为最后一个节点并与父节点比较，如果小于，则交换两节点位置，再将此节点与父节点比较，直到待插入节点值大于父节点值或者已经是根节点即停止；下滤就是触发一个定时器之后删除此定时器，然后取完全二叉树的深度最大一排的最后一个叶子节点作为根节点，然后用此根节点与两个儿子节点比较，若根节点值大于儿子节点中最大者，则交换这两个节点，再以这个节点做同样的下滤，如果对于任意完全二叉树，这样简单的下滤不能找出最小节点，但是配合定时器添加时的上滤操作后的完全二叉树，这样就可行。

下面贴自己改的代码：

    #include <stdio.h>
    #include <stdlib.h>
    #include <unistd.h>
    #include <string.h>
    #include <signal.h>
    #include <time.h>

    typedef struct client_data {
        char buf[512];
        time_t tt;
        void* data;
    } client_data;

    typedef struct heap_timer {
        //单个定时器设置绝对时间
        time_t expire;
        //加的设置的定时器到时秒数
        int timeout;
        //到时回调函数
        void (*cb_func)( void* );
        //回调函数参数
        struct client_data* user_data;
    } heap_timer ;

    typedef struct h_t_manager {
        //定时器指针数组
        struct heap_timer** array;
        //当前定时管理器支持的最大定时器个数
        int capacity;
        //当前定时管理器上的定时器个数
        int cur_size;
    } h_t_manager ;
    void cb_func( void* param )
    {
        time_t tt = time(NULL);
        printf("\n-----------------------------------------------\n");
        printf("\tontime,cur time - old time:%d\n", (int)(tt - ((client_data*)param)->tt));
        printf("\toldtime:%s", ctime(&(((client_data*)param)->tt)));
        printf("\tcurtime:%s", ctime(&tt));
        printf("-----------------------------------------------\n");
    }
    static void percolate_down( h_t_manager* tmanager, int hole )
    {
        heap_timer* temp = tmanager->array[hole];
        int flag = 0;
        int child = 0;
        for ( ; ((hole*2+1) <= (tmanager->cur_size-1)); hole = child )
        {
            flag = 0;
            child = hole * 2 + 1;

            //这里找出当前节点最小儿子节点
            if ( (child < (tmanager->cur_size-1))
                && (tmanager->array[child+1]->expire)
                    < tmanager->array[child]->expire )
            {
                ++child;
            }
            //比较待删除节点和最小儿子节点，若大于就交换
            if ( tmanager->array[child]->expire < temp->expire )
            {
                //这里的交换其实该用内存拷贝比较好
                int tmp_expire = tmanager->array[hole]->expire;
                int tmp_timeout = tmanager->array[hole]->timeout;
                tmanager->array[hole]->expire = tmanager->array[child]->expire;
                tmanager->array[hole]->timeout = tmanager->array[child]->timeout;
                tmanager->array[child]->expire = tmp_expire;
                tmanager->array[child]->timeout = tmp_timeout;
            }
            else
            {
                break;
            }
            //新的下滤比较
            temp = tmanager->array[child];
            //tmanager->array[hole] = temp;
        }
    }
    //将堆数组的容量扩大1倍
    static int resize( h_t_manager* tmanager )
    {
        heap_timer** temp = (heap_timer**)
                malloc( 2 * tmanager->capacity * sizeof(h_t_manager) );

        int i = 0;
        for ( ; i < 2 * tmanager->capacity; ++i )
        {
            temp[i] = NULL;
        }
        if ( !temp )
        {
            return -1;
        }
        tmanager->capacity = 2 * tmanager->capacity;
        for ( i = 0; i < tmanager->cur_size; ++i )
        {
            temp[i] = tmanager->array[i];
            free(tmanager->array[i]);
            tmanager->array[i] = NULL;
        }
        tmanager->array = temp;

        return 0;
    }
    int init_ht_manager_0( h_t_manager* tmanager, int cap )
    {
        tmanager->capacity = cap;
        tmanager->cur_size = 0;
        tmanager->array = (heap_timer**)malloc(cap * sizeof(h_t_manager));

        int i = 0;
        for ( ; i < cap; ++i )
        {
            tmanager->array[i] = NULL;
        }
    }
    //添加定时器
    int add_timer( h_t_manager* tmanager, int timeout )
    {
        if ( !tmanager || timeout <= 0)
        {
            return -1;
        }
        if ( tmanager->cur_size >= tmanager->capacity )
        {
            resize( tmanager );
        }
        int hole = tmanager->cur_size++;
        int parent = 0;
        heap_timer* timer = (heap_timer*)malloc(sizeof(heap_timer));
        time_t tt = time(NULL);
        timer->expire = (int)tt + timeout;
        timer->timeout = timeout;
        timer->user_data = (client_data*)malloc(sizeof(client_data));
        timer->user_data->tt = time(NULL);
        timer->cb_func = cb_func;
        for ( ; hole > 0; hole = parent )
        {
            parent = ( hole - 1 ) / 2;
            if ( tmanager->array[parent]->expire <= timer->expire )
            {
                break;
            }
            tmanager->array[hole] = tmanager->array[parent];
        }
        tmanager->array[hole] = timer;

        return 0;
    }
    int del_timer( h_t_manager* tmanager, heap_timer* timer )
    {
        if ( !tmanager || !timer )
        {
            return -1;
        }
        timer->cb_func = NULL;
    }
    int empty( h_t_manager* tmanager )
    {
        return tmanager->cur_size == 0;
    }
    heap_timer* top( h_t_manager* tmanager )
    {
        if ( empty(tmanager) )
        {
            printf("!!!!!!!top->empty cur size\n");
            return NULL;
        }
        return tmanager->array[0];
    }
    int pop_timer( h_t_manager* tmanager )
    {
        if ( empty( tmanager ) )
        {
            printf("!!!!!!!pop_timer->empty cur size\n");
            return -1;
        }
        if ( tmanager->array[0] )
        {
            free( tmanager->array[0] );
            tmanager->array[0] = NULL;
            tmanager->array[0] = tmanager->array[--tmanager->cur_size];
            percolate_down( tmanager, 0 );
        }
        return 0;
    }
    void tick( h_t_manager* tmanager )
    {
        heap_timer* tmp = tmanager->array[0];
        time_t cur = time( NULL );
        while ( !empty(tmanager) )
        {
            if ( !tmp )
            {
                break;
            }
            if ( tmp->expire > cur )
            {
                break;
            }
            if ( tmanager->array[0]->cb_func )
            {
                printf("timer on time,heap:");
                int i = 0;
                for ( ; i < 10; i++ )
                {
                    if ( tmanager->array[i] )
                        printf("%d:%d ", i, tmanager->array[i]->timeout);
                }

                tmanager->array[0]->cb_func( tmanager->array[0]->user_data );
            }
            pop_timer( tmanager );
            tmp = tmanager->array[0];
            printf("after timer on time,heap:");
            int i = 0;
            for ( ; i < 10; i++ )
            {
                if ( tmanager->array[i] )
                    printf("%d:%d ", i, tmanager->array[i]->timeout);
            }
            printf("\n");
            if ( tmanager->array[0] )
                printf("the next alarm is:%d\n", (int)(tmanager->array[0]->timeout));
            printf("current timer count:%d\n", tmanager->cur_size);
        }
    }
    h_t_manager tmanager;
    void alarm_handler( int sig )
    {

        tick( &tmanager );

        alarm( tmanager.array[0]->expire - time(NULL) );
    }
    int main()
    {
        printf("start timer 。。。。。。。\n");

        init_ht_manager_0( &tmanager, 20 );

        add_timer( &tmanager, 11 );
        add_timer( &tmanager, 5 );
        add_timer( &tmanager, 11 );
        // add_timer( &tmanager, 22 );
        add_timer( &tmanager, 6 );
        add_timer( &tmanager, 9 );
        // add_timer( &tmanager, 23 );
        // add_timer( &tmanager, 33 );
        // add_timer( &tmanager, 28 );
        // add_timer( &tmanager, 1 );
        // add_timer( &tmanager, 66 );
        // add_timer( &tmanager, 77 );
        // add_timer( &tmanager, 88 );
        // add_timer( &tmanager, 55 );

        signal( SIGALRM, alarm_handler );

        time_t tt = time(NULL);
        printf("current time:%s", ctime(&tt));
        alarm( tmanager.array[0]->expire - time(NULL) );

        while(1)
            sleep(5);

        return 0;
    }

因为用的完全二叉树，比较简单，跟上一篇时间轮一样，存在注意某些使用问题，整理的时候发现时间堆代码的注释还没写，想着自己加一点，发现很久前写的代码了，有点忘了，加了点也困了，打完羽毛球回来又连撸两篇博客 ，还被蚊子咬，一挠还挠死个咬得比我写博客还专注的蚊子 ，醉了 ~


[time_heap]:/img/time_heap.jpeg "time_heap"

