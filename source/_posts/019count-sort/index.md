---
title: 计数排序-《算法导论》学习笔记八
date: 2018-06-19 20:27:08
categories:
- 开发
tags:
- 数据结构/算法
---

计数排序：假设n个输入元素中的每一个都是在0-k区间内的一个整数(k为正整数)，对每一个输入元素x，确定小于x的元素个数，用一个0-k长度的数组做记录，例如输入数组的数为0-10长度，随机出[2,5,3,0,2,3,0,3]，可以计算出一个0-10的数组，分别表示小于等于x的数的个数，于是有：0->2,1->2,2->4,3->7,4->7,5->8,6->8,7->8,9->8,10->8，然后再倒序遍历待排序数组，根据值去0-10数组取索引，然后将数组值放入一个新的等长数组那个索引处，取过的索引值要减一，这是为了下次再取到相同值不会放在同一位置，代码：

    #include <stdio.h>
    #include <string.h>
    #include <unistd.h>
    #include <time.h>
    #include <stdlib.h>


    void counting_sort(
        int *in_arr,
        int *out_arr,
        int arr_len,
        int k )
    {
        int *tmp_arr = ( int * )malloc( sizeof(int) * (k + 1) );
        int i = 0, j = 0;

        for ( i = 0; i < k; i++ ) {
            tmp_arr[i]  = 0;
        }

        for ( j = 0; j < arr_len; j++ ) {
            tmp_arr[in_arr[j]] = tmp_arr[in_arr[j]] + 1;

        }

        for ( i = 1; i < k + 1; i++ ) {
            tmp_arr[i] = tmp_arr[i] + tmp_arr[i - 1];
        }

        for ( j = arr_len - 1; j >= 0; j-- ) {
            //printf("j:%d, in_arr[j]:%d, tmp_arr[in_arr[j]]:%d\n",
            //  j, in_arr[j], tmp_arr[in_arr[j]]);
            out_arr[tmp_arr[in_arr[j]] - 1] = in_arr[j];
            tmp_arr[in_arr[j]] = tmp_arr[in_arr[j]] - 1;
        }

        free( tmp_arr );
    }

    void check_is_inc_arr(
        int *arr,
        int len )
    {
        int i = 0;

        for ( i = 0; i < len; i++ ) {
            if ( arr[i] < arr[i - 1] ) {
                printf("check_is_inc_arr fail.\n");
                return;
            }
        }
    }
    //初始化随机数组
    void initArr( int *arr, int lowV,
        int upV, int len )
    {
        int i = 0;
        int size = upV - lowV;

        for ( i = 0; i < len; i++ )
        {
            arr[i] = rand() % size + lowV;
        }
    }
    void print_arr(
        int *arr,
        int len )
    {
        int i = 0;

        printf("\n=============================================\n");
        for ( i = 0; i < len; i++ ) {
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

        int *arr = ( int * )malloc( sizeof(int) * (length) );
        int *new_arr = ( int * )malloc( sizeof(int) * length );


        int i = 1000;

        for ( ; i > 0; i-- ) {
            initArr( arr, lowV, upV, length );
            //print_arr( arr, length );
            counting_sort( arr, new_arr, length, upV );
            //print_arr( new_arr, length );
            check_is_inc_arr( new_arr, length );
        }

        return 0;
    }