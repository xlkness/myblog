---
title: 通用链表/栈/队列-算法学习笔记十六
date: 2018-06-20 12:47:44
categories:
- 开发
tags:
- 数据结构/算法
---

今天准备学习数据结构-图，会用到栈和队列，因此写了下代码，底层用了通用链表，为循环双向结构，结点数据域为void *；通用链表层之上封装了栈和队列，比较简单，但是代码行数有点多，单独摘出：

    #include <stdio.h>
    #include <stdlib.h>
    #include <string.h>
    #include <unistd.h>
    #include <time.h>

    #define MAX_VERTEX_NUM 1000

    #define PRINT(format, arg...) \
    do{ \
        printf("[%s/%d]:", __func__, __LINE__); \
        printf(format, ##arg); \
        printf("\n"); \
    }while(0)

    /*****************************底层链表***************************/
    // 循环双向链表
    typedef struct link_list_node {
        void *data;
        struct link_list_node *pre;
        struct link_list_node *next;
    } link_list_node;
    typedef struct link_list {
        int size;

        //链表插入函数
        void (*insert)( struct link_list *list, void *data );

        //链表取第一个结点
        void *(*get_first)( struct link_list *list );

        //链表取最后一个结点
        void *(*get_last)( struct link_list *list );

        //链表删除第一个结点
        void *(*del)( struct link_list *list );

        struct link_list_node *head;
    } link_list;
    link_list_node *create_link_list_node(
        void *data )
    {
        link_list_node *node = NULL;

        node = ( link_list_node * )malloc( sizeof(link_list_node) );
        node->data = data;
        node->pre = NULL;
        node->next = NULL;

        return node;
    }
    //头插
    void insert_head(
        link_list *list,
        void *data )
    {
        if ( !list ) {
            PRINT("list is null.");
            return;
        }

        link_list_node *node = create_link_list_node( data );

        if ( !list->head ) {
            node->next = node;
            node->pre = node;
            list->head = node;
            list->size = 1;
            return;
        }

        node->next = list->head;
        node->pre = list->head->pre;
        list->head->pre->next = node;
        list->head->pre = node;
        list->head = node;

        list->size += 1;
    }
    //尾插，栈/队列的插入
    void insert_tail (
        link_list *list,
        void *data )
    {
        if ( !list ) {
            return;
        }

        link_list_node *node = create_link_list_node( data );

        if ( !list->head ) {
            node->next = node;
            node->pre = node;
            list->head = node;
            list->size = 1;
            return;
        }

        node->pre = list->head->pre;
        node->next = list->head;
        list->head->pre->next = node;
        list->head->pre = node;

        list->size += 1;
    }
    //获取链表头数据
    void *get_head(
        link_list *list )
    {
        if ( !list || !list->head ) {
            PRINT("list is null.");
            return NULL;
        }
        return list->head->data;
    }
    //获取链表尾数据
    void *get_tail(
        link_list *list )
    {
        if ( !list || !list->head ){
            PRINT("list is null.");
            return NULL;
        }
        return list->head->pre->data;
    }
    //删除链表头，并返回删除结点的数据
    void *del_head(
        link_list *list )
    {
        if ( !list || !list->head ) {
            return NULL;
        }

        if ( list->size == 1 ) {
            void *data = list->head->data;
            free( list->head );
            list->head = NULL;
            list->size = 0;
            return data;
        }

        link_list_node *p = list->head;
        void *data = p->data;

        list->head->pre->next = list->head->next;
        list->head->next->pre = list->head->pre;
        list->head = list->head->next;

        list->size -= 1;
        free( p );

        return data;
    }
    //删除链表尾，并返回删除结点的数据
    void *del_tail(
        link_list *list )
    {
        if ( !list || !list->head ) {
            PRINT("list is null.");
            return NULL;
        }

        if ( list->size == 1 ) {
            void *data = list->head->data;
            free( list->head );
            list->head = NULL;
            list->size = 0;
            return data;
        }

        link_list_node *p = list->head->pre;
        void *data = p->data;

        list->head->pre->pre->next = list->head;
        list->head->pre = list->head->pre->pre;

        list->size -= 1;
        free( p );

        return data;
    }
    /******************************************************************/

    /********************************队列***********************************/
    typedef struct queue {
        void *(*first)( struct queue *this );
        void *(*last)( struct queue *this );
        void (*enqueue)( struct queue *this, void *data );
        void *(*dequeue)( struct queue *this );
        int (*size)( struct queue *this );
        void (*free)( struct queue *this );
        struct link_list *list;
        struct queue *this;
    } queue;
    void *queue_get_first(
        queue *q )
    {
        if ( !q || !q->list )
            return NULL;

        return q->list->get_first( q->list );
    }
    void *queue_get_last(
        queue *q )
    {
        if ( !q || !q->list )
            return NULL;

        return q->list->get_last( q->list );
    }
    void enqueue(
        queue *q,
        void *data )
    {
        if ( !q || !q->list )
            return ;

        q->list->insert( q->list, data );
    }
    void *dequeue(
        queue *q )
    {
        if ( !q || !q->list )
            return NULL;

        q->list->del( q->list );
    }
    int queue_size(
        queue *q )
    {
        if ( !q || !q->list)
            return 0;
        return q->list->size;
    }
    void queue_free(
        queue *q )
    {
        if ( !q )
            return ;
        if ( !q->list ) {
            free( q );
            return;
        }

        while ( q->dequeue( q->this ) );

        free( q->list );
        free( q );
    }
    queue *create_queue()
    {
        queue *q = NULL;

        q = ( queue * )malloc( sizeof(queue) );
        q->first = queue_get_first;
        q->last = queue_get_last;
        q->enqueue = enqueue;
        q->dequeue = dequeue;
        q->size = queue_size;
        q->free = queue_free;
        q->list = ( link_list * )malloc( sizeof(link_list) );
        q->list->size = 0;
        q->list->insert = insert_tail;
        q->list->get_first = get_head;
        q->list->get_last = get_tail;
        q->list->del = del_head;
        q->this = q;
    }

    #undef DEBUG_QUEUE
    #ifdef DEBUG_QUEUE
    void test_queue()
    {
        queue *q = create_queue();
        int data1 = 5;
        int data2 = 6;
        int data3 = 7;
        q->enqueue(q->this, (void *)&data1);
        PRINT("q->last:%d", *(int *)q->last(q->this));
        q->enqueue(q->this, (void *)&data2);
        PRINT("q->last:%d", *(int *)q->last(q->this));
        q->enqueue(q->this, (void *)&data3);
        PRINT("q->last:%d\n", *(int *)q->last(q->this));

        PRINT("q->size:%d\n", q->size(q->this));

        PRINT("q->first:%d", (int)*(int *)(q->first(q->this)));
        PRINT("q->last:%d\n", (int)*(int *)(q->last(q->this)));

        PRINT("q->dequeue:%d", (int)*(int *)(q->dequeue(q->this)));
        PRINT("q->first:%d", (int)*(int *)(q->first(q->this)));
        PRINT("q->size:%d\n", q->size(q->this));

        PRINT("q->dequeue:%d", (int)*(int *)(q->dequeue(q->this)));
        PRINT("q->first:%d", (int)*(int *)(q->first(q->this)));
        PRINT("q->size:%d\n", q->size(q->this));

        PRINT("q->dequeue the last element");
        q->dequeue(q->this);
        PRINT("q->size:%d\n", q->size(q->this));

        q->free( q->this );
    }
    int main()
    {
        test_queue();

        return 0;
    }
    #endif

    /*****************************************************************/

    /*****************************栈********************************/

    typedef struct stack {
        void *(*first)( struct stack *this );
        void *(*last)( struct stack *this );
        void (*push)( struct stack *this, void *data );
        void *(*pop)( struct stack *this );
        int (*size)( struct stack *this );
        void (*free)( struct stack *this );
        struct link_list *list;
        struct stack *this;
    } stack;
    void *stack_get_first(
        stack *s )
    {
        if ( !s || !s->list )
            return NULL;

        return s->list->get_first( s->list );
    }
    void *stack_get_last(
        stack *s )
    {
        if ( !s || !s->list )
            return NULL;

        return s->list->get_last( s->list );
    }
    void stack_push(
        stack *s,
        void *data )
    {
        if ( !s || !s->list )
            return ;

        s->list->insert( s->list, data );
    }
    void *stack_pop(
        stack *s )
    {
        if ( !s || !s->list )
            return NULL;

        s->list->del( s->list );
    }
    int stack_size(
        stack *s )
    {
        if ( !s || !s->list)
            return 0;
        return s->list->size;
    }
    void stack_free(
        stack *s )
    {
        if ( !s )
            return ;
        if ( !s->list ) {
            free( s );
            return;
        }

        while ( s->pop( s->this ) );

        free( s->list );
        free( s );
    }
    stack *create_stack()
    {
        stack *s = NULL;

        s = ( stack * )malloc( sizeof(stack) );
        s->first = stack_get_first;
        s->last = stack_get_last;
        s->push = stack_push;
        s->pop = stack_pop;
        s->size = stack_size;
        s->free = stack_free;
        s->list = ( link_list * )malloc( sizeof(link_list) );
        s->list->size = 0;
        s->list->insert = insert_tail;
        s->list->get_first = get_tail;
        s->list->get_last = get_head;
        s->list->del = del_tail;
        s->this = s;
    }

    #define DEBUG_STACK
    #ifdef DEBUG_STACK
    void test_stack()
    {
        stack *s = create_stack();
        int data1 = 5;
        int data2 = 6;
        int data3 = 7;
        s->push(s->this, (void *)&data1);
        PRINT("s->first:%d", *(int *)s->first(s->this));
        s->push(s->this, (void *)&data2);
        PRINT("s->first:%d", *(int *)s->first(s->this));
        s->push(s->this, (void *)&data3);
        PRINT("s->first:%d\n", *(int *)s->first(s->this));

        PRINT("s->size:%d\n", s->size(s->this));

        PRINT("s->last:%d", (int)*(int *)(s->last(s->this)));
        PRINT("s->first:%d\n", (int)*(int *)(s->first(s->this)));

        PRINT("s->pop:%d", (int)*(int *)(s->pop(s->this)));
        PRINT("s->first:%d", (int)*(int *)(s->first(s->this)));
        PRINT("s->size:%d\n", s->size(s->this));

        PRINT("s->pop:%d", (int)*(int *)(s->pop(s->this)));
        PRINT("s->first:%d", (int)*(int *)(s->first(s->this)));
        PRINT("s->size:%d\n", s->size(s->this));

        PRINT("s->pop the first element");
        s->pop(s->this);
        PRINT("s->size:%d\n", s->size(s->this));

        s->free( s->this );
    }
    int main()
    {
        test_stack();

        return 0;
    }
    #endif