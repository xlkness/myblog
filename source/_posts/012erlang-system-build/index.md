---
title: ubuntu16+ideaIC+rebar3搭建erlang开发环境
date: 2018-06-19 19:42:44
categories:
- 开发
tags:
- Erlang
---


# 1.ubuntu16系统
# 2.安装各种库

    sudo apt-get install build-essential
    sudo apt-get install libncurses5-dev
    sudo apt-get install libssl-dev
    sudo apt-get install m4
    sudo apt-get install unixodbc unixodbc-dev
    sudo apt-get install freeglut3-dev libwxgtk2.8-dev
    sudo apt-get install xsltproc
    sudo apt-get install fop
    sudo apt-get install tk8.5

# 3.安装erlang源码

deb安装包：esl-erlang_19.1.3-1-ubuntu-xenial_amd64.deb

`dpkg -i esl-erlang_19.1.3-1-ubuntu-xenial_amd64.deb`

# 4.安装ideaIC工具

百度搜索安装ideaIC，我安装的ideaIC3.4版本，[地址][1]


# 5.下载rebar3
rebar3[地址][2]

# 6.ideaIC安装erlang插件
打开ideaIC，进入configure菜单进入settings进行设置：settings -> Plugins -> Browse repositories，然后搜索erlang，就可以安装erlang插件了

# 7.配置
安装完后再次进入settings界面：settings -> Build,Execution,Deployment -> Compiler -> Erlang Compiler，将”Compiler project with rebar”和”Add debug info”都打勾。

接着：settings -> Other Settings -> Erlang External Tools，将”rebar”的路径设置为下载的rebar3可执行路径的目录。
配置完成。

# 8.创建、打开新工程等
略

[1]:http://www.jetbrains.com/idea/
[2]:http://www.rebar3.org/
