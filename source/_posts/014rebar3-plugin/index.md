---
title: erlang编写rebar3插件
date: 2018-06-19 20:11:35
categories:
- 开发
tags:
- Erlang
---

# 1.生成插件工程
假设插件名为testp，执行rebar3 new plugins testp，即生成了插件工程项目，查看目录结构如图：

![rebar3plugin1]

testp.erl文件调用初始化的代码，而插件最重要的代码在testp_prv.erl的文件，文件里提供了三个接口，分别为`init/1`,`do/1`,`format_error/1`，init做插件初始化的工作，初始化命名空间/初始化命令，然后将命令加入到rebar3，贴一份配置：

    init(State) ->
            Provider = providers:create([
                    {name, compile},            % The 'user friendly' name of the task
                    {module, ?MODULE},            % The module implementation of the task
                    {namespace, testp},
                    {bare, true},                 % The task can be run by the user, always true
                    {deps, ?DEPS},                % The list of dependencies
                    {example, "rebar3 testp compile"}, % How to use the plugin
                    {opts, []},                   % list of options understood by the plugin
                    {short_desc, "rebar3 testp compiler"},
                    {desc, "rebar3 testp compiler"}
            ]),
            {ok, rebar_state:add_provider(State, Provider)}.

这份配置就是为rebar3添加一个rebar3 testp compile命令，如果只输入`rebar3 testp`，会提示`rebar3 testp compiler`。
第二个方法do就是主要的业务逻辑了，执行了`rebar3 testp compiler`命令就会调用里面的do方法。
如果想添加多个命令，可以在testp.erl里多写一个命令的init调用，然后再添加一个命令类似的prv文件，写不同的do业务逻辑，就可以上传git，将git地址配置在rebar.config的plugins里了，rebar3 compile编译就自动拉取插件代码。
插件还可以配置钩子(provider_hook)使用，例如添加编译前调用
`{provider_hooks,[{pre[{compile, {testp, compile}}]}]}`，编译后调用将pre改为post即可


[rebar3plugin1]:/img/rebar3plugin1.jpeg ""