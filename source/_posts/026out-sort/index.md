---
title: 外排序多路归并+败者树-算法学习笔记十五
date: 2018-06-20 12:40:53
categories:
- 开发
tags:
- 数据结构/算法
---

问题：一个文件有大量的数，现要对文件排序，但内存无法一次读取完全，而磁盘空间足够，要如何排序。

学习了几篇博客：
1. july[大神的海量数据排序][1](他的其他博客都很值得看)
2. 对july大神的算法进行改进不用选择法而是败者树的[博客][2]
3. 以及另一篇但不知道是否为原创的[博客][3]
4. 还有[生成不重复乱序m-n的数的博客][4](先生成m-n的数，然后洗牌算法)

#
以上几篇博客写得很完全了，看懂了思路，自己临摹写一个简单的测试 ….
**先用生成随机数的代码生成data.txt待排序大文件:**

    //生成随机的不重复的测试数据
    #include <iostream>
    #include <stdio.h>
    #include <time.h>
    #include <assert.h>
    #include <stdlib.h>  // RAND_MAX
    using namespace std;
    //产生[i,u]区间的随机数
    int randint(int l, int u)
    {
        int a = RAND_MAX * rand();
        int b = rand();
        //取低31位
        int c = ( a + b ) & (0x7fffffff) % ( u - l + 1 );
        int d = l + c;
        return d;
    }

    const int size = 10000000;
    // const int size = 10;
    int num[size];
    int main()
    {
        srand((int)time(NULL));
        int i, j;
        FILE *fp = fopen("data.txt", "w");
        assert(fp);
        for (i = 0; i < size; i++)
            num[i] = i+1;
        // printf("rand_max:%d\n", RAND_MAX);
        for (i = 0; i < size; i++)
        {
            j = randint(i, size-1);
            // printf("%d ", j);
            fflush(stdout);
            int t = num[i]; num[i] = num[j]; num[j] = t;
            //swap(num[i], num[j]);
        }
        // printf("\n");
        for (i = 0; i < size; i++)
            fprintf(fp, "%d\n", num[i]);
        fclose(fp);
        return 0;
    }

**对data.txt文件开始外排序:**

    #include <stdio.h>
    #include <string.h>
    #include <stdlib.h>
    #include <unistd.h>
    #include <sys/types.h>
    #include <sys/stat.h>
    #include <unistd.h>
    #include <errno.h>
    #include <time.h>

    #define TEMP_PREFIX "ftemp_"
    #define OUTPUT_FILE "out_data.txt"
    #define MAX_WAYS  100
    // 无穷大，用于某一路文件或缓冲区读到尾了
    // 败者树产生一个注定失败的结点
    #define INFINITY 1000000000

    int *buf;
    int lst[MAX_WAYS];

    void read_data(FILE *fp, int *buf)
    {
        if ( fscanf(fp, "%d ", buf) == EOF )
            *buf = INFINITY;
    }
    int partition( int *arr, int p, int r )
    {
        int x = arr[r];
        int i = p - 1;
        int j = 0, temp;

        for ( j = p; j <= r - 1; j++ ) {
            if ( arr[j] <= x ) {
                i += 1;
                temp = arr[i];
                arr[i] = arr[j];
                arr[j] = temp;
            }
        }
        temp = arr[i + 1];
        arr[i + 1] = arr[r];
        arr[r] = temp;

        return i + 1;
    }
    void quick_sort( int *arr, int p, int r )
    {
        if ( p < r ) {
            int q = partition( arr, p, r );
            quick_sort( arr, p, q - 1 );
            quick_sort( arr, q + 1, r );
        }
    }
    void adjust( int k, int s )
    {
        // t为结点s在败者树数组的父结点，
        // 例如64路归并，输入63/62，他们
        // 的父结点均为 63
        int t = ( k + s ) / 2;

        while ( t > 0 ) {
            if ( s == -1 ) {
                break;
            }

            // 第一趟，输入一个叶子结点s，让s与父结点的值比较
            // （这里父结点一定保存上一次比较的较大者），如果s
            // 大于父结点的值，表示s为新的败者，将表示s结点的
            // 索引放到父结点处，胜者（父结点）继续到更上层的
            // 父结点进行比较，这样比较完后，顶点一定放置的最
            // 小的值

            // 将败者树想象为一场淘汰赛，假如有8个参赛者入口，
            // 8个参赛者编号1-8，第一轮（即初始化败者树），假
            // 设产生了2/4/6/8四强败者，放置到8个入口上一层（
            // 即父结点），然后再产生两强败者5/7，放置到更上一
            // 层的结点，然后产生最后的失败者1，而剩余的8就是最
            // 终的胜者，这样一棵败者树就初始化好了，

            // 8个入口，之后我们可以随便哪个入口加入一个参赛者，
            // 这个参赛者只需要与父结点进行比较，败者留下，胜者
            // 可以往更高的父结点去参加比赛....这样每一轮进入一
            // 个参赛者，每次能得到一个新的冠军(最小值)，然后写入
            // 文件末尾

            // 这里的8其实就是8路归并，这个入口的参赛者每次就去
            // 读取8个排好序的文件或缓冲区
            if ( buf[s] > buf[lst[t]] ) {
                int temp = s;
                s = lst[t];
                lst[t] = temp;
            }
            t >>= 1;
        }
        // 以2^n次方来算，顶层败者编号为1，所以败者树数组lst[0]一定
        // 没存东西，可以用来存放最后的冠军
        lst[0] = s;
    }
    void create_loser_tree(int k)
    {
        int i = 0;
        for( i; i < k; i++ ) {
            lst[i] = -1;
        }

        for( i = k - 1; i >= 0; i-- ) {
            adjust(k, i);
        }
    }
    void k_merge(
        int k )
    {
        int i = 0;
        FILE **ftemp = ( FILE * )malloc( sizeof(FILE *) * k );
        FILE *fout = NULL;

        // 归并路数大小的数组，每个数组值存放每一个归并路文件读取的
        // 一个值，某一个索引的值写入输出文件，又读取对应文件下一个
        // 值补充
        buf = ( int * )malloc( sizeof(int) * k );

        fout = fopen( OUTPUT_FILE, "w+" );

        for ( i; i < k; i++ ) {
            char file_name[20] = {0};
            snprintf( file_name, sizeof(file_name), TEMP_PREFIX"%d", i );
            ftemp[i] = fopen( file_name, "r" );
            // 读取每个排序好的临时文件第一个数
            fscanf( ( FILE * )ftemp[i], "%d ", buf + i );
        }

        // 以排好序文件第一个数的数组来创建败者树，
        // 树结点产生败者，这样以后的每轮比较只需要
        // 去文件或缓冲区读取下一个值加入败者树入口即可
        create_loser_tree( k );

        // 开始归并， 哪一个入口产生的冠军，先把冠军写入输出文件，
        // 然后冠军所属的文件或缓冲区再读入一个数进行比赛，如果某一路
        // 文件或缓冲区读到尾了，那么这个入口的参赛者为无限大，这样
        // 与之共有一个父结点的兄弟结点每次读取的值都能成为胜者，参加
        // 父结点以上的比较，到所有节点都读完时，最终败者结点，即lst[1]
        // 为无穷大，再加入一个参赛者，lst[0]也为无穷大了，
        while ( buf[lst[0]] != INFINITY ) {
            // 读取冠军的值
            int q = lst[0];

            // 将冠军写入输出文件
            fprintf(fout, "%d\n", buf[q]);

            // 读取冠军所属队列（文件或缓冲区）的下一个值
            read_data(ftemp[q], &buf[q]);

            // 加入了一个新参赛者，调整败者树
            adjust(k, q);
        }

        // 清理
        free( buf );

        for ( i = 0; i < k; i++ ) {
            fclose(ftemp[i]);
        }
    }

    void memory_sort_small_file(
        FILE *fp,
        int num, // 待排序数的数量
        int k )
    {
        int i = 0;
        int num_per_ways = num / k; // 每一路多少个数
        int *buf = NULL;
        FILE **ftemp = ( FILE * )malloc( sizeof(FILE *) * k );

        buf = ( int * )malloc( sizeof(int) * num_per_ways + 1000 );

        // for ( i = 0; i < k; i++ ) {
        //     char temp_buf[20] = {0};
        //     snprintf( temp_buf, sizeof(temp_buf), TEMP_PREFIX"%d", i);
        //     ftemp[i] = fopen( temp_buf, "w+" );
        //     if ( ftemp[i] == NULL ) {
        //         printf("[%s:%d],error occured!!(%s)\n", __func__, __LINE__, strerror(errno));
        //         exit( 0 );
        //     }
        // }

        // 先不处理最后一个，可能总数/k路带余数，多余的
        // 留到最后一个文件处理
        k--;
        while ( k > 0 ) {

            char temp_buf[20] = {0};
            snprintf( temp_buf, sizeof(temp_buf), TEMP_PREFIX"%d", k);
            ftemp[k] = fopen( temp_buf, "w+" );
            if ( ftemp[k] == NULL ) {
                printf("[%s:%d],error occured!!(%s)\n", __func__, __LINE__, strerror(errno));
                exit( 0 );
            }

            i = 0;
            memset( buf, 0, sizeof(buf) );

            for ( i; i < num_per_ways; i++ ) {
                fscanf(fp, "%d ", &buf[i]);
            }
            printf("%s:%d, K:%d\n", __func__, __LINE__, k);
            quick_sort( buf, 0, num_per_ways - 1 );
            for ( i = 0; i < num_per_ways; i++ ) {
                fprintf(ftemp[k], "%d ", buf[i]);
            }
            fclose( ftemp[k] );
            k--;
        }

        // 处理剩余的最后一个待排序文件
        char temp_buf[20] = {0};
        snprintf( temp_buf, sizeof(temp_buf), TEMP_PREFIX"%d", 0);
        ftemp[0] = fopen( temp_buf, "w+" );
        if ( ftemp[0] == NULL ) {
            printf("[%s:%d],error occured!!(%s)\n", __func__, __LINE__, strerror(errno));
            exit( 0 );
        }

        i = 0;
        while ( fscanf(fp, "%d ", &buf[i]) != EOF ) i++;
        printf("%s:%d, K:%d\n", __func__, __LINE__, 0);
        quick_sort( buf, 0, i );

        int j = 0;
        for ( j = 0; j <= i; j++ ) {
            fprintf(ftemp[0], "%d ", buf[j]);
        }
        free( buf );
        fclose( ftemp[0] );
    }
    int main(
        int argc,
        char **argv )
    {
        if ( argc != 3 ) {
            printf("usage:\n\t./xxx file_name k ways to merge\n");
            exit( 0 );
        }



        int k = atoi( argv[2] );
        char *file_name = argv[1];

        FILE *fp = fopen(file_name, "r");
        if ( fp == NULL ) {
            printf("[%s:%d],error occured!!(%s)\n", __func__, __LINE__, strerror(errno));
            exit( 0 );
        }

        time_t t1 = time(NULL), t2, t3;
        memory_sort_small_file( fp, 10000000, k );

        t2 = time(NULL);

        k_merge( k );

        t3 = time(NULL);

        printf("---------------------------finish-----------------------------\n");

        printf("\tmemory sort & ouput to temp file cost:  %ds\n", (int)(t2 - t1));

        printf("\tk_merge & ouput to file cost:  %ds\n", (int)(t3 - t2));

        printf("\ttotal cost time:  %ds\n", (int)(t3 - t1));

        printf("--------------------------------------------------------------\n");

        fclose( fp );

        return 0;
    }


64路归并排序1000w个数用时：

![out_sort1]

生成文件：

![out_sort2]

排序后的文件头和尾：

![out_sort3]
![out_sort4]

代码注释写了很多了，以后忘了回头看看也能记起来 …..

[1]:https://blog.csdn.net/v_july_v/article/details/6451990
[2]:www.cnblogs.com/harryshayne/archive/2011/07/02/2096196.html
[3]:http://kenby.iteye.com/blog/1017532
[4]:http://www.cnblogs.com/eaglet/archive/2011/01/17/1937083.html

[out_sort1]:/img/out_sort1.png ""
[out_sort2]:/img/out_sort2.png ""
[out_sort3]:/img/out_sort3.png ""
[out_sort4]:/img/out_sort4.png ""