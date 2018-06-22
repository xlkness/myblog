---
title: 【从零开始构建erlang服务器】-02构建应用
date: 2018-06-22 20:06:28
categories:
- 开发
tags:
- Erlang
---

# 一、简介
开始一个erlang服务器应用的构建。
项目管理工具使用rebar3。配置方式参考我另一篇ubuntu16+ideaIC+rebar3搭建erlang开发环境

# 二、新建应用
服务器应用名：erlserver，终端执行：

    rebar3 new app erlserver

    ===> Writing erlserver/src/erlserver_app.erl
    ===> Writing erlserver/src/erlserver_sup.erl
    ===> Writing erlserver/src/erlserver.app.src
    ===> Writing erlserver/rebar.config
    ===> Writing erlserver/.gitignore
    ===> Writing erlserver/LICENSE
    ===> Writing erlserver/README.md

此时在当前目录就生成了erlserver项目文件夹。

# 三、添加本地调试配置

    {deps, [
        sync
    ]}.

    {shell, [
        {apps, [erlserver, sync]}
    ]}.

    {dist_node, [
        {setcookie, 'only4test'},
        {name, 'erlserver@127.0.0.1'}
    ]}.

sync是个erlang shell应用，可以动态更新编译最新erlang项目代码，{shell...是rebar3 shell的配置和启动的应用。

# 四、运行应用

shell执行：`rebar3 shell`

    ===> Compiling erlserver
    Erlang/OTP 20 [erts-9.0] [source] [64-bit] [smp:4:4] [ds:4:4:10] [async-threads:0] [hipe] [kernel-poll:false]

    Eshell V9.0  (abort with ^G)
    (erlserver@127.0.0.1)1> ===> The rebar3 shell is a development tool; to deploy applications in production, consider using releases (http://www.rebar3.org/docs/releases)
    Starting Sync (Automatic Code Compiler / Reloader)
    Scanning source files...
    ===> Booted erlserver
    ===> Booted syntax_tools
    ===> Booted compiler
    ===> Booted sync

以上结果可以看到，执行了rebar3 shell之后，会先编译整个项目，然后运行erlserver应用节点，这时候我们就进入了一个erlang shell，并且已经启动了erlserver,sync应用。

# 五、集成网络库

这里我们使用开源的erlang tcp网络库ranch

## 1、添加依赖

 在rebar.config的deps里添加一个ranch依赖：

    {deps, [
        sync,
        {ranch, {git, "https://github.com/ninenines/ranch", {branch, "master"}}}
    ]}.

经过这步就将ranch下载下来，但是依然没有将其运行起来。

在erlserver.app.src的依赖应用里面添加ranch：

    {applications,
       [kernel,
        stdlib,
        ranch
       ]},

五、测试网络库

经过以上步骤，ranch就集成进项目了，使用rebar3 shell运行节点，可以看到ranch应用已经启动。测试ranch能否正常启动和监听端口（照着ranch的example可以创建简单的echo服务器试试）

六、后续展望

服务器项目已经构建，网络库也集成进来了，后面会开始利用ranch创建客户端一对一服务器进程树、以及利用protocol buffer协议做请求与服务、单元测试、common test、应用发布和部署等等。

（project-https://github.com/xlkness/erlserver.git）

（未完待续。。。）











