---
title: 图的简单表示-算法学习笔记十七
date: 2018-06-20 12:49:36
categories:
- 开发
tags:
- 数据结构/算法
---

基于邻接矩阵和邻接链表的图表示法，以及各自的深度优先遍历和广度优先遍历，但图的表示中没有加带权的边，只是简单写一写，学习一下，底层链表和队列用了通用链表

    #include <stdio.h>
    #include <stdlib.h>
    #include <string.h>
    #include <unistd.h>

    #define PRINT(format, arg...) \
    do{ \
        printf("[%s/%d]:", __func__, __LINE__); \
        printf(format, ##arg); \
        printf("\n"); \
    }while(0)

    /*****************************图********************************/

    #define MAX_VERTEX_NUM 100

    typedef struct init_array {
        int start_v;
        int end_v;
    } init_array;

    /**************邻接矩阵表示****************/
    typedef struct matrix_graph {
        int vertex[MAX_VERTEX_NUM];  // 存放顶点信息
        int v_num;  // 顶点数
        int e_num;  // 边数
        int matrix[MAX_VERTEX_NUM][MAX_VERTEX_NUM];
    } matrix_graph;

    matrix_graph *init_matrix_graph(
        int v_num,
        int e_num )
    {
        int i = 0, j = 0;
        matrix_graph *graph = NULL;

        graph = ( matrix_graph * )malloc( sizeof(matrix_graph) );
        graph->v_num = v_num;
        graph->e_num = e_num;

        for ( i = 0; i < MAX_VERTEX_NUM; i++ ) {
            if ( i < v_num )
                graph->vertex[i] = i;
            else
                graph->vertex[i] = -1;

            for ( j = 0; j < MAX_VERTEX_NUM; j++ ) {
                graph->matrix[i][j] = 0;
            }
        }

        return graph;
    }
    matrix_graph *create_matrix_graph(
        init_array *i_arr,
        int arr_size,
        int v_num )
    {
        int i = 0;
        matrix_graph *graph = NULL;

        graph = init_matrix_graph( v_num, arr_size );

        for ( i = 0; i < arr_size; i++ ) {
            graph->matrix[i_arr[i].start_v][i_arr[i].end_v] = 1;
        }

        return graph;
    }
    int visited[MAX_VERTEX_NUM] = {0};
    // 深度优先遍历
    void dfs_matrix_graph1(
        matrix_graph *g,
        int v )
    {
        visited[v] = 1;
        printf("%d ", v);

        // 搜索此结点的下一个结点
        int i = 0;
        for ( i; i < g->v_num; i++ ) {
            if ( g->matrix[v][i] && !visited[i] )
                dfs_matrix_graph1( g, i );
        }
    }
    void dfs_matrix_graph(
        matrix_graph *g )
    {
        int i = 0;

        for ( i; i < g->v_num; i++ )
            visited[i] = 0;

        printf("deep first visit:\n\t");

        for ( i = 0; i < g->v_num; i++ ) {
            if ( !visited[i] )
                dfs_matrix_graph1( g, i );
        }
        printf("\n");
    }
    // 广度优先搜索
    void bfs_matrix_graph(
        matrix_graph *g )
    {
        int i = 0;
        // 栈结点数据域，需要void *
        int tmp_vertex[MAX_VERTEX_NUM] = {0};
        queue *q = create_queue();

        for ( i = 0; i < g->v_num; i++ )
            visited[i] = 0;

        printf("broad first visit:\n\t");

        for ( i = 0; i < g->v_num; i++ ) {
            if ( !visited[i] ) {
                tmp_vertex[i] = i;
                q->enqueue(q->this, &tmp_vertex[i]);

                while ( q->size(q->this) != 0 ) {
                    int tmp = *( int * )q->dequeue( q->this );
                    if ( !visited[tmp] )
                        printf("%d ", tmp);

                    visited[tmp] = 1;

                    int j = 0;
                    for ( j = 0; j < g->v_num; j++ ) {
                        if ( g->matrix[tmp][j] && !visited[j] ) {
                            tmp_vertex[j] = j;
                            q->enqueue( q->this, &tmp_vertex[j] );
                        }
                    }
                }
            }
        }

        q->free( q->this );
        printf("\n");
    }
    void free_matrix_graph(
        matrix_graph *g )
    {
        free( g );
    }
    #undef DEBUG_MATRIX_GRAPH
    #ifdef DEBUG_MATRIX_GRAPH

    int main()
    {
        init_array i_arr[20] =
            {
                {0, 4},
                {0, 9},
                {1, 5},
                {2, 1},
                {2, 6},
                {3, 2},
                {3, 6},
                {4, 7},
                {4, 8},
                {5, 2},
                {6, 5},
                {7, 6},
                {7, 3},
                {8, 7},
                {8, 4},
                {9, 8},
            };

        matrix_graph *g = create_matrix_graph( i_arr, 16, 10 );

        dfs_matrix_graph( g );

        bfs_matrix_graph( g );

        free_matrix_graph( g );
    }

    #endif
    /********************************************/

    /**************图的邻接链表*******************/
    typedef struct link_list_graph {
        int vertex[MAX_VERTEX_NUM];
        int v_num;
        int e_num;
        link_list v_list[MAX_VERTEX_NUM];
    } link_list_graph;

    // 链表数据域，需要void *
    int tmp_vertex[MAX_VERTEX_NUM] = {0};

    link_list_graph *create_link_list_graph(
        init_array *i_arr,
        int arr_size,
        int v_num )
    {
        int i = 0;
        link_list_graph *graph = NULL;

        graph = ( link_list_graph * )malloc( sizeof(link_list_graph) );
        graph->e_num = arr_size;
        graph->v_num = v_num;
        for ( i = 0; i < v_num; i++ ) {
            tmp_vertex[i] = i;

            //定制链表操作
            graph->v_list[i].size = 0;
            graph->v_list[i].insert = insert_tail;
            graph->v_list[i].get_first = get_head;
            graph->v_list[i].get_last = get_tail;
            graph->v_list[i].del = del_tail;
            graph->v_list[i].insert( &graph->v_list[i], &tmp_vertex[i]);
        }

        for ( i = 0; i < arr_size; i++ ) {
            int start = i_arr[i].start_v;
            int end = i_arr[i].end_v;
            graph->v_list[start].insert( &graph->v_list[start], &tmp_vertex[end]);
            printf("start:%d->end:%d, size:%d\n", start, end, graph->v_list[start].size);
        }

        return graph;
    }
    void dfs_link_list_graph1(
        link_list_graph *g,
        int v )
    {
        link_list_node *p = g->v_list[v].head;

        if ( !visited[*(int *)(p->data)] ) {
            printf("%d ", *(int *)(p->data));
            visited[v] = 1;
            while ( 1 ) {
                p = p->next;
                if ( p == g->v_list[v].head )
                    break;
                if ( !visited[*(int *)(p->data)] ) {
                    dfs_link_list_graph1( g, *(int *)(p->data) );
                }
            }
        }
    }
    void dfs_link_list_graph(
        link_list_graph *g )
    {
        int i = 0;

        printf("深度优先遍历：\n\t");
        for ( i = 0; i < g->v_num; i++ )
            visited[i] = 0;

        for ( i = 0; i < g->v_num; i++ ) {
            if ( g->v_list[i].size > 0 && !visited[i] )
                dfs_link_list_graph1( g, i );
        }
        printf("\n");
    }
    void bfs_link_list_graph(
        link_list_graph *g )
    {
        int i = 0;
        // 栈结点数据域，需要void *
        int tmp_vertex[MAX_VERTEX_NUM] = {0};
        queue *q = create_queue();

        for ( i = 0; i < g->v_num; i++ ){
            tmp_vertex[i] = i;
            visited[i] = 0;
        }

        printf("广度优先遍历:\n\t");

        for ( i = 0; i < g->v_num; i++ ) {
            if ( !visited[i] ) {
                tmp_vertex[i] = i;
                q->enqueue(q->this, &tmp_vertex[i]);

                while ( q->size(q->this) != 0 ) {
                    int tmp = *( int * )q->dequeue( q->this );
                    if ( !visited[tmp] )
                        printf("%d ", tmp);

                    visited[tmp] = 1;

                    int j = 0;
                    link_list_node *p = g->v_list[tmp].head->next;

                    while ( p != g->v_list[tmp].head ) {
                        if ( !visited[*(int *)p->data] ) {
                            q->enqueue( q->this, &tmp_vertex[*(int *)p->data] );
                        }
                        p = p->next;
                    }
                }
            }
        }
        q->free( q->this );
        printf("\n");
    }
    void free_link_list_graph(
        link_list_graph *g )
    {
        int i = 0;

        for ( i = 0; i < g->v_num; ++i ) {
            del_list( &g->v_list[i] );
        }
        free( g );
    }
    #define DEBUG_LINK_LIST_GRAPH
    #ifdef DEBUG_LINK_LIST_GRAPH

    int main()
    {
        init_array i_arr[20] =
            {
                {0, 4},
                {0, 9},
                {1, 5},
                {2, 1},
                {2, 6},
                {3, 2},
                {3, 6},
                {4, 7},
                {4, 8},
                {5, 2},
                {6, 5},
                {7, 6},
                {7, 3},
                {8, 7},
                {8, 4},
                {9, 8},
            };

        link_list_graph *g = create_link_list_graph( i_arr, 16, 10 );

        dfs_link_list_graph( g );

        bfs_link_list_graph( g );

        free_link_list_graph( g );
    }

    #endif

    /********************************************/


    /***************************************************************/