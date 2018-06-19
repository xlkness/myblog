---
title: 堆排序-《算法导论》学习笔记六
date: 2018-06-19 20:18:42
categories:
- 开发
tags:
- 数据结构/算法
---


堆排序就是将一组数按二叉树层序遍历的存储顺序，经过一系列比较转移，最终变成有序的数组，这里的二叉树堆一定是完全二叉树。堆排序能进行的基础是有个最大堆的数组，最大堆性质是指树上的每个节点的子节点都比自己小或等于。
因此最顶上的根节点一定是值最大的节点，有了最大堆在，堆排序就可以进行了，从层序遍历的最后一个节点开始倒序循环，交换当前节点与最顶层根节点，即最大值的节点，这样每次最大的节点都被放在层序遍历的最后位置，类似冒泡排序了，而放了最大值的节点即从堆中排除（只要堆长度减一即表示堆没有这个节点了），交换到顶点根节点的值再做一次下滤操作（以这个节点值与子树的最大值交换），保证剩余子树一定也是最大堆性质。
贴代码：

    #include <stdio.h>
    #include <string.h>
    #include <unistd.h>
    #include <time.h>
    #include <stdlib.h>

    int parent( int );
    int left( int );
    int right( int );

    inline int parent( int index )
    {
        return index / 2;
    }
    inline int left( int index )
    {
        return index * 2;
    }
    inline int right( int index )
    {
        return index * 2 + 1;
    }

    //下滤
    void max_heapify(
        int *arr,
        int heap_size,
        int index )
    {
        int l = left( index );
        int r = right( index );
        int largest = 0;

        if ( l <= heap_size && arr[l] > arr[index] ) {
            largest = l;
        } else {
            largest = index;
        }
        if ( r <= heap_size && arr[r] > arr[largest] ) {
            largest = r;
        }
        if ( largest != index ) {
            int temp = arr[index];
            arr[index] = arr[largest];
            arr[largest] = temp;
            max_heapify( arr, heap_size, largest );
        }
    }
    //构建最大堆
    void build_max_heap(
        int *arr,
        int length,
        int heap_size )
    {
        int i = 0;

        for ( i = length / 2; i >= 1; i-- ) {
            max_heapify( arr, heap_size, i );
        }
    }
    //开始堆排序
    void heap_sort(
        int *arr,
        int length,
        int heap_size )
    {
        int i = 0;

        build_max_heap( arr, length, heap_size );

        for ( i = heap_size; i >= 2; i-- ) {
            int temp = arr[1];
            arr[1] = arr[i];
            arr[i] = temp;
            heap_size -= 1;
            max_heapify( arr, heap_size, 1 );
        }
    }
    void check_is_inc_arr(
        int *arr,
        int len )
    {
        int i = 0;

        for ( i = 1; i <= len; i++ ) {
            if ( arr[i] < arr[i - 1] ) {
                printf("check_is_inc_arr fail.\n");
                return;
            }
        }
        printf("check_is_inc_arr ok.\n");
    }
    //初始化随机数组
    void initArr( int *arr, int lowV,
        int upV, int len )
    {
        int i = 0;
        int size = upV - lowV;

        for ( i = 1; i < len + 1; i++ )
        {
            arr[i] = rand() % size + lowV;
        }
        arr[0] = 0;
    }
    void print_arr(
        int *arr,
        int len )
    {
        int i = 0;

        printf("\n=============================================\n");
        for ( i = 0; i <= len; i++ ) {
            printf("%d ", arr[i]);
        }
        printf("\n=============================================\n");
    }
    int main(
        int argc,
        char **argv )
    {
        if ( argc != 4 ) {
            printf("input array length and the random value range.\n");
            exit( 0 );
        }

        srand( (int)time(NULL) );

        int length = atoi( argv[1] );
        int lowV = atoi( argv[2] );
        int upV = atoi( argv[3] );

        int *arr = ( int * )malloc( sizeof(int) * (length + 1) );

        initArr( arr, lowV, upV, length );

        //print_arr( arr, length );
        heap_sort( arr, length, length );
        //print_arr( arr, length );

        check_is_inc_arr( arr, length );

        return 0;
    }

上面代码中因为c数组从0开始的原因，因此0位置不参与存储和运算，数据形式是{0, 2, 3, 4, 5, 6}。