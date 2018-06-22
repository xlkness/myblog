---
title: Erlang Abstract Syntax Tree和汇编字节码
date: 2018-06-22 20:35:59
categories:
- 开发
tags:
- Erlang
---

# 一、抽象语法树简介
抽象语法树(Abstract Syntax Tree)是源代码的抽象语法结构的树状表示。

抽象语法树是解析器(parser)的产物，解析器广义来说输入一般是程序的源码，输出一般是语法树（syntax tree，也叫parse tree等）或抽象语法树。进一步剥开来，广义的解析器里一般会有扫描器（scanner，也叫tokenizer或者lexical analyzer，词法分析器），以及狭义的解析器（parser，也叫syntax analyzer，语法分析器）。扫描器的输入一般是文本，经过词法分析，输出是将文本切割为单词的流。狭义的解析器输入是单词的流，经过语法分析，输出是语法树或者精简过的AST。

例如：将i = a + b * c作为源代码输入到解析器里，则广义上的解析器的工作流程如下图：

![erlang_ast1]

# 二、用途
	erlang beam是寄存器虚拟机，因此erlang的源码会被erlang解析器的词法分析变为AST，然后经过树形遍历解析器给转换为汇编字节码。


# 三、抽象语法树初识

## 1、创建一个test1.erl的文件，输入以下代码：

    -module(test1).
    -export([start/0]).

    -compile({parse_transform, test_parser}).

    start() ->
        "hello world".

## 2、创建一个test_parser.erl的文件，输入以下代码：

    -module(test_parser).

    -export([parse_transform/2]).

    parse_transform(AST, _Options) ->
        io:format("old:~w~n~n", [AST]),
        Acc = parse_ast(AST, []),
        io:format("new:~w~n~n", [Acc]),
        Acc.


    parse_ast([{attribute, _, _, _} = H | R], Acc) ->
        parse_ast(R, [H | Acc]);
    parse_ast([{function, _Line, _Fun, _Arity, _Args} = H | R], Acc) ->
        parse_ast(R, [parse_fun(H) | Acc]);
    parse_ast([H | R], Acc) ->
        parse_ast(R, [H | Acc]);
    parse_ast([], Acc) ->
        lists:reverse(Acc).

    parse_fun({function, Line, Fun, Arity, Clause}) ->
        {function, Line, Fun, Arity, parse_clause(Clause, [])}.

    parse_clause([{clause, Line, P, Guard, Return}], Acc) ->
        [{clause, Line, P, Guard, parse_return(Return, [])} | Acc].

    parse_return([{string, Line, Value} | R], Acc) ->
        parse_return(R, [{bin, Line, [{bin_element, Line, {string, Line, Value},default,default}]} | Acc]);
    parse_return([], Acc) ->
        lists:reverse(Acc).

## 3、编译

    1> c(test_parser).
    {ok,test_parser}
    2> c(test1).
    old:[{attribute,1,file,{[116,101,115,116,49,46,101,114,108],1}},{attribute,1,module,test1},{attribute,3,export,[{start,0}]},
    {function,7,start,0,[{clause,7,[],[],[{string,8,[104,101,108,108,111,32,119,111,114,108,100]}]}]},
    {eof,9}]

    new:[{attribute,1,file,{[116,101,115,116,49,46,101,114,108],1}},{attribute,1,module,test1},{attribute,3,export,[{start,0}]},{function,7,start,0,[{clause,7,[],[],[{bin,8,[{bin_element,8,{string,8,[104,101,108,108,111,32,119,111,114,108,100]},default,default}]}]}]},{eof,9}]

    {ok,test1}

先不管io:format的打印，我们直接执行test1:start()，

    3> test1:start().
    <<"hello world">>

发现终端输出的是二进制！而我们源码明明写的是个字符串！！破编译器对我们的代码干了什么！！

别惊慌，这时候我们就可以来看看我们打印的抽象语法树了。

先看old，attribute都是源码文件的属性，例如file、export、import、module名等，可以略过，以及eof表示源码文件的结尾，也可以忽略。直接定位到function元组，{function, Line, Name, Arity, Clauses}表示一个函数的源码行、函数名、形参数、模式匹配列表，再定位到模式匹配列表clause，其表示了一个函数的多个模式匹配和返回值，当前start函数就一个匹配，因此clause只有一个tuple元素，返回值就是一个{string, Line, "hello world"}，
那么我们是不是修改这个{string, Line, "hello world"}为一个二进制字符串，就能达到我们演示的效果呢？是的！

这就是test1.erl中-compile({parse_transform, test_parser}).的作用，erlang的解析器会在词法分析完成后，调用我们自己指定的函数对ast进行二次更改，于是test_parser.erl里面就将start函数的返回值更改为了一个二进制。然后返回一个新的AST，再经树形解析编译为erlang汇编字节码文件，供erlang vm调用。

# 四、应用
lager日志库 https://github.com/erlang-lager/lager

以前一直疑惑使用lager日志库为什么要在编译选项上加一个{parse_transform, lager_transform}，现在也懂了，

就是编译我们项目每一个源文件时，都将最终的抽象语法树再次添加一些特性代码。

这里我们可以打印一下，现在我在一个叫erlserver_app.erl的文件里加入lager日志打印代码：


    -module(erlserver_app).
    -behaviour(application).

    -export([start/2, stop/1]).

    -compile([{parse_transform, lager_transform}]).

    start(_StartType, _StartArgs) ->
        lager:warning("12345678987654321~w~n", ["abcdedeba"]),
        erlserver_sup:start_link().

    %%--------------------------------------------------------------------
    stop(_State) ->
        ok.

然后在lager_transform里继续像上面那样打印，

最终编译输出：

    ===> old:
    [{attribute,1,file,{[47,104,111,109,101,47,108,105,107,117,110,47,115,116,117,100,121,47,101,114,108,97,110,103,47,101,114,108,115,101,114,118,101,114,47,95,98,117,105,108,100,47,100,101,102,97,117,108,116,47,108,105,98,47,101,114,108,115,101,114,118,101,114,47,115,114,99,47,101,114,108,115,101,114,118,101,114,95,97,112,112,46,101,114,108],1}},{attribute,6,module,erlserver_app},{attribute,8,behaviour,application},{attribute,11,export,[{start,2},{stop,1}]},{attribute,13,compile,[]},{function,19,start,2,[{clause,19,[{var,19,'_StartType'},{var,19,'_StartArgs'}],[],[{call,20,{remote,20,{atom,20,lager},{atom,20,warning}},[{string,20,[49,50,51,52,53,54,55,56,57,56,55,54,53,52,51,50,49,126,119,126,110]},{cons,20,{string,20,[97,98,99,100,101,100,101,98,97]},{nil,20}}]},{call,21,{remote,21,{atom,21,erlserver_sup},{atom,21,start_link}},[]}]}]},{function,24,stop,1,[{clause,24,[{var,24,'_State'}],[],[{atom,25,ok}]}]},{eof,30}]


    ===> new:
    [{attribute,1,file,{[47,104,111,109,101,47,108,105,107,117,110,47,115,116,117,100,121,47,101,114,108,97,110,103,47,101,114,108,115,101,114,118,101,114,47,95,98,117,105,108,100,47,100,101,102,97,117,108,116,47,108,105,98,47,101,114,108,115,101,114,118,101,114,47,115,114,99,47,101,114,108,115,101,114,118,101,114,95,97,112,112,46,101,114,108],1}},{attribute,6,module,erlserver_app},{attribute,6,lager_records,[]},{attribute,8,behaviour,application},{attribute,11,export,[{start,2},{stop,1}]},{attribute,13,compile,[]},{function,19,start,2,[{clause,19,[{var,19,'_StartType'},{var,19,'_StartArgs'}],[],[{'case',20,{tuple,20,[{call,20,{atom,20,whereis},[{atom,20,lager_event}]},{call,20,{atom,20,whereis},[{atom,20,lager_event}]},{call,20,{remote,20,{atom,20,lager_config},{atom,20,get}},[{tuple,20,[{atom,20,lager_event},{atom,20,loglevel}]},{tuple,20,[{integer,20,0},{nil,20}]}]}]},[{clause,20,[{tuple,20,[{atom,20,undefined},{atom,20,undefined},{var,20,'_'}]}],[],[{call,20,{'fun',20,{clauses,[{clause,20,[],[],[{tuple,20,[{atom,20,error},{atom,20,lager_not_running}]}]}]}},[]}]},{clause,20,[{tuple,20,[{atom,20,undefined},{var,20,'_'},{var,20,'_'}]}],[],[{call,20,{'fun',20,{clauses,[{clause,20,[],[],[{tuple,20,[{atom,20,error},{tuple,20,[{atom,20,sink_not_configured},{atom,20,lager_event}]}]}]}]}},[]}]},{clause,20,[{tuple,20,[{var,20,'__Piderlserver_app20'},{var,20,'_'},{tuple,20,[{var,20,'__Levelerlserver_app20'},{var,20,'__Traceserlserver_app20'}]}]}],[[{op,20,'orelse',{op,20,'/=',{op,20,'band',{var,20,'__Levelerlserver_app20'},{integer,20,16}},{integer,20,0}},{op,20,'/=',{var,20,'__Traceserlserver_app20'},{nil,20}}}]],[{call,20,{remote,20,{atom,20,lager},{atom,20,do_log}},[{atom,20,warning},{cons,20,{tuple,20,[{atom,20,application},{atom,20,erlserver}]},{cons,20,{tuple,20,[{atom,20,module},{atom,20,erlserver_app}]},{cons,20,{tuple,20,[{atom,20,function},{atom,20,start}]},{cons,20,{tuple,20,[{atom,20,line},{integer,20,20}]},{cons,20,{tuple,20,[{atom,20,pid},{call,20,{atom,20,pid_to_list},[{call,20,{atom,20,self},[]}]}]},{cons,20,{tuple,20,[{atom,20,node},{call,20,{atom,20,node},[]}]},{call,20,{remote,20,{atom,20,lager},{atom,20,md}},[]}}}}}}},{string,20,[49,50,51,52,53,54,55,56,57,56,55,54,53,52,51,50,49,126,119,126,110]},{cons,20,{string,20,[97,98,99,100,101,100,101,98,97]},{nil,20}},{integer,20,4096},{integer,20,16},{var,20,'__Levelerlserver_app20'},{var,20,'__Traceserlserver_app20'},{atom,20,lager_event},{var,20,'__Piderlserver_app20'}]}]},{clause,20,[{var,20,'_'}],[],[{atom,20,ok}]}]},{call,21,{remote,21,{atom,21,erlserver_sup},{atom,21,start_link}},[]}]}]},{function,24,stop,1,[{clause,24,[{var,24,'_State'}],[],[{atom,25,ok}]}]},{eof,30}]

看来添加的东西挺多，没事，慢慢看，我们根据官方文档The Abstract Format一个一个对比看，

最终将添加的代码还原出来：

    case {whereis(lager_event), whereis(lager_event), lager_config:get({lager_event, loglevel}, {0, []})} of
        {undefined, undefined, _} ->
            {error, lager_not_running};
        {undefined, _, _} ->
            {error, sink_not_configured, lager_event};
        {__Piderlserver_app20, _, {__Levelerlserver_app20, __Traceserlserver_app20}} when __Levelerlserver_app20 band 16 /= 0 orelse __Traceserlserver_app20 /= nil ->
            lager:do_log(warning, [{application, erlserver}, {atom, erlserver_app}, {function, start}, {line, 20}, {pid, pid_to_list(self())}, {node, node()}, lager:md()],
                [49,50,51,52,53,54,55,56,57,56,55,54,53,52,51,50,49,126,119,126,110], [[97,98,99,100,101,100,101,98,97], []], 4096, 16, __Levelerlserver_app20,
                    __Traceserlserver_app20, lager_event, __Piderlserver_app20);
        _ ->
            ok
    end

小小一句lager:warning()原来有这么多弯弯绕啊！

# 五、erlang汇编字节码
一直说erlang汇编字节码，但是具体是个什么东西，还没有认识。

下面新建一个文件test.erl，输入以下内容：

    -module(test).
    -export([start/2]).

    start(A, B) ->
            abs(A) + abs(B).

## 1、编译.S汇编码
    erlc -S test.erl
## 2、编译.dis汇编码
    erl
    c(test).
    erts_debug:df(test).

然后打开得到的test.S文件：

    {module, test}.  %% version = 0
    {module, test}.  %% version = 0

    {exports, [{module_info,0},{module_info,1},{start,2}]}.

    {attributes, []}.

    {labels, 7}.


    {function, start, 2, 2}.
      {label,1}.
        {line,[{location,"test.erl",4}]}.
        {func_info,{atom,test},{atom,start},2}.
      {label,2}.
        {line,[{location,"test.erl",5}]}.
        {gc_bif,abs,{f,0},2,[{x,0}],{x,0}}.
        {line,[{location,"test.erl",5}]}.
        {gc_bif,abs,{f,0},2,[{x,1}],{x,1}}.
        {line,[{location,"test.erl",5}]}.
        {gc_bif,'+',{f,0},2,[{x,0},{x,1}],{x,0}}.
        return.


    {function, module_info, 0, 4}.
      {label,3}.
        {line,[]}.
        {func_info,{atom,test},{atom,module_info},0}.
      {label,4}.
        {move,{atom,test},{x,0}}.
        {line,[]}.
        {call_ext_only,1,{extfunc,erlang,get_module_info,1}}.


    {function, module_info, 1, 6}.
      {label,5}.
        {line,[]}.
        {func_info,{atom,test},{atom,module_info},1}.
      {label,6}.
        {move,{x,0},{x,1}}.
        {move,{atom,test},{x,0}}.
        {line,[]}.
        {call_ext_only,2,{extfunc,erlang,get_module_info,2}}.

每一个function元组标明了模块中的函数，我们直接定位到function start模块，每个label表示pc指针的执行顺序，跳过label1，我们直接看label2，有许多寄存器x0,x1,...xn y0,y1,...yn。x寄存器是用来存储函数入参的，特殊的x0寄存器，函数第一个入参是存放在里面，并且也用来作为函数返回值，而y寄存器是栈上分配的临时寄存器，

有了以上知识，我们直接看汇编码：

（忽略某些无用行）

    {gc_bif,abs,{f,0},2,[{x,0}],{x,0}}. //用x0去调用bif函数abs，存储结果到x0寄存器，

    {gc_bif,abs,{f,0},2,[{x,1}],{x,1}}. //用x1去调用bif函数abs，存储结果到x1寄存器，

    {gc_bif,'+',{f,0},2,[{x,0},{x,1}],{x,0}}. //执行bif函数+， x0+x1，结果放入x0寄存器

    return. //返回，直接返回x0的值

这就是.S汇编字节码，它具有一定可读性，方便人和机器阅读

再打开.dis汇编码文件：

    00007FDAE5183770: i_func_info_IaaI 0 test start 2
    00007FDAE5183798: i_gc_bif1_jIsId j(0000000000000000) abs/1 x(0) 2 x(0)
    00007FDAE51837C8: i_gc_bif1_jIsId j(0000000000000000) abs/1 x(1) 2 x(1)
    00007FDAE51837F8: i_plus_jIxxd j(0000000000000000) 2 x(0) x(1) x(0)
    00007FDAE5183828: return

    00007FDAE5183830: i_func_info_IaaI 0 test module_info 0
    00007FDAE5183858: move_cr test r(0)
    00007FDAE5183868: allocate_tt 0 1
    00007FDAE5183878: call_bif_e erlang:get_module_info/1
    00007FDAE5183888: deallocate_return_Q 0

    00007FDAE5183898: i_func_info_IaaI 0 test module_info 1
    00007FDAE51838C0: move_rx r(0) x(1)
    00007FDAE51838D0: move_cr test r(0)
    00007FDAE51838E0: allocate_tt 0 2
    00007FDAE51838F0: call_bif_e erlang:get_module_info/2
    00007FDAE5183900: deallocate_return_Q 0

其实大体上差不多，不过多了真是调用的函数和执行地址，但是erlang的汇编操作码太多，因此这个文件可读性很差，如果想搞懂具体指令作用可以去查阅相关文档。

[erlang_ast1]:/img/erlang_ast.png ""