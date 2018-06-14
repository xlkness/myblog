---
title: 编译60个小程序之makefile
date: 2018-06-14 03:39:08
categories:
- 开发
tags:
- C/C++
---

公司有个任务需要编译60个c语言小程序，工程目录结构为：

src：放所有小程序源文件.c

drv：所有小程序编译后都为对应.drv

其它头文件、库目录省略。

makefile不太熟，也很菜，我第一想法是用for循环进行循环编译，还用到了makefile自定义函数，贴代码：

    CC = arm-linux-gcc

    CFLAGS +=-Wall -O -D_REENTRANT -fpic -shared
    LDFLAGS += -L./lib -lutility -lcrc -lmxml -lserial -lsocket -lpthread

    ROOT_DIR = $(shell pwd)
    SRC_DIR = ./src
    DRV_DIR = ./drv
    #src/*.c
    SRC := $(wildcard ${SRC_DIR}/*.c)
    SRC1 := $(notdir $(SRC))
    ALL_NAME := $(basename ${SRC1})

    #自定义了一个compile_file函数
    define compile_file
            $(CC) $(CFLAGS) -o $1 $2 $(LDFLAGS)
    endef

    default:
            for name in $(ALL_NAME); do \
    #这里调用自定义函数，传输两个参数:一个drv/xx.drv，一个src/xx.c
            ${call compile_file, $(DRV_DIR)/$$name.drv, $(SRC_DIR)/$$name.c}; done

    clean:
            rm -f $(DRV_DIR)/*.drv

编译的效果是将所有的小程序都编译一遍，不管有没有出错，不管是否为最新。我需要的效果是编译所有程序，编译到哪一个出错即停止，编译前还要检查目标文件和源文件的更新时间，因此这个makefile不好用，只是学习了一下makefile的循环和自定义函数。

然后又构思makefile该如何写，就在思考的过程中想起来了学裸机程序时工程有一个.S和一个.c文件的编译，再结合makefile的伪目标，结构就很清晰了，这里用一个变量ALL_NAME表示获取到的所有src目录下的.c文件的名字替换为.drv（去除src/目录名和.c后缀，再补上drv/和.drv后缀，形式为drv/xxx.drv），代码忘了拷，贴部分自己能记住的吧：

    .PHONY:default clean

    default:$(ALL_NAME)

    $(DRV_DIR)/%.drv:$(SRC_DIR)/%.c
        $(CC) $(CFLAGS) -o $@ $< $(LDFLAGS)

    clean:
        rm $(DRV_DIR)/*.drv -r

这个makefile就能满足之前的要求了。通过两个makefile的编写，学习了makefile的函数、自定义函数、循环、伪目标、makefile规则与shell规则的混合问题。