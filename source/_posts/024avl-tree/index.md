---
title: avl树-《算法导论》学习笔记十三
date: 2018-06-20 10:55:19
categories:
- 开发
tags:
- 数据结构/算法
---

（引用算法导论）AVL树是一种高度平衡的二叉搜索树：对每一个结点x，y的左子树与右子树的高度至多为1。AVL树相比二叉搜索树，每个结点维护一个额外的属性：结点的高度。

AVL树实现了几个操作：

# 树结点创建
# 遍历
>递归的前序、中序、后序遍历，以及基于层序遍历的简单图形打印
# 后序遍历释放树结点
# 搜索
# 寻找结点子树最小关键字结点、寻找结点子树最大关键字结点
# 求结点高度
# 左旋
# 右旋
# 插入


>先以普通二叉搜索树的方式插入结点；
>
>插入结点后可能影响平衡（一个子树高度-另一子树高度等于2），具体的不平衡的情况分四种：
>1. 结点左子树高度-右子树高度=2，且结点左孩子的左子树更高
>2. 结点左子树高度-右子树高度=2，且结点右孩子的右子树更高
>3. 结点右子树高度-左子树高度=2，且结点右孩子的右子树更高
>4. 结点右子树高度-左子树高度=2，且结点右孩子的左子树更高
>
>对于情况1，只需对结点进行右旋即可重新平衡；
>
>对于情况3，只需对结点进行左旋即可重新平衡；
>
>对于情况2，需要先对结点的左孩子进行左旋，然后对结点进行右旋即可平衡；
>
>对于情况4，需要先对结点的右孩子进行右旋，然后对结点进行左旋即可平衡；
>
>旋转之后，结点的左子树与右子树达到平衡，但结点父结点的树可能不平衡，
>
>需要循环向根节点判断结点高度有无平衡，直至根节点。

# 删除

>1. 先查询相同的key的结点；
>
>2. 找到待删除结点，如果待删除结点的左孩子与右孩子都不为空，则判断左右孩子的树高：
>
>* 若左子树更高，则将左子树的最大关键字结点的关键字替换掉待删除结点的关键字，然后再删除那个左子树最大关键字结点
>* 若右子树与左子树同高或更高，则将右子树的最小关键字结点的关键字替换掉待删除节点的关键字，然后再删除那个右子树最小关键字结点
>
>3. 删除结点后，会影响树平衡，具体不平衡情况与插入时相同，依然进行相同操作来重新平衡树

#

代码:

    #include <stdio.h>
    #include <stdlib.h>
    #include <string.h>
    #include <unistd.h>
    #include <time.h>
    #include <math.h>


    #define MAX(a, b) ((a > b) ? (a) : (b))

    typedef struct avlTreeNode {
        int key;
        int height;

        //这里有父结点，有了父结点多很多代码，
        //大部分的调试都是父结点的值没注意，
        //可取消这个字段
        struct avlTreeNode *parent;
        struct avlTreeNode *left;
        struct avlTreeNode *right;
    }avlTreeNode;

    typedef struct avlTree {
        int node_num;
        avlTreeNode *root;
    }avlTree;

    // 随机基数尽量大，代码不支持key值相同的
    // 树节点，但不影响插入，删除会出错，所以
    // 尽量保证不同
    int random_num()
    {
        int a = 1;
        int b = 10000;

        return rand() % ( b - a ) + a;
    }
    avlTreeNode *tree_create_node(
        int key,
        avlTreeNode *parent,
        avlTreeNode *left,
        avlTreeNode *right )
    {
        avlTreeNode *node = ( avlTreeNode * )malloc( sizeof(avlTreeNode) );
        node->height = 1;
        node->key = key;
        node->parent = parent;
        node->left = left;
        node->right = right;

        return node;
    }
    //后续遍历释放树结点
    void postorder_tree_free(
        avlTreeNode *root )
    {
        if ( root != NULL ) {
            postorder_tree_free( root->left );
            postorder_tree_free( root->right );
            printf("%d ", root->key);
            free( root );
        }
    }
    void inorder_tree_walk(
        avlTreeNode *root )
    {
        if ( root != NULL ) {
            inorder_tree_walk( root->left );
            printf("%d(%d) ", root->key, root->height);
            inorder_tree_walk( root->right );
        }
    }
    void preorder_tree_walk(
        avlTreeNode *root )
    {
        if ( root != NULL ) {
            printf("%d(%d) ", root->key, root->height);
            preorder_tree_walk( root->left );
            preorder_tree_walk( root->right );
        }
    }
    void postorder_tree_walk(
        avlTreeNode *root )
    {
        if ( root != NULL ) {
            postorder_tree_walk( root->left );
            postorder_tree_walk( root->right );
            printf("%d(%d) ", root->key, root->height);
        }
    }
    //层序遍历，支持简单的树形打印
    //为了调试代码，而不用中序和前序
    //去推算树
    int queue_pre = -1;
    int queue_post = 0;
    avlTreeNode *node_queue[1000] = {NULL};
    int factor( int n )
    {
        if ( n == 1 )
            return 1;
        return (pow(2,n-1) + factor( n - 1 ));
    }
    void graph_tree_walk(
        avlTreeNode *root,
        int height,
        int root_height )
    {
        if ( queue_pre == factor( root_height ) - 1 )
            return ;

        queue_pre++;

        if ( root == NULL ) {
            printf("* ");
            node_queue[queue_post] = NULL;
            queue_post++;
            node_queue[queue_post] = NULL;
            queue_post++;
        } else {
            node_queue[queue_post] = root->left;
            queue_post++;
            node_queue[queue_post] = root->right;
            queue_post++;

            printf("%d ", root->key);
        }
        int new_height = height;

        if ( queue_pre == factor( height ) - 1 ) {
            printf("\n");
            new_height = height + 1;
        }
        graph_tree_walk( node_queue[queue_pre], new_height, root_height );
    }
    void print_tree(
        avlTree *tree )
    {
        printf("中序:");
        inorder_tree_walk( tree->root );
        printf("\n");
        printf("前序:");
        preorder_tree_walk( tree->root );
        printf("\n");
        printf("后序:");
        postorder_tree_walk( tree->root );
        printf("\n");
        printf("图形:\n");
        graph_tree_walk( tree->root, 1, tree->root->height );
        queue_pre = -1;
        queue_post = 0;

        int i = 999;

        for ( i ; i >=0; i-- )
            node_queue[i] = NULL;

        printf("\n\n");
    }
    //检查avl树的树高是否差2以内
    int check_is_avltree(
        avlTreeNode *root )
    {
        if ( root == NULL )
            return 0;

        int left_height = check_is_avltree( root->left );
        int right_height = check_is_avltree( root->right );

        if ( left_height - right_height >= 2 || right_height - left_height >= 2 )
            printf("it's not a avl tree!!!! l:%d, r:%d\n", left_height, right_height);
        else
            //printf("left:%d,right:%d\n", left_height, right_height);

        return (left_height > right_height ? left_height : right_height ) + 1;
    }
    avlTreeNode *tree_search_recursion(
        avlTreeNode *root,
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

    avlTreeNode *tree_search(
        avlTreeNode *root,
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

    avlTreeNode *tree_minimum(
        avlTreeNode *root )
    {
        while ( root->left != NULL ) {
            root = root->left;
        }

        return root;
    }

    avlTreeNode *tree_maximum(
        avlTreeNode *root )
    {
        while ( root->right != NULL ) {
            root = root->right;
        }

        return root;
    }

    int node_height(
        avlTreeNode *node )
    {
        if ( node == NULL )
            return 0;

        return node->height;
    }
    int tree_height(
        avlTree *tree )
    {
        return node_height( tree->root );
    }

    avlTreeNode *left_rotate(
        avlTreeNode *node )
    {
        if ( node != NULL && node->right != NULL ) {
            avlTreeNode *p = node->right->left;
            if ( node->parent == NULL ) {
                node->right->parent = NULL;
            } else if ( node == node->parent->left ) {
                node->parent->left = node->right;
                node->right->parent = node->parent;
            } else {
                node->parent->right = node->right;
                node->right->parent = node->parent;
            }

            node->parent = node->right;
            node->right->left = node;
            node->right = p;

            if ( p != NULL )
                p->parent = node;

            node->height = MAX( node_height(node->left),
                                node_height(node->right) ) + 1;

            node->parent->height = MAX( node_height(node->parent->left),
                                        node_height(node->parent->right) ) + 1;

            return node->parent;
        }

        return node;
    }
    avlTreeNode *right_rotate(
        avlTreeNode *node )
    {
        if ( node != NULL && node->left != NULL ) {
            avlTreeNode *p = node->left->right;
            if ( node->parent == NULL ) {
                node->left->parent = NULL;
            } else if ( node == node->parent->left ) {
                node->parent->left = node->left;
                node->left->parent = node->parent;
            } else {
                node->parent->right = node->left;
                node->left->parent = node->parent;
            }

            node->parent = node->left;
            node->left->right = node;
            node->left = p;

            if ( p != NULL )
                p->parent = node;

            node->height = MAX( node_height(node->left),
                                node_height(node->right) ) + 1;

            node->parent->height = MAX( node_height(node->parent->right),
                                        node_height(node->parent->left) ) + 1;

            return node->parent;
        }


        return node;
    }
    avlTreeNode *tree_insert1(
        avlTreeNode *root,
        avlTreeNode *node )
    {
        if ( root == NULL ) {
            node->height = 1;
            return node;
        } else if ( node->key < root->key ) {
            root->left = tree_insert1( root->left, node );
            root->left->parent = root;
            if ( node_height( root->left ) - node_height( root->right ) == 2 ) {
                if ( node->key < root->left->key ) {
                    // right rotate;
                    root = right_rotate( root );
                } else {
                    // left right rotate;
                    root->left = left_rotate( root->left );
                    root = right_rotate( root );
                }
            }
            root->height = MAX( node_height(root->left), node_height(root->right) ) + 1;
        } else {

            root->right = tree_insert1( root->right, node );
            root->right->parent = root;
            if ( node_height( root->right ) - node_height( root->left ) == 2 ) {
                if ( node->key >= root->right->key ) {
                    root = left_rotate( root );
                } else {
                    root->right = right_rotate( root->right );
                    root = left_rotate( root );
                }
            }
            root->height = MAX( node_height(root->right), node_height(root->left) ) + 1;
        }

        return root;
    }
    void tree_insert(
        avlTree *tree,
        int key )
    {
        avlTreeNode *node = tree_create_node( key, NULL, NULL, NULL );
        if ( tree->root == NULL ) {
            tree->root = node;
        } else {
            tree->root = tree_insert1( tree->root, node );
        }
    }
    avlTreeNode *tree_delete1(
        avlTreeNode *root,
        int key )
    {
        if ( root == NULL )
            return NULL;

        if ( key < root->key ) {
            root->left = tree_delete1( root->left, key );
            if ( node_height(root->right) - node_height(root->left) == 2 ) {
                if ( node_height(root->right->left) > node_height(root->right->right) ) {
                    root->right = right_rotate( root->right );
                    root = left_rotate( root );
                } else {
                    root = left_rotate( root );
                }
            }
        } else if ( key > root->key ) {
            root->right = tree_delete1( root->right, key );
            if ( node_height(root->left) - node_height(root->right) == 2 ) {
                if ( node_height(root->left->right) > node_height(root->left->left) ) {
                    root->left = left_rotate( root->left );
                    root = right_rotate( root );
                } else {
                    root = right_rotate( root );
                }
            }
        } else {

            if ( root->left != NULL && root->right != NULL ) {

                avlTreeNode *replace = NULL;

                // 这里做了替换后要删除最大值结点，但匹配无法确认同key值的不同结点
                // ，因此树不能有相同key值的结点，但插入没有这个问题
                if ( node_height(root->left) > node_height(root->right) ) {
                    replace = tree_maximum( root->left );
                    root->key = replace->key;
                    root->left = tree_delete1( root->left, replace->key );
                } else {
                    replace = tree_minimum( root->right );
                    root->key = replace->key;
                    root->right = tree_delete1( root->right, replace->key );
                }
            } else {
                avlTreeNode *delete = root;
                if ( root->left ) {
                    root->left->parent = root->parent;
                    root = delete->left;
                } else if ( root->right ) {
                    root->right->parent = root->parent;
                    root = delete->right;
                } else {
                    root = NULL;
                }
                free( delete );
            }
        }

        if ( root != NULL )
            root->height = MAX( node_height(root->left), node_height(root->right) ) + 1;

        return root;
    }
    void tree_delete(
        avlTree *tree,
        int key )
    {
        if ( tree->root != NULL ) {
            tree->root = tree_delete1( tree->root, key );
        }
    }
    void free_tree(
        avlTree *tree )
    {
        printf("\ndelete tree node...\n");
        postorder_tree_free( tree->root );
        tree->root = NULL;
        printf("\ndelete tree...\n");
        free( tree );
        printf("free tree over!\n");
    }
    int main(
        int argc,
        char **argv )
    {
        srand((int)time(NULL));

        int i = 10;
        int len = 20;
        int arr[10] = {10,4,15,14,5, 2,8,13,1,19};

        avlTree *tree = ( avlTree * )malloc( sizeof(avlTree) );


        int delete_key_a, delete_key_b, delete_key_c;
        for ( i = 0; i < len; i++ ) {
            //tree_insert( tree, arr[i] );
            int randomnum = random_num();
            if ( i == 2 )
                delete_key_a = randomnum;
            if ( i == 5 )
                delete_key_b = randomnum;
            if ( i == 9 )
                delete_key_c = randomnum;
            //printf("randon num:%d\n", randomnum);
            tree_insert( tree, randomnum );
        }

        print_tree( tree );
        check_is_avltree( tree->root );

        printf("删除结点%d\n", delete_key_a);
        tree_delete( tree, delete_key_a );
        print_tree( tree );
        check_is_avltree( tree->root );

        tree_delete( tree, delete_key_b );
        printf("删除结点%d\n", delete_key_b );
        print_tree( tree );
        check_is_avltree( tree->root );

        tree_delete( tree, delete_key_c );
        printf("删除结点%d\n", delete_key_c );
        print_tree( tree );
        check_is_avltree( tree->root );


        printf("\n===========================================free================================================\n");
        free_tree( tree );

        return 0;
    }