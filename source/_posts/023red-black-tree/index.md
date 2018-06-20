---
title: 红黑树-《算法导论》学习笔记十二
date: 2018-06-19 20:38:53
categories:
- 开发
tags:
- 数据结构/算法
---

红黑树是一种二叉搜索树，它在每个结点上增加了一个存储为来表示结点的颜色，或红或黑，通过从根到叶子的简单路径上各个结点的颜色进行约束，红黑树确保没有一条路径会比其它路径长出2倍，近似平衡的。

树种每个结点包含5个属性：color、key、left、right、parent，如果一个结点没有子结点或父结点，则该结点相应指针属性值指向空（这里的空不是空指针，而是定义一个空结点，结点颜色为黑色），一颗红黑树是满足几个特殊性质的二叉搜索树：


* 每个结点或是红色的，或是黑色的
* 根结点是黑色的
* 每个叶结点（亦即空结点）为黑色的
* 如果一个结点是红色的，则它的两个子结点都是黑色的
* 对每个结点，从该结点到其所有后代叶结点的简单路径上，均包含相同数目的黑色结点（经过的黑色结点数为黑高）

与二叉搜索树相似，红黑树也有一些操作接口：


* 遍历
* 寻找结点中序遍历的后继:tree_minimum
* 寻找结点中序遍历的前驱:tree_maximum
* 结点左右旋转（红黑树不同于二叉搜索树的操作），用来平衡黑高
    * 左旋：结点的父结点的左孩子指向结点的右孩子，右孩子的左孩子变为结点的右孩子，右孩子变为它的父结点
    * 右旋：结点的父结点的右孩子指向结点的左孩子，做孩子的右孩子变为结点的左孩子，左孩子变为它的父结点*
* 插入，插入一个新结点时，考虑如果插入黑色结点，破坏了原树的红黑性质（黑高），要进行重新平衡
* 删除，删除了一个结点后，考虑如果删除的是黑色结点，或删除了结点，后继提升到删除节点的位置后，平衡红黑性质

**———-插入**


1. 如果结点的父结点为黑色，表示没插入这个结点前满足红黑树
性质，且插入这个红色结点后，也不影响红黑树黑高，故保持当前位置

2. 如果结点的父结点为红色，那么判断插入结点与父结点，以及父结点的
父结点，叔结点，这四个结点是否组成一个倒立的”v”字形：

    (1). 如果是倒立”v”，则插入结点的父结点为红色，破坏了红黑树的：红
色结点的左右孩子都为黑色的性质，故父结点一定要变为黑色;
那么将父结点变为黑色;

    判断叔结点若为红色，则将叔结点变为黑色，父结点的父结点变为红色，
插入结点的指针指向父结点的父结点，再次从步骤1判断（因为经过变换，
从插入结点的父结点的父结点开始的子数都满足红黑树性质，只需要将
父结点的父结点单独当作一个新插入它的父结点的子结点再进行处理即可）;

    否则叔结点为黑色，则直接将父结点的父结点变为红色，但是这样做之后，
父结点因为变为黑色，而叔结点还是黑色，两个子树路径的黑高相差1,要
维持黑高相等，则将父结点的父结点的父结点向叔结点相同的方向旋转（
可以针对这种情况画图，会发现这样旋转后，两个黑高不一样的子树分开了，
，且满足红黑树性质）。

    (2). 如果不是倒立”v”，即插入结点的父结点所处子树为某方向上的子树，但插入
结点又是父结点另一方的孩子结点，则只需要进行一下旋转，就将结点变为
倒立的”v”形状，继续从步骤1开始作为倒立”v”子树判断。

3. 将根结点变为黑色

**———-删除**


1. 以指针p指向待删除结点node，p_color记录node颜色

2. 判断node的左右孩子
    (1). 如果左孩子为空
寻找node的右孩子，记为q（不管是否指向空）
用q结点替换node结点

    (2). 但如果右孩子为空
寻找node的左孩子，记为q（不管是否指向空）
用q结点替换node结点

    (3). 否则左右孩子都不为空
寻找node的中序遍历后继(tree_minimum())，p指向它;

    p_color记录后继的颜色;

    q指向p的右结点（不管是否为空）;

    判断p的父结点是否为node:是，则， q的父结点设为p（只有当q为空结点时有用）/否，则，用p的右结点替换p(rb_transplant(tree, p, p->right))，用p结点接管node结点右子树（p->right = node->right, p->right->parent = p）;

    用p结点替换node结点(rb_transplant(tree, node, p));

    p结点接管node结点左子树(p->left = node->left,p->left->parent = p);

    p结点的颜色设置为node结点的颜色

3. 判断p_color的颜色如果为黑色，表示删除结点后，黑高可能-1,
要修复红黑树，从结点q处开始修复红黑树，根据q的颜色来修复

4. 循环.直到q结点不为根结点以及p结点的颜色不为黑色

    (1). 如果node的兄弟结点为红色
（因为node结点替换了删除结点，黑高-1,与兄弟结点子树黑高差1）
设置兄弟结点为黑色，父结点设置为红色，从父结点开始向node一侧旋转，兄弟结点成为新的子树根结点，因为兄弟结点为黑色，则经过兄弟结点到node结点的子树黑高+1,平衡，但对于兄弟结点的父结点的子树，黑高又可能不平衡，node指针指向兄弟结点，继续步骤4

    (2). 如果node的兄弟结点为黑色，且兄弟结点的左右孩子均为黑色
（因为node结点替换了删除结点，黑高-1,与兄弟结点子树黑高差1）
设置兄弟结点颜色为红色，兄弟结点子树黑高-1,达到平衡，但父结点以上的子树可能受影响，node指针指向父结点，继续步骤4

    (3). 如果node的兄弟结点为黑色，且兄弟结点的靠近node一侧的孩子结点颜色为红色，另一侧的孩子结点为黑色
（因为node结点替换了删除结点，黑高-1,与兄弟结点子树黑高差1）
兄弟结点靠近node一侧的孩子结点设置为黑色，兄弟结点设置为红色，从兄弟结点开始进行向远离node一侧旋转，兄弟结点的孩子结点成为它的父结点，原兄弟结点的左右子树黑高不变，但node结点所处子树因为替换删除结点，黑高-1，还是没有平衡，node指针指向新的兄弟结点的孩子结点，继续4

    (4). 如果node的兄弟结点为黑色，且兄弟结点的远离node一侧的孩子结点颜色为红色，另一侧颜色未知
（因为node结点替换了删除结点，黑高-1,与兄弟结点子树黑高差1）
兄弟结点设置为node父结点的颜色，父结点设置为黑色，兄弟结点为红色的设置为黑色，然后从node父结点开始进行向node一侧旋转，旋转前，可以思考，父结点若为红色，旋转后不改变子树黑高，若父结点为黑色，旋转后也不会改变黑高经过一轮变换后，node结点的父结点的父结点左右子树黑高一样，且经过node替换后-1的黑高，因为变换，又+1,本子树在整个红黑树中没有破坏黑高一致性，故将node指针指向整个树的根结点，跳出循环
设置node指针的结点颜色为黑色

代码：

    #include <stdio.h>
    #include <stdlib.h>
    #include <string.h>
    #include <unistd.h>
    #include <time.h>

    //#define COLOR_RED   1;
    //#define COLOR_BLACK 2;



    typedef enum color {
        COLOR_RED,
        COLOR_BLACK
    }color;

    typedef struct rbTreeNode {
        int key;
        color color;
        struct rbTreeNode *parent;
        struct rbTreeNode *left;
        struct rbTreeNode *right;
    }rbTreeNode;

    typedef struct rbTree {
        int node_num;
        int height;
        rbTreeNode *root;
        rbTreeNode *null;
    }rbTree;

    void print_tree( rbTree * );

    rbTreeNode *tree_minimum(
        rbTreeNode *root,
        rbTreeNode *null )
    {
        while ( root->left != null ) {
            root = root->left;
        }

        return root;
    }

    rbTreeNode *tree_maximum(
        rbTreeNode *root,
        rbTreeNode *null )
    {
        while ( root->right != null ) {
            root = root->right;
        }

        return root;
    }


    void left_rotate(
        rbTree *tree,
        rbTreeNode *node )
    {
        rbTreeNode *p = node->right;
        node->right = p->left;

        if ( p->left != tree->null )
            p->left->parent = node;

        p->parent = node->parent;

        if ( node->parent == tree->null )
            tree->root = p;
        else if ( node == node->parent->left )
            node->parent->left = p;
        else
            node->parent->right = p;

        p->left = node;
        node->parent = p;
    }
    void right_rotate(
        rbTree *tree,
        rbTreeNode *node )
    {
        rbTreeNode *p = node->left;
        node->left = p->right;

        if ( p->right != tree->null )
            p->right->parent = node;

        p->parent = node->parent;

        if ( node->parent == tree->null )
            tree->root = p;
        else if ( node == node->parent->right )
            node->parent->right = p;
        else
            node->parent->left = p;

        p->right = node;
        node->parent = p;
    }
    void rb_insert_fixup(
        rbTree *tree,
        rbTreeNode *node )
    {
        rbTreeNode *p = tree->null;

        while ( node->parent->color == COLOR_RED ) {
            if ( node->parent == node->parent->parent->left ) {

                p = node->parent->parent->right;
                if ( p->color == COLOR_RED ) {
                    node->parent->color = COLOR_BLACK;
                    p->color = COLOR_BLACK;
                    node->parent->parent->color = COLOR_RED;
                    node = node->parent->parent;
                } else if ( node == node->parent->right ) {
                    node = node->parent;
                    left_rotate( tree, node );
                } else {
                    node->parent->color = COLOR_BLACK;
                    node->parent->parent->color = COLOR_RED;
                    right_rotate( tree, node->parent->parent );
                }

            } else {

                p = node->parent->parent->left;
                if ( p->color == COLOR_RED ) {
                    node->parent->color = COLOR_BLACK;
                    p->color = COLOR_BLACK;
                    node->parent->parent->color = COLOR_RED;
                    node = node->parent->parent;
                } else if ( node == node->parent->left ) {
                    node = node->parent;
                    right_rotate( tree, node );
                } else {
                    node->parent->color = COLOR_BLACK;
                    node->parent->parent->color = COLOR_RED;
                    left_rotate( tree, node->parent->parent );
                }

            }
        }

        tree->root->color = COLOR_BLACK;
    }
    void rb_insert(
        rbTree *tree,
        rbTreeNode *node )
    {
        rbTreeNode *p = tree->null;
        rbTreeNode *q = tree->root;

        while ( q != tree->null ) {
            p = q;
            if ( node->key < q->key )
                q = q->left;
            else
                q = q->right;
        }

        node->parent = p;

        if ( p == tree->null )
            tree->root = node;
        else if ( node->key < p->key )
            p->left = node;
        else
            p->right = node;

        node->left = tree->null;
        node->right = tree->null;
        node->color = COLOR_RED;
        rb_insert_fixup( tree, node );
    }
    void rb_transplant(
        rbTree *tree,
        rbTreeNode *node_a,
        rbTreeNode *node_b )
    {
        if ( node_a->parent == tree->null )
            tree->root = node_b;
        else if ( node_a == node_a->parent->left )
            node_a->parent->left = node_b;
        else
            node_a->parent->right = node_b;

        node_b->parent = node_a->parent;
        free( node_a );
    }
    void rb_delete_fixup(
        rbTree *tree,
        rbTreeNode *node )
    {
        rbTreeNode *p = tree->null;

        while ( node != tree->null && node->color == COLOR_BLACK ) {
            if ( node == node->parent->left ) {
                p = node->parent->right;
                if ( p->color == COLOR_RED ) {
                    p->color = COLOR_BLACK;
                    node->parent->color = COLOR_RED;
                    left_rotate( tree, node->parent );
                    p = node->parent->right;
                } else if ( p->left->color == COLOR_BLACK
                    && p->right->color == COLOR_BLACK ) {
                    p->color = COLOR_RED;
                    node = node->parent;
                } else if ( p->right->color == COLOR_BLACK ) {
                    p->left->color = COLOR_BLACK;
                    p->color = COLOR_RED;
                    right_rotate( tree, p );
                    p = node->parent->right;
                } else {
                    p->color = node->parent->color;
                    node->parent->color = COLOR_BLACK;
                    left_rotate( tree, node->parent );
                    node = tree->root;
                }
            } else {
                p = node->parent->left;
                if ( p->color == COLOR_RED ) {
                    p->color = COLOR_BLACK;
                    node->parent->color = COLOR_RED;
                    right_rotate( tree, node->parent );
                    p = node->parent->left;
                } else if ( p->left->color == COLOR_BLACK
                    && p->right->color == COLOR_BLACK ) {
                    p->color = COLOR_RED;
                    node = node->parent;
                } else if ( p->left->color == COLOR_BLACK ) {
                    p->right->color = COLOR_BLACK;
                    p->color = COLOR_RED;
                    left_rotate( tree, p );
                    p = node->parent->left;
                } else {
                    p->color = node->parent->color;
                    node->parent->color = COLOR_BLACK;
                    right_rotate( tree, node->parent );
                    node = tree->root;
                }
            }
        }
        node->color = COLOR_BLACK;
    }
    void rb_delete(
        rbTree *tree,
        rbTreeNode *node )
    {
        rbTreeNode *p = node;
        rbTreeNode *q = tree->null;
        color origin_color = p->color;

        if ( node->left == tree->null ) {
            q = node->right;
            rb_transplant( tree, node, node->right );
        } else if ( node->right == tree->null ) {
            q = node->left;
            rb_transplant( tree, node, node->left );
        } else {
            p = tree_minimum( node->right, tree->null );
            origin_color = p->color;
            q = p->right;

            if ( p->parent == node )
                q->parent = p;
            else {
                rb_transplant( tree, p, p->right );
                p->right = node->right;
                p->right->parent = p;
            }

            rb_transplant( tree, node, p );
            p->left = node->left;
            p->left->parent = p;
            p->color = node->color;
        }

        if ( origin_color == COLOR_BLACK )
            rb_delete_fixup( tree, q );
    }
    int random_num()
    {
        int a = 1;
        int b = 100;

        return rand() % ( b - a ) + a;
    }
    void postorder_tree_free(
        rbTreeNode *root,
        rbTreeNode *null )
    {
        if ( root != null ) {
            postorder_tree_free( root->left, null );
            postorder_tree_free( root->right, null );
            free( root );
        }
    }
    void inorder_tree_walk(
        rbTreeNode *root,
        rbTreeNode *null )
    {
        if ( root != null ) {
            inorder_tree_walk( root->left, null );
            printf("%d(%d) ", root->key, root->color);
            inorder_tree_walk( root->right, null );
        }
    }
    void preorder_tree_walk(
        rbTreeNode *root,
        rbTreeNode *null )
    {
        if ( root != null ) {
            printf("%d(%d) ", root->key, root->color);
            preorder_tree_walk( root->left, null );
            preorder_tree_walk( root->right, null );
        }
    }
    void postorder_tree_walk(
        rbTreeNode *root,
        rbTreeNode *null )
    {
        if ( root != null ) {
            postorder_tree_walk( root->left, null );
            postorder_tree_walk( root->right, null );
            printf("%d(%d) ", root->key, root->color);
        }
    }
    void print_tree(
        rbTree *tree )
    {
        printf("中序:");
        inorder_tree_walk( tree->root, tree->null );
        printf("\n");
        printf("前序:");
        preorder_tree_walk( tree->root, tree->null );
        printf("\n");
        printf("后序:");
        postorder_tree_walk( tree->root, tree->null );
        printf("\n\n");
    }
    rbTreeNode *tree_search_recursion(
        rbTreeNode *root,
        int key,
        rbTreeNode *null)
    {
        if ( root == null || key == root->key ) {
            return root;
        }

        if ( key < root->key )
            return tree_search_recursion( root->left, key, null );
        else
            return tree_search_recursion( root->right, key, null );
    }

    rbTreeNode *tree_search(
        rbTreeNode *root,
        int key,
        rbTreeNode *null )
    {
        while ( root != null && key != root->key ) {
            if ( key < root->key )
                root = root->left;
            else
                root = root->right;
        }

        return root;
    }
    // 找前驱 即中序遍历的前一个位置值
    rbTreeNode *tree_predecessor(
        rbTreeNode *node,
        rbTreeNode *null )
    {
        if ( node->left != null )
            return tree_maximum( node->left, null );

        rbTreeNode *p = node->parent;

        while ( p != null && node == p->left ) {
            node = p;
            p = p->parent;
        }

        return p;
    }
    // 找后继 即中序遍历的后一个位置值
    rbTreeNode *tree_successor(
        rbTreeNode *node,
        rbTreeNode *null )
    {
        if ( node->right != null )
            return tree_minimum( node->right, null );

        rbTreeNode *p = node->parent;

        while ( p != null && node == p->right ) {
            node = p;
            p = p->parent;
        }

        return p;
    }
    void free_tree(
        rbTree *tree )
    {
        rbTreeNode *root = tree->root;

        postorder_tree_free( root, tree->null );

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

        rbTree *tree = ( rbTree * )malloc( sizeof(rbTree) );
        tree->node_num = 0;
        tree->height = 0;
        tree->null = ( rbTreeNode * )malloc( sizeof(rbTreeNode) );
        tree->null->color = COLOR_BLACK;
        tree->root = tree->null;

        for ( i = 0; i < len; i++ ) {
            rbTreeNode *node =
                ( rbTreeNode * )malloc( sizeof(rbTreeNode) );
            //node->key = random_num();
            node->key = arr[i];
            node->parent = tree->null;
            node->left = tree->null;
            node->right = tree->null;
            rb_insert( tree, node );
            printf("insert %d successfully\n", arr[i]);
        }

        print_tree( tree );

        rbTreeNode *delete_node = tree_search( tree->root, 4, tree->null );

        if ( delete_node != tree->null ) {
            rb_delete( tree, delete_node );
        }
        printf("\n删除%d的节点后遍历顺序为:\n", 4);
        print_tree( tree );
        printf("\n");

        delete_node = tree_search( tree->root, 15, tree->null );
        if ( delete_node != tree->null ) {
            rb_delete( tree, delete_node );
        }
        printf("\n删除%d的节点后中序遍历顺序为:\n", 15);
        print_tree( tree );
        printf("\n");

        free_tree( tree );

        return 0;
    }