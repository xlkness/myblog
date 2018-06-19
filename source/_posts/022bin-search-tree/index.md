---
title: 二叉搜索树-《算法导论》学习笔记十一
date: 2018-06-19 20:36:13
categories:
- 开发
tags:
- 数据结构/算法
---

二叉搜索树是以一颗二叉树来组织的，每个节点除数据外，还包括三个分别指向父结点、左孩子、右孩子的指针，二叉搜索树有个特性：某个结点root的左子树的某个节点x的关键值小于等于root结点右子树某个结点y的关键值。

二叉搜索树有几个操作：

**1、查找**

查找与给定关键值相等的结点

**2、遍历**

前序、中序、后序遍历输出

**3、从某结点出发，寻找子树中最小关键值的结点**

**4、从某结点出发，寻找子树中最大关键值的结点**

**5、以某种遍历方式的次数，寻找某结点的前驱结点和后继结点**

例如中序遍历的顺序为123456，则4结点的前驱为3，后继为5

**6、插入结点**

**7、删除结点**

删除某个结点后，要把它的后继结点补在删除的位置上，要注意：


* 如果结点没有孩子结点，那么只是简单地将它删除，并删改它的父结点的孩子指针指向它
* 如果结点只有一个孩子，那么将它孩子提升到它的位置，并修改它的父结点的孩子指针指向它

* 如果节点有两个孩子，那么寻找它的后继（按中序遍历来说一定在右子树），并让后继占据它的位置，后继（按中序遍历来说一定没有左子树）的子树提升到后继的位置

* 情况如上，具体删除时如何替换，又有不同情况：

* 如果结点只有左或孩子，用孩子替换结点
* 如果结点有左右两个孩子，那么要查找结点的后继：(1)、如果后继是结点的右孩子，用后继替换结点，并留下后继的右孩子；(2)、后继位于结点的右子树，但并不是结点的右孩子，则，先用后继的右孩子替换后继，再用后继替换结点

代码：

    #include <stdio.h>
    #include <stdlib.h>
    #include <string.h>
    #include <unistd.h>
    #include <time.h>

    typedef struct biSearchTreeNode {
        int key;
        struct biSearchTreeNode *parent;
        struct biSearchTreeNode *left;
        struct biSearchTreeNode *right;
    }biSearchTreeNode;

    typedef struct biSearchTree {
        int node_num;
        int height;
        biSearchTreeNode *root;
    }biSearchTree;

    int random_num()
    {
        int a = 1;
        int b = 100;

        return rand() % ( b - a ) + a;
    }
    void postorder_tree_free(
        biSearchTreeNode *root )
    {
        if ( root != NULL ) {
            postorder_tree_free( root->left );
            postorder_tree_free( root->right );
            free( root );
        }
    }
    void inorder_tree_walk(
        biSearchTreeNode *root )
    {
        if ( root != NULL ) {
            inorder_tree_walk( root->left );
            printf("%d ", root->key);
            inorder_tree_walk( root->right );
        }
    }
    void preorder_tree_walk(
        biSearchTreeNode *root )
    {
        if ( root != NULL ) {
            printf("%d ", root->key);
            preorder_tree_walk( root->left );
            preorder_tree_walk( root->right );
        }
    }
    void postorder_tree_walk(
        biSearchTreeNode *root )
    {
        if ( root != NULL ) {
            postorder_tree_walk( root->left );
            postorder_tree_walk( root->right );
            printf("%d ", root->key);
        }
    }
    void print_tree(
        biSearchTree *tree )
    {
        printf("中序:");
        inorder_tree_walk( tree->root );
        printf("\n");
        printf("前序:");
        preorder_tree_walk( tree->root );
        printf("\n");
        printf("后序:");
        postorder_tree_walk( tree->root );
        printf("\n\n");
    }
    biSearchTreeNode *tree_search_recursion(
        biSearchTreeNode *root,
        int key )
    {
        if ( root == NULL || key == root->key ) {
            return root;
        }

        if ( key < root->key )
            return tree_search_recursion( root->left, key );
        else
            return tree_search_recursion( root->right, key );
    }

    biSearchTreeNode *tree_search(
        biSearchTreeNode *root,
        int key )
    {
        while ( root != NULL && key != root->key ) {
            if ( key < root->key )
                root = root->left;
            else
                root = root->right;
        }

        return root;
    }

    biSearchTreeNode *tree_minimum(
        biSearchTreeNode *root )
    {
        while ( root->left != NULL ) {
            root = root->left;
        }

        return root;
    }

    biSearchTreeNode *tree_maximum(
        biSearchTreeNode *root )
    {
        while ( root->right != NULL ) {
            root = root->right;
        }

        return root;
    }

    // 找前驱 即中序遍历的前一个位置值
    biSearchTreeNode *tree_predecessor(
        biSearchTreeNode *node )
    {
        if ( node->left != NULL )
            return tree_maximum( node->left );

        biSearchTreeNode *p = node->parent;

        while ( p != NULL && node == p->left ) {
            node = p;
            p = p->parent;
        }

        return p;
    }
    // 找后继 即中序遍历的后一个位置值
    biSearchTreeNode *tree_successor(
        biSearchTreeNode *node )
    {
        if ( node->right != NULL )
            return tree_minimum( node->right );

        biSearchTreeNode *p = node->parent;

        while ( p != NULL && node == p->right ) {
            node = p;
            p = p->parent;
        }

        return p;
    }
    void tree_insert(
        biSearchTree *tree,
        biSearchTreeNode *node )
    {
        biSearchTreeNode *root = tree->root;
        biSearchTreeNode *p = NULL;
        biSearchTreeNode *q = root;

        while ( q != NULL ) {
            p = q;

            if ( node->key < q->key )
                q = q->left;
            else
                q = q->right;
        }

        node->parent = p;

        if ( p == NULL ) {
            tree->root = node;
        }
        else if ( node->key < p->key )
            p->left = node;
        else
            p->right = node;
    }
    void transplant(
        biSearchTree *tree,
        biSearchTreeNode *node_a,
        biSearchTreeNode *node_b )
    {
        if ( node_a->parent == NULL )
            tree->root = node_b;
        else if ( node_a == node_a->parent->left )
            node_a->parent->left = node_b;
        else
            node_a->parent->right = node_b;

        if ( node_b != NULL )
            node_b->parent = node_a->parent;

    }
    void tree_delete(
        biSearchTree *tree,
        biSearchTreeNode *node )
    {
        if ( node->left == NULL )
            transplant( tree, node, node->right );
        else if ( node->right == NULL )
            transplant( tree, node, node->left );
        else {
            biSearchTreeNode *p = tree_minimum( node->right );
            if ( p->parent != node ) {
                transplant( tree, p, p->right );
                p->right = node->right;
                p->right->parent = p;
            }
            transplant( tree, node, p );
            p->left = node->left;
            p->left->parent = p;
        }

        free( node );

    }
    void free_tree(
        biSearchTree *tree )
    {
        biSearchTreeNode *root = tree->root;

        postorder_tree_free( root );

        free( tree );
    }
    int main(
        int argc,
        char **argv )
    {
        srand((int)time(NULL));

        int i = 10;
        int len = 10;
        int arr[10] = {10,4,15,14,5, 2,8,13,1,19};

        biSearchTree *tree = ( biSearchTree * )malloc( sizeof(biSearchTree) );
        tree->node_num = 0;
        tree->height = 0;

        for ( i = 0; i < len; i++ ) {
            biSearchTreeNode *node =
                ( biSearchTreeNode * )malloc( sizeof(biSearchTreeNode) );
            //node->key = random_num();
            node->key = arr[i];
            node->parent = NULL;
            node->left = NULL;
            node->right = NULL;
            tree_insert( tree, node );
        }

        print_tree( tree );

        for ( i = 0; i < len; i++ ) {
            biSearchTreeNode *pre = tree_predecessor( tree_search(tree->root, arr[i]) );
            biSearchTreeNode *post = tree_successor( tree_search(tree->root, arr[i]) );

            printf("\n");
            if ( pre != NULL )
                printf("查找%d的中序遍历前驱为:%d\n", arr[i], pre->key);
            else
                printf("查找%d的中序遍历前驱为空！\n", arr[i]);

            if ( post != NULL )
                printf("查找%d的中序遍历后继为:%d\n", arr[i], post->key);
            else
                printf("查找%d的中序遍历后继为空！\n", arr[i]);

        }


        biSearchTreeNode *delete_node = tree_search( tree->root, 4 );

        if ( delete_node != NULL ) {
            tree_delete( tree, delete_node );
        }
        printf("\n删除%d的节点后中序遍历顺序为:", 4);
        inorder_tree_walk( tree->root );
        printf("\n");

        delete_node = tree_search( tree->root, 15 );
        if ( delete_node != NULL ) {
            tree_delete( tree, delete_node );
        }
        printf("\n删除%d的节点后中序遍历顺序为:", 15);
        inorder_tree_walk( tree->root );
        printf("\n");

        free_tree( tree );

        return 0;
    }