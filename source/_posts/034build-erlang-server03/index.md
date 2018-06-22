---
title: 【从零开始构建erlang服务器】-03用户层和日志
date: 2018-06-22 20:06:44
categories:
- 开发
tags:
- Erlang
---

# 一、简介
	上一篇讲了创建服务器项目以及添加ranch网络库，本篇利用网络库创建client socket消息处理的用户层代码以及服务器开发调试运维的日志集成。

# 二、编写用户层代码
	前面知道网络层处理客户端连接，以及可以将客户端socket文件描述符授权给其它worker进程来一对一为客户端服务（得益于erlang actor模型的轻进程，可以做到一个连接一个进程），而网络库我们使用的ranch，因此可以参照ranch的example程序编写一个处理socket消息的进程，也就是为用户一对一服务器的用户层代码。

仿照ranch例子创建一个erlserver_user.erl文件

添加如下代码：


    -module(erlserver_user).

    -behaviour(gen_server).
    -behaviour(ranch_protocol).

    %% API.
    -export([start_link/4]).

    %% gen_server.
    -export([init/1]).
    -export([handle_call/3]).
    -export([handle_cast/2]).
    -export([handle_info/2]).
    -export([terminate/2]).
    -export([code_change/3]).

    -define(TIMEOUT, 5000).

    -record(state, {socket, transport}).

    %% API.

    start_link(Ref, Socket, Transport, Opts) ->
        {ok, proc_lib:spawn_link(?MODULE, init, [{Ref, Socket, Transport, Opts}])}.

    init({Ref, Socket, Transport, _Opts = []}) ->
        ok = ranch:accept_ack(Ref),
        ok = Transport:setopts(Socket, [{active, once}]),
        gen_server:enter_loop(?MODULE, [],
                              #state{socket=Socket, transport=Transport},
                              ?TIMEOUT).

    handle_info({tcp, Socket, Data}, State=#state{
        socket=Socket, transport=Transport})
        when byte_size(Data) > 1 ->
        Transport:setopts(Socket, [{active, once}]),
        {ok, PeerName} = inet:peername(Socket),
        io:format("receive from client[~w], msg:~p~n", [PeerName, Data]),
        {noreply, State, ?TIMEOUT};
    handle_info({tcp_closed, _Socket}, State) ->
        {stop, normal, State};
    handle_info({tcp_error, _, Reason}, State) ->
        {stop, Reason, State};
    handle_info(timeout, State) ->
        {stop, normal, State};
    handle_info(_Info, State) ->
        {stop, normal, State}.

    handle_call(_Request, _From, State) ->
        {reply, ok, State}.

    handle_cast(_Msg, State) ->
        {noreply, State}.

    terminate(_Reason, _State) ->
        ok.

    code_change(_OldVsn, State, _Extra) ->
        {ok, State}.

阅读过ranch源码，就知道ranch_acceptor在监听一个客户端连接后，就会调用我们指定的模块的start_link/4方法，当然，我们可以直接在start_link中启动一个gen_server，不过ranch的这个例子因为要有一个ranch:accept_ack操作，所以会阻塞在init，因此没有用gen_server，用了prob_lib:spawn_link，立即返回一个可用的pid给ranch库，然后在慢慢初始化，初始化完成后用gen_server:enter_loop将这个进程转变为一个gen_server进程。

# 三、启动ranch监听功能
上一篇只在app.src启动了ranch，这只是表示应用启动起来，要使ranch开始监听某个端口，还需要显示调用ranch:start_listener/5，这里我们要监听本机8888端口，并且使用我们在第二步写好的客户端socket套接字消息处理进程，其它tcp参数都使用默认，则我们在erlserver_app.erl启动中添加以下代码：

    {ok, _} = ranch:start_listener(erlserver,
                                      ranch_tcp, [{port, 8888}], erlserver_user, []),

rebar3 shell启动我们的应用测试一下是否可用，另开一台机器进入erlang shell，输入{ok, Fd} = gen_tcp:connect({xxx,xxx,xxx,xxx}, 8888, [binary, {active, false}, {packet, raw}]), gen_tcp:send(Fd, "for test").  看到输出，就说明现在可以创建一个并发服务器了，并且每个客户端连接都对应一个erlserver_user进程单独处理。

# 四、添加lager日志库
在第二步的代码中，我们用了io:format来打印收到的客户端信息，这样做其实是不好的，终端打印的消息无法输出到文件记录下来，并且io打印的消息都是原始消息，不方便调试，要同时输出模块、行号等信息，每一次的io打印还都要加上这些信息，总之，对于服务器来说，没有一个专门的日志记录都是不好的。

## 1、rebar.config添加依赖：
	{lager, {git, "https://github.com/erlang-lager/lager", {branch, "master"}}}

## 2、erlserver.app.src启动lager
（lager的顺序尽量放在除erlang runtime库之前，例如放在ranch之前，这样我们就可以在其它应用启动中也能使用lager输出应用启动的信息了）。

## 3、添加lager配置
这里前期为了调试方便，我们直接利用rebar3 shell加载一个本地调试配置文件
		在项目根目录创建一个config文件夹，并在里面创建一个erlserver.config配置文件，输入以下内容

    [
        {lager, [
            {log_root, "./log"},
            {handlers, [
                {lager_console_backend,
                 [{level, info}, {formatter, lager_default_formatter},
                  {formatter_config, [date, "/", time, "[", severity, "][", module, ":", function, ":", line,"]", "|", message, "\n"]}]},
                {lager_file_backend,
                 [{file, "error.log"}, {level, error}, {formatter, lager_default_formatter},
                  {formatter_config, [date, "/", time, "[", module, ":", function, ":", line,"]", "|", message, "\n"]}]},
                {lager_file_backend,
                 [{file, "console.log"}, {level, info}, {formatter, lager_default_formatter},
                  {formatter_config, [date, "/", time, "[", module, ":", function, ":", line,"]", "|", message, "\n"]}]}
            ]}
        ]}
    ].

这就是一个lager的简单配置，日志都输出到log目录，并且info对应console.log文件（目前都是本地调试，并不是发布的配置）。

# 4、配置rebar3 shell读取config文件以及添加lager的parse_transform
修改rebar.config的shell配置

    {shell, [
        {apps, [erlserver, sync]},
        {config, "config/erlserver.config"}
    ]}.

修改rebar.config的erl_opts配置（这里使用到的知识点可以参考我的另一篇erlang抽象语法树博客）


    {erl_opts, [
        debug_info,
        {parse_transform, lager_transform}
    ]}.


# 五、使用日志库
修改第二步的erlserver_user的io:format打印为lager:info("xxxxx"),再来尝试客户端连接并且传输信息。

# 六、后续
客户端消息处理层（用户层）以及日志都写好了，后续开始制定协议层了。

（project-https://github.com/xlkness/erlserver.git）