---
title: 递归插入排序-《算法导论》学习笔记三
date: 2018-06-15 20:15:00
categories:
- 开发
tags:
- 摄像头
---

算法导论第二章结尾练习2.3-4提到将插入排序写递归版本，然后尝试写了个，本来写了就好了，但是调试的时候排序10w个数可以，排序100w个数就段错误，分析了一下，把结果放上来以后查看，先贴代码：

    #include <stdio.h>
    #include <unistd.h>
    #include <time.h>
    #include <string.h>
    #include <stdlib.h>

    void insert_sort0( int *arr, int index )
    {
            int i, tmp = arr[index];
            //printf("index:%d\n", index);
            for ( i = index - 1; i >= 0; i-- ) {
                    if ( arr[i] > tmp ) {
                            arr[i + 1] = arr[i];
                    } else {
                            arr[i + 1] = tmp;
                            return;
                    }
            }

            arr[i + 1] = tmp;
    }

    void insert_sort( int *arr, int index )
    {
            //printf("index:%d\n", index);
            if ( index > 0 ) {
                    insert_sort( arr, index - 1 );
                    //printf("index:%d\n", index);
                    insert_sort0( arr, index );
            }
    }
    void sortLoop0( int *arr, int len )
    {
            //sleep( 20 );
            insert_sort( arr, len - 1 );
    }
    //排序前后做时间对比，精度sec
    void sortLoop( int *arr, int len )
    {
            time_t tt0 = time( NULL );
            printf("\tbefore sort:%s", ctime(&tt0));
            sortLoop0( arr, len );
            time_t tt1 = time( NULL );
            printf("\tafter sort:%s", ctime(&tt1));
            printf("\tsort the array cost %d(sec),%d(min)\n",
                    (int)(tt1 - tt0), (int)((tt1 - tt0) / 60));
    }
    //初始化随机数组
    void initArr( int *arr, int lowV,
            int upV, int len )
    {
            srand( (int)time(NULL) );
            int i = 0;
            int size = upV - lowV;

            for ( ; i < len; i++ )
            {
                    arr[i] = rand() % size + lowV;
            }
    }
    //打印数组
    void printArr( int *arr, int len )
    {
            int i = 0;
            for ( ; i < len; i++ )
                    printf("%d ", arr[i]);
            printf("\n");
    }
    //这里写了个简单得数组检查，检查是否
    //是正确的递增序列(怕自己的代码没检查
    //边界条件偶尔有错误的排序数组)
    int check_arr_increase( int *arr, int len )
    {
            int i = 0;

            for ( ; i < len - 1; i++ )
            {
                    if ( arr[i] <= arr[i + 1] )
                            continue;
                    else
                    {
                            printf("array isn't increase!!!\n");
                            return -1;
                    }

            }
            printf("array is increase!!!!\n");
            return 0;
    }
    int main( int argc, char **argv )
    {
            if ( argc != 4 )
            {
                    printf("usage: ./execfile lowV upV len\n");
                    return 0;
            }

            int lowV = atoi( argv[1] );
            int upV = atoi( argv[2] );
            unsigned int len = atoi( argv[3] );

            int *arr = NULL;
            arr = ( int *) malloc( len * sizeof(int) );

            initArr( arr, lowV, upV, len );
            //printArr( arr, len );
            sortLoop( arr, len );
            check_arr_increase( arr, len );
            //printArr( arr, len );

            free( arr );
            arr = NULL;

            return 0;
    }

开始一直以为是代码有数组越界或者边界条件没考虑到，后来看了代码许久没发现问题。然后gdb看core文件，bt看调用栈发现段在递归向下大概70w-80w的位置，而递归栈还没返回，执行过程很正常没出问题，想了想是栈溢出，每次自顶向下调用一次自身函数就要在栈里保存一次函数地址和参数。

用ulimit -a|grep stack看了一下默认的栈空间大小为8192kb=8m=8*1024*1024=8388608byte。网上查了下说一次函数调用每次占用栈内存为：4返回地址 + 4*参数个数 + 4寄存器保护 + 4*局部变量数（未验证准确与否），我的排序占用4+4*2+12=24字节，但根据我每次设置栈大小调试程序，设置到31256kb时程序没有段错误，也就是31256*1024/1000000=32字节，这里的一次函数调用为32字节。

用ulimit -s 102400修改了栈空间大小100M（暂时修改），再次运行程序排序100w个数，发现打印正常，程序也没段错误了，只是递归栈开始返回等了好久。

对归并排序的思考：以前也有用归并排序排1亿个数（http://blog.csdn.net/u012785877/article/details/52475023）[1]，没有出现栈溢出，里面也是用递归的方法，因此仔细看了下归并排序（算法导论里有提到归并排序的递归树，可以算出数量，但是我看着头大，准备略过这些数学问题，先照着把代码写出来，看完了再回头深入看一遍），模拟了一下递归的栈过程如下：

假设有134217728个数待排序(2^27)。因此递归树根节点为2^27；递归第一次递归树有二层，两个节点分别为2^26、2^26；递归第三次递归树为有三层，四个节点分别为2^25、2^25、2^25、2^25……这样一直推到递归树叶子节点为2^1，此递归树有27层，2^(floor - 1) + 1 = 2^(26 -1) + 1 = 67108865个节点，也就是要维护67108865*30约为20亿的函数调用栈空间，为什么没有栈溢出呢？。通过调试归并排序，排序4个数时，是将0-3分为0-1，在将0-1分为0，对0再分发现不能分了，则此次递归返回到父节点，划分右子树1，对1再分发现不能分了，于是调用排序，将0、1排为0-1，再回归0-1的父节点，再对右子树2-3递归进行此过程。其实排序4个数最大只需要5次递归函数调用（即递归树走到最深度），如果排序n层递归树，最大需要n+1次。回到以上排序2^27个数，则最大同时需要28次递归函数调用，其余时候都是栈空间在不断出栈入栈。

上面写得有点昏，无奈不是学文科的，又不想画图，额。这个归并排序的分析结果只花了2个小时分析，也不能确定准不准确，如果错了欢迎批评指正。

[1]:http://blog.csdn.net/u012785877/article/details/52475023）