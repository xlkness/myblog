---
title: erlang的终端带颜色输出与中文字符串输出
date: 2018-06-19 20:19:59
categories:
- 开发
tags:
- Erlang
---

# 1.带颜色输出
erlang终端支持带颜色输出，例如lager日志库就可以。其实就是在输出前设置一下输出属性，正常的字体是：”\e[0;38m”
下面自己弄了一些宏：

    -define(CONSOLE_COLOR_RED,      "\e[0;31m").
    -define(CONSOLE_COLOR_RED_BOLD, "\e[1;31m").
    -define(CONSOLE_COLOR_YELLOW1,  "\e[0;32m").
    -define(CONSOLE_COLOR_YELLOW2,  "\e[0;33m").
    -define(CONSOLE_COLOR_BLUE,     "\e[0;34m").
    -define(CONSOLE_COLOR_PURPLE,   "\e[0;35m").
    -define(CONSOLE_COLOR_GREEN,    "\e[0;36m").
    -define(CONSOLE_COLOR_GRAY,     "\e[0;37m").
    -define(CONSOLE_COLOR_NORMAL,   "\e[0;38m").

用法就是打印的字符串前后加上要设置的属性例如
`io:format(“~s~s~s~n”, [“\e[0;31m”, debug, “\e[0;38m”]).`
就是在输出debug之前设置字体为红色，然后输出结束后设置字体为正常白色。

# 2.中文输出
中文输出乱码在rebar3插件里用rebar_info输出以及在ct里用io:format(user, ….)输出会遇到，带中文的字符串要转为utf8，代码形如：

    NewFormat = io_lib:format(Format, Args),
    io:format("~s", [unicode:characters_to_binary(NewFormat, utf8)]).