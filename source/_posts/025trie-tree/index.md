---
title: trie树-《算法导论》学习笔记十四
date: 2018-06-20 12:34:50
categories:
- 开发
tags:
- 数据结构/算法
---



引用一下百度百科的话吧：
Trie树，又称单词查找树，是一种树形结构，是一种哈希树的变种。典型应用是用于统计，排序和保存大量的字符串（但不仅限于字符串），所以经常被搜索引擎系统用于文本词频统计。它的优点是：利用字符串的公共前缀来减少查询时间，最大限度地减少无谓的字符串比较，查询效率比哈希树高。

这里构建了一棵字典树，每个结点有52个孩子指针，对应26个小写字母和26个大写字母，根节点不存储数据，一个单词从第一个字母开始经由根结点走对应分支进行插入和统计。

trie树结点卫星数据包含了字母、出现次数、是否构成一个单词，孩子指针就是一个52大小的trie树结点指针数组。

实现了几个操作：

# 插入单词
>遍历每个字母，从根结点出发，如果结点对应字母的孩子结点为空，就创建结点，出现次数为1，如果存在这个结点，出现次数就+1，并且如果单词结束，结束处的结点是否构成一个单词字段标识为构成
# 遍历树，并打印所有单词和每个单词出现次数
#  统计树，按给定的数字统计出现次数前几的单词
>树统计，与遍历类似，用尾递归，并传入一个大于单词最大长度的数组来存储每个分支的单词，如果遇到结点能构成一个单词，就判断你单词个数，并以插入排序的方式插入创建的统计链表（类似打扑克的插排序）；
 统计链表有更新操作，根据输入的统计前几的数字来维护这个链表该去掉哪些结点，该更新哪些结点的顺序等

##
获取单词来源为编写的一个简单单词随机生成代码，写入一个文件中，可指定单词最大长度，全大写/全小写/大小写均有，单词个数，单词范围（只支持a-*或A-*，例如5，就是生成a-e/A-E的单词）
贴代码：
**随机生成单词**

    #include <stdio.h>
    #include <sys/types.h>
    #include <sys/stat.h>
    #include <fcntl.h>
    #include <time.h>
    #include <unistd.h>
    #include <errno.h>
    #include <string.h>
    #include <stdlib.h>

    int word_len = 0;
    int upper_low = 65;
    int lowwer_low = 97;

    // if letter_size = 5
    // it will generate a-e or A-E letter.
    int letter_size = 0;

    int random_letter()
    {
        return rand() % letter_size;
    }
    int random_word(
        char *word,
        int opt )
    {
        // minimum word'length is 3.
        int true_word_len = rand() % word_len + 3;
        int true_word_len1 = true_word_len;

        while ( true_word_len-- ) {
            char letter = 0;

            if ( opt == 0 ) {
                letter = random_letter() + lowwer_low;
            } else if ( opt == 1 ) {
                letter = random_letter() + upper_low;
            } else {
                int opt_case = rand() % 2;
                if ( opt_case == 0 )
                    letter = random_letter() + lowwer_low;
                else
                    letter = random_letter() + upper_low;
            }
            word[true_word_len] = letter;
        }

        return true_word_len1;
    }
    void gen_word(
        int fd,
        int word_num,
        int opt )
    {
        char word[20] = {0};
        int true_word_len = 0;

        while ( word_num-- ) {
            memset( word, 0, 20);
            true_word_len = random_word( word, opt );
            word[true_word_len] = '\n';
            write( fd, word, true_word_len + 1 );
        }
    }
    int main(
        int argc,
        char **argv )
    {
        srand((int)time(NULL));
        if ( argc != 5 ) {
            printf("please input "
                    "word's length & "
                    "words' number & "
                    "word's range & "
                    "gen_case(0:lowwer case,1:upper case,other:both\n");
            exit( 0 );
        }

        word_len = atoi( argv[1] );
        int word_num = atoi( argv[2] );
        letter_size = atoi( argv[3] );
        int opt = atoi( argv[4] );

        int fd = open("word.txt", O_RDWR | O_TRUNC, 0777);

        gen_word( fd, word_num, opt );

        close( fd );

        return 0;
    }

**trie树**

    #include <stdio.h>
    #include <unistd.h>
    #include <stdlib.h>
    #include <string.h>
    #include <sys/types.h>
    #include <sys/stat.h>
    #include <fcntl.h>
    #include <errno.h>

    #define MAX_CHILD_NUM 52
    #define UPPER_LOW 65
    #define UPPER_UP 90
    #define LOWER_LOW 97
    #define LOWER_UP 122

    #define PRINT(format, arg...) \
    do { \
        printf("[%s/%d]:", __func__, __LINE__); \
        printf(format, ##arg); \
        printf("\n"); \
    }while(0)

    typedef struct trieTreeNode {
        char letter;
        int count;
        int is_word;
        struct trieTreeNode *next[MAX_CHILD_NUM];
    } trieTreeNode;

    typedef struct trieTree {
        trieTreeNode *root;
    } trieTree;

    typedef struct count_data {
        int order;
        int count;
        char string[20];
        struct count_data *next;
    } count_data;

    int trans_letter_2_index(
        char letter )
    {
        int index = -1;
        if ( letter >= LOWER_LOW
            && letter <= LOWER_UP ) {
            index = letter - LOWER_LOW + 26;
        } else if ( letter >= UPPER_LOW
            && letter <= UPPER_UP ) {
            index = letter - UPPER_LOW;
        } else {
            PRINT("error letter input:%c", letter);
            exit( 0 );
        }

        return index;


    }
    trieTreeNode *create_node(
        char letter )
    {
        trieTreeNode *node =
            ( trieTreeNode * )calloc( 1, sizeof(trieTreeNode) );
        node->letter = letter;
        node->count = 0;
        node->is_word = 0;
    }
    void insert(
        trieTreeNode *root,
        char *word )
    {
        if ( root == NULL ) {
            PRINT("root node is null.");
            return;
        }
        int i = 0;
        trieTreeNode *cur = root;

        for ( i; word[i] != '\0'; i++ ) {
            int next_index = trans_letter_2_index(word[i]);
            //PRINT("letter:%c, index:%d", word[i], next_index);
            if ( cur->next[next_index] == NULL ) {
                cur->next[next_index] = create_node( word[i] );
            } else {
                //cur->next[next_index]->count += 1;
            }
            if ( word[i+1] == '\0' ) {
                cur->next[next_index]->count += 1;
                cur->next[next_index]->is_word = 1;
            }
            cur = cur->next[next_index];
        }
    }
    // 删除链表所有结点
    void delete_list_all_node(
        count_data *node )
    {
        count_data *p = NULL;
        while ( node ) {
            p = node;
            node = node->next;
            free( p );
        }
    }
    void print_list_all_node(
        count_data *node )
    {
        printf("\n");
        node = node->next;
        while ( node ) {
            printf("[%d],count:%d\tword:%s\n",
                node->order, node->count, node->string);
            node = node->next;
        }
        printf("\n");
    }
    void update_insert_node(
        count_data *insert_node )
    {
        if ( !insert_node->next )
            return;
        count_data *print_p = insert_node;

        if ( insert_node->order == 1 ) {
            delete_list_all_node( insert_node->next );
            insert_node->next = NULL;
        } else if ( insert_node->order < 1 ) {
            PRINT("ERROR!!!!!");
            exit( 0 );
        } else {
            count_data *p = insert_node;
            insert_node = insert_node->next;
            while ( insert_node ) {
                if ( insert_node->count < p->count ) {
                    insert_node->order = p->order - 1;
                } else if ( insert_node->count > p->count ) {
                    PRINT("ERROR!!!cur->count:%d, pre->count:%d",
                        insert_node->count, p->count);
                    exit( 0 );
                } else {
                    insert_node->order = p->order;
                }
                if ( insert_node->order < 1 ) {
                    delete_list_all_node( insert_node );
                    p->next = NULL;
                    break;
                }
                p = insert_node;
                insert_node = insert_node->next;
            }
        }
    }
    void list_insert(
        char *tmp_word,
        int cur_count,
        int tail,
        count_data *head,
        int top_num )
    {
        tmp_word[tail] = '\0';
        count_data *new_data = ( count_data * )malloc( sizeof(count_data) );
        new_data->count = cur_count;
        memcpy( new_data->string, tmp_word, tail + 1 );
        new_data->next = NULL;

        //PRINT("count:%d\ttmp_word:%s, string:%s", cur_count, tmp_word, new_data->string);

        if ( head->next == NULL ) {
            head->next = new_data;
            new_data->order = top_num;
        } else if ( cur_count > head->next->count ) {
            new_data->order = head->next->order;
            new_data->next = head->next;
            head->next = new_data;
            update_insert_node( new_data );
        } else {
            while ( 1 ) {
                head = head->next;
                if ( head->next == NULL ) {
                    if ( head->order > 1 ) {
                        head->next = new_data;
                        if ( head->count == new_data->count )
                            new_data->order = head->order;
                        else
                            new_data->order = head->order - 1;

                        head->next = new_data;
                    } else if ( head->count > new_data->count ) {
                        // 不插入
                        free( new_data );
                    } else if ( head->count == new_data->count ) {
                        head->next = new_data;
                        new_data->order = head->order;
                    } else if ( head->count < new_data->count ) {
                        // 此种情况只有求出现次数最多的前1个单词时有
                        head->count = new_data->count;
                        free( new_data );
                    }
                    break;
                } else if ( head->count >= cur_count
                    && head->next->count < cur_count ) {
                    new_data->next = head->next;
                    head->next = new_data;
                    new_data->order = head->order;
                    update_insert_node( new_data );
                    break;
                }
            }
        }
    }
    void find_top_count1(
        trieTreeNode *root,
        char *tmp_word,
        int tail,
        count_data *head,
        int top_num )
    {
        if ( !root )
            return;

        tmp_word[tail] = root->letter;
        tail++;

        if ( root->is_word ) {

            /*
            printf("\n--------------before delete------------------\n");
            print_list_all_node( head );
            printf("\n--------------------------------------------\n");
            */

            list_insert( tmp_word, root->count, tail, head, top_num );

            /*
            printf("\n--------------------after delete----------------------------\n");
            print_list_all_node( head );
            printf("\n-----------------------------------------------------------\n");
            */
        }

        int i = 0;
        for ( i; i < MAX_CHILD_NUM; i++ ) {
            find_top_count1( root->next[i], tmp_word, tail, head, top_num );
        }
    }
    void find_top_count(
        trieTreeNode *root,
        int top_num )
    {
        if ( !root )
            return;

        int i = 0;

        count_data *head = ( count_data * )malloc( sizeof(count_data) );

        for ( i; i < MAX_CHILD_NUM; i++ ) {
            char tmp_word[20] = {0};
            find_top_count1( root->next[i], tmp_word, 0, head, top_num );
        }

        printf("出现次数最大前%d次的单词:\n", top_num);
        count_data *p = head->next;
        count_data *free_p = NULL;
        while ( p != NULL ) {
            free_p = p;
            printf("前%d,count:%d\t%s\n", p->order, p->count, p->string);
            p = p->next;
            free( free_p );
        }
        free( head );
    }

    void tree_walk1(
        trieTreeNode *root,
        char *tmp_word,
        int tail )
    {
        if ( !root )
            return;

        tmp_word[tail] = root->letter;
        tail++;
        //printf("%c\n", root->letter);
        if ( root->is_word ) {
            int j = 0;
            printf("count:%d\t", root->count);
            for ( j; j < tail; j++ ) {
                printf("%c", tmp_word[j]);
            }
            printf("\n");
        }

        int i = 0;
        for ( i; i < MAX_CHILD_NUM; i++ ) {
            tree_walk1( root->next[i], tmp_word, tail );
        }
    }
    void tree_walk(
        trieTreeNode *root )
    {
        if ( !root )
            return;

        int i = 0;

        for ( i; i < MAX_CHILD_NUM; i++ ) {
            char tmp_word[20] = {0};
            tree_walk1( root->next[i], tmp_word, 0 );
        }
    }
    int main(
        int argc,
        char **argv )
    {
        if ( argc != 3 ) {
            PRINT("USAGE: please input words file & top number");
            exit( 0 );
        }

        char *file_name = argv[1];
        int top_num = atoi( argv[2] );

        trieTree *tree = ( trieTree * )malloc( sizeof(trieTree) );

        tree->root = create_node( -1 );

        int fd = open(file_name, O_RDONLY);
        if ( fd < 0 ) {
            PRINT("OPEN FILE %s ERROR!!!(%s)", file_name, (char *)strerror(errno));
            exit( 0 );
        }
        // 每次读取文件的缓冲区
        char buf[1024 * 10] = {0};

        // 每次读取的大小
        int read_len = 1024;

        // 读取的返回值
        int read_bytes = 0;

        // 从读取的缓冲区每次提取'\n' - '\n'之间的单词
        char tmp_word[20] = {0};

        // 读取文件缓冲区如果出现单词隔断，剩余部分在下一次
        // read才能读到，这个index做单词继续拼接
        int tmp_index = 0;

        while ( 1 ) {
            memset( buf, 0, read_len );
            read_bytes = read( fd, buf, read_len );
            if ( read_bytes <= 0 )
                break;
            //printf("readbytes:%d------\n%s\n", read_bytes, buf);
            int cur = 0;
            while ( cur < read_bytes ) {
                // 单词文件最后一个单词末尾一定要有'\n'
                if ( buf[cur] == '\n' ) {
                    tmp_word[tmp_index] = '\0';
                    //printf("insert word:%s\n", tmp_word);
                    insert( tree->root, tmp_word );
                    memset( tmp_word, 0, 20 );
                    tmp_index = 0;
                } else {
                    tmp_word[tmp_index] = buf[cur];
                    tmp_index++;
                }
                cur++;
            }
        }
        printf("\n========================================\n");
        tree_walk( tree->root );

        find_top_count( tree->root, top_num );

        close( fd );

        return 0;
    }

trie树的代码使用：./xxx word.txt 10即统计出现次数前10的单词，并打印单词和次数

例如对生成了10000个单词的word.txt文件，统计前5：
./xxx word.txt 5


![trie_tree]

[trie_tree]:/img/trie_tree.png ""