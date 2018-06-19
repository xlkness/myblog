---
title: n阶矩阵一般乘法-《算法导论》学习笔记五
date: 2018-06-19 20:16:29
categories:
- 开发
tags:
- 数据结构/算法
---

A、B两个矩阵均是nxn的矩阵，则两个矩阵的乘法：

![nmatrix1]

一般的矩阵乘法代码：

    #include <stdio.h>
    #include <string.h>
    #include <stdlib.h>
    #include <unistd.h>
    #include <time.h>

    class SquareMatrix {
    public:
        SquareMatrix(){}
        SquareMatrix( int row, int col ):
            mRow(row), mCol(col)
        {
            Init();
        }
        ~SquareMatrix()
        {
            for ( int i = 0; i < mRow; i++ ) {
                delete[] mElement[i];
                //printf("delete %d\n", i);
            }
            delete[] mElement;
        }
        void Init()
        {
            mElement = new int*[mRow];
            for ( int i = 0; i < mCol; i++ ) {
                mElement[i] = new int[i];
            }
        }
        void SetElement( int lowV, int upV )
        {
            int size = upV - lowV;

            for ( int i = 0; i < mRow; i++ ) {
                for ( int j = 0; j < mCol; j++ ) {
                    mElement[i][j] = rand() % size + lowV;
                }
            }
        }
        void PrintElement()
        {
            printf("=========================================\n");
            for ( int i = 0; i < mRow; i++ ) {
                for ( int j = 0; j < mCol; j++ ) {
                    printf("%d ", mElement[i][j]);
                }
                printf("\n");
            }
            printf("=========================================\n");
        }
    public:
        int mRow;
        int mCol;
        int *(*mElement);
    };
    void SquareMatrixMultiply(
        SquareMatrix &a,
        SquareMatrix &b,
        SquareMatrix &c )
    {
        int n = a.mRow;

        c.mRow = c.mCol = n;
        c.Init();

        for ( int i = 0; i < n; i++ ) {
            for ( int j = 0; j < n; j++ ) {
                c.mElement[i][j] = 0;
                for ( int k = 0; k < n; k++ ) {
                    c.mElement[i][j] += a.mElement[i][k] * b.mElement[j][k];
                }
            }
        }
    }
    int main( int argc, char **argv )
    {

        if ( argc != 2 ) {
            printf("Usage:./binaryfile num\n");
            exit( 0 );
        }

        int n = atoi( argv[1] );

        SquareMatrix smA( n, n ), smB( n, n ), smC;
        smA.SetElement( 1, 10 );
        smA.PrintElement();

        smB.SetElement( 1, 10 );
        smB.PrintElement();

        SquareMatrixMultiply( smA, smB, smC );
        smC.PrintElement();

        printf("init square matrix finished\n");
        return 0;
    }

算法复杂度为O(n^3)，而Stranssen算法通过分治法将大矩阵切分为小矩阵进行计算，算法复杂度可以降低为O(n^2.81)，但是尝试写下代码，发现切割子矩阵时有点复杂，普通的切分会创建子矩阵并复制值，而用下标进行计算又比较复杂，下次有空再尝试写吧。

[nmatrix1]:/img/nmatrix1.png ""