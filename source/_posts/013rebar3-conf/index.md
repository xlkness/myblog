---
title: erlang rebar3配置文件
date: 2018-06-19 19:59:19
categories:
- 开发
tags:
- Erlang
---


rebar3的简单使用可以参考rebar3的官方文档。以下讲解一些rebar3的配置，初入erlang，理解还不甚深刻。
用rebar3进行工程创建，会生成rebar.config文件，贴一些配置的使用方法

# 1.编译设置

    %% 编译设置
    {erl_opts, [
    {parse_transform, lager_transform}
    , {parse_transform, ms_transform}
    , report
    , warn_export_all
    , warn_export_vars
    , warn_obsolete_guard
    , warn_shadow_vars
    , warn_unused_function
    , warn_deprecated_function
    %% ,warn_missing_spec
    , warn_unused_import
    ]}.

{parse_transform, lager_transform}是lager依赖库的编译选项，修改抽象语法树的方式在编译期生成对应的代码，lager源代码里本身没有lager:error，lager:info等等方法

# 2.rebar3 shell
    {shell, [
    {apps, [app_name, sync, recon]}
    , {config, “config/app_name.config”}
    ]}.

这个配置支持在项目根目录直接运行rebar3 shell启动一个erl shell来运行我们的app，而其配置可以指定为config目录下的某个配置文件，而不是sys.config，适合本地调试，app_name后面的app名字是需要依赖启动的app

# 3.rebar3插件

    {plugins,
    [
    rebar3_run
    , rebar3_auto
    , {relflow, “1.0.5”}
    ]}.

配置我们项目需要的plugins，这里的插件可以是我们自己编写的rebar3插件

# 4.钩子 provider_hooks

    {provider_hooks,
    [
    {pre,[ {compile, {my_plugins, do_something}} ]},
    {post,[{compile,{my_plugins1, clean}}]}
    ]}.

例如这份配置，就是在执行rebar3 compile之前(pre)运行以my_plugins命名空间下的do_something命令，简单说就是编写了一个rebar3的插件叫my_plugins，提供一个命令叫do_something，即可以在命令行执行rebar3 my_plugins do_something的功能，只是现在配置之后自动调用了命令；post同理就是在compile之后执行那个插件的clean功能，clean功能具体干什么我们不得而知。

# 5.环境
    {profiles, [
    {profile1, [
    {erl_opts, [no_debug_info]},
    {relx, [
    {include_src, false}
    , {dev_mode, false}
    , {include_erts, true}
    , {system_libs, true}
    ]}
    ]}]}

例如这份配置，指定了一种环境叫profile1，编译选项erl_opts为no_debug_info，打包发布的选项为不包含源码，禁止开发模式(目录不是软连接于default环境)，包含erlang环境等，当然还可以加其它很多选项，为每种环境单独自定义需要的功能，常用default、prod、test等

# 6.覆盖

    {overrides, [{add, deps1, [{erl_opts, [no_debug_info]}]}]}

例如这份配置，就是对名为deps1的依赖的rebar.config再添加一个配置，overrides提供了add和override两种功能，第一种是加配置，第二种也就是用配置的数据去覆盖原来依赖中有的数据