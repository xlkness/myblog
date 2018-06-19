---
title: 分治之最大子数组-《算法导论》学习笔记四
date: 2018-06-19 19:35:51
categories:
- 开发
tags:
- 数据结构/算法
---

《算法导论》第4章4.1使用分治策略求最大子数组（数组包含负数，不然整个数组即使最大子数组，求解没意义）。

思路：数组头为low，尾为high，mid=(low+high)/2，这样将数组分为了两段。首先肯定存在这个最大子数组。那么子数组的位置要么处于mid左边，要么处于mid右边，要么包含mid。假设最大子数组出现在mid左边，对mid左边子数组再进行(low+high)/2切分，那么最大子数组可能在切分出的子数组中的位置又存在三种情况中的一种，这样递归地切分下去，最终切分到整个数组都变为1-2个元素的子数组，这时候的情况就像一颗二叉树，例如数组[53,-4,-73,-16,88,91,-50,-15,-15,52,-19]，切分之后的二叉树：

![maxsubarr]

然后用后续遍历二叉树的方式计算、查找最大子数组。如图，节点8的子数组有53、-4、53,-4，则找出来最大子数组为[53]，后序遍历的方式遍历完左子树，返回根，到右子树，有一个节点9，则最大子数组为9，对于节点4的左右子树的最大子数组都找到，再对节点8、4、9合并的数组找位于mid的最大子数组(此时low为0，mid为1，high为2，从mid出发往左走找最大子数组，再从mid+1出发往右走找最大子数组，再将找到的两个数组合并起来，为经过mid的最大子数组)，找到为[53,-4,-73]，将[53]、[-73]，[53,-4,-73]比较，得出根节点为4的树最大数组为[53]，则再返回根节点2，再后续到节点5，对5再求最大子数组为[88,91]，再对节点2求经过mid的最大子数组为[53,-4,-73,-16,88,91]，和为139，与[53]、[88,91]比较，选[88,91]；接着返回根节点1，对右子树节点2求最大子数组，为[52]，再对节点1组成的数组求经过mid的最大子数组，为[88,91,-50,-15,-15,52]，和为151，与[88,91]、[52]比较，选[88,91]。

    c代码：

    #include <stdio.h>
    #include <unistd.h>
    #include <time.h>
    #include <string.h>
    #include <stdlib.h>


    int crossLow = 0, crossHigh = 0, crossSum = 0;

    int finalLeftIndex = -1;
    int finalRightIndex = -1;
    int finalSubArraySum = 0;

    //求经过mid的最长子数组，范围low-high
    void find_max_crossing_subarray( int *arr,
        int low, int mid, int high )
    {
        if ( low == mid && mid == high ) {
            crossLow = low;
            crossHigh = high;
            crossSum = arr[mid];
            return ;
        }
        int lMax = -10000000, rMax = -10000000;
        int lTmpSum = 0, rTmpSum = 0;
        int i = 0;
        for ( i = mid; i >= low; i-- ) {
            lTmpSum += arr[i];
            if ( lTmpSum > lMax ) {
                lMax = lTmpSum;
                crossLow = i;

            }
        }
        for ( i = mid + 1; i <= high; i++ ) {
            rTmpSum += arr[i];
            if ( rTmpSum > rMax ) {
                rMax = rTmpSum;
                crossHigh = i;

            }
        }
        crossSum = lMax + rMax;
    }
    //分治，将数组切分子规模的待求数组
    void find_maximum_subarray( int *arr,
        int low, int high )
    {
        if ( high <= low + 1 ) {
            find_max_crossing_subarray( arr, low, low, high );
            if ( arr[low] >= arr[high] && arr[low] >= crossSum ) {
                finalLeftIndex = low;
                finalRightIndex = low;
                finalSubArraySum = arr[low];
            }
            else if ( arr[high] >= arr[low] && arr[high] >= crossSum ) {
                finalLeftIndex = high;
                finalRightIndex = high;
                finalSubArraySum = arr[high];
            }
            else {
                finalLeftIndex = low;
                finalRightIndex = high;
                finalSubArraySum = crossSum;
            }
        }
        else {
            int lLow = 0, lHigh = 0, lSum = 0;
            int rLow = 0, rHigh = 0, rSum = 0;

            //其实这里的递归就像二叉树的后续遍历，直到遍历完左子树，再开始右子树，
            //这样的好处假如数组有上亿的元素，不会造成栈空间不足，
            //解决了一个左子树就返回了递归栈，再进行右子树的展开工作
            int mid = ( low + high ) / 2;
            find_maximum_subarray( arr, low, mid );
            lLow = finalLeftIndex;
            lHigh = finalRightIndex;
            lSum = finalSubArraySum;
            find_maximum_subarray( arr, mid + 1, high );
            rLow = finalLeftIndex;
            rHigh = finalRightIndex;
            rSum = finalSubArraySum;
            find_max_crossing_subarray( arr, low, mid, high );
            // printf("%d %d %d/%d %d %d/%d %d %d\n",
            //     lLow, lHigh, lSum,
            //     rLow, rHigh, rSum,
            //     crossLow, crossHigh, crossSum);
            if ( lSum >= rSum && lSum >= crossSum ) {
                finalLeftIndex = lLow;
                finalRightIndex = lHigh;
                finalSubArraySum = lSum;
            }
            else if ( rSum >= lSum && rSum >= crossSum ) {
                finalLeftIndex = rLow;
                finalRightIndex = rHigh;
                finalSubArraySum = rSum;
            }
            else {
                finalLeftIndex = crossLow;
                finalRightIndex = crossHigh;
                finalSubArraySum = crossSum;
            }
            // printf("------%d %d %d\n", finalLeftIndex, finalRightIndex, finalSubArraySum);
        }
    }
    //暴力破解法求最长子数组，不过这里用来测试我写的分治求法结果是否正确
    int check_result( int *arr, int len, int lIndex, int rIndex, int Sum )
    {
        int maxSum = -100000000;
        int lIndex1;
        int rIndex1;
        int i = 0, j = 0;

        for ( i = 0; i < len; i++ ) {
            int tmpSum = 0;
            for ( j = i; j < len; j++ ) {
                tmpSum += arr[j];
                if ( tmpSum > maxSum ) {
                    maxSum = tmpSum;
                    lIndex1 = i;
                    rIndex1 = j;
                }
            }
        }
        if ( Sum == maxSum ) {
            if ( lIndex == lIndex1 && rIndex == rIndex1 ) {
                return 0;
            }
            printf("equal!!\n");
            printf("exhaustivly_find_result:lIndex->%d,rIndex->%d,Sum->%d\n",
                lIndex1, rIndex1, maxSum);
            printf("divide_and_conquer_result:lIndex->%d,rIndex->%d,Sum->%d\n",
                lIndex, rIndex, Sum);
            return 1;
        }
        // else {
        //     printf("not equal!!!!!!!!!");
        // }
        return -1;
    }
    void mainLoop( int *arr, int len )
    {
        find_maximum_subarray( arr, 0, len - 1 );
        printf("----------------------------------------------------\n");
        printf("lIndex:%d,rIndex:%d,subArrSum:%d\n",
            finalLeftIndex, finalRightIndex, finalSubArraySum);
        printf("----------------------------------------------------\n");
    }
    void stepLoop( int *arr, int len )
    {
        time_t tt0 = time( NULL );
        printf("before:%s", ctime(&tt0));
        mainLoop( arr, len );
        time_t tt1 = time( NULL );
        printf("after:%s", ctime(&tt1));
        printf("cost %d(sec),%d(min)\n",
            (int)(tt1 - tt0), (int)((tt1 - tt0) / 60));
    }
    //初始化随机数组
    void initArr( int *arr, int lowV,
        int upV, int len )
    {
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
    int main( int argc, char **argv )
    {
        srand( (int)time(NULL) );
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

        // int len = 10;
        // int arr[10] = {7, 0, -86, 61, -72, 50, -38, -25, -70, -76};

        int i = 0;

        //随机10000个数组求最大子数组，然后检测结果是否正确
        for ( i = 0; i < 10000; i++ ) {
            initArr( arr, lowV, upV, len );
            printArr( arr, len );
            // stepLoop( arr, len );
            usleep(1000);
            find_maximum_subarray( arr, 0, len - 1 );
            int ret = check_result( arr, len,
                        finalLeftIndex, finalRightIndex, finalSubArraySum );
            if ( ret < 0 ) {
                printf("failed!!\n");
                return 0;
            }
            printf("-------------equal:%d\n", i);
        }

        free( arr );
        arr = NULL;

        return 0;
    }

以上代码定义了几个全局变量，与《算法导论》的代码有点出入，书上的函数块返回3个值，例如：(left-low,left-high,left-sum) = FIND-MAXIMUM-SUBARRAY(A, low, mid)，这里可能用其它编程语言更好描述算法，erlang/golang/python的函数就可以返回一个元组并接收{LeftLow, LeftHigh, LeftSum} = FIND….()。

[maxsubarr]:/img/max_sub_arr.jpeg ""
