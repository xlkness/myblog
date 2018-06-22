---
title: Erlang词法分析器、语法分析器(lexer-leex,yac-yecc)
date: 2018-06-22 20:36:09
categories:
- 开发
tags:
- Erlang
---


# 一、简介
一门编程语言的编译器或者解释器通常功能分解为两步：
> 1、读取源码文件然后分析它的结构
>
>2、处理这些结构，例如生成目标程序
lexer和yacc就是能完成第一步以便生成程序段的工具。而第一步的任务又能分为两个子任务：
>1、分割源码文件内容为很多tokens(lexer)
>
>2、分析出程序的分级结构(yacc)

# 二、lexer（词法分析工具）
lexer的源就是一个正则表达式表，其正则规则符合目标程序的代码片段。正则表达式表读取输入流，并将其根据正则规则转换为分割的字符串，然后输出为输出流，再翻译为一个程序。当 Lex 接收到文件或文本形式的输入时，它试图将文本与常规表达式进行匹配。 它一次读入一个输入字符，直到找到一个匹配的模式。 如果能够找到一个匹配的模式，Lex 就执行相关的动作（可能包括返回一个标记）。 另一方面，如果没有可以匹配的常规表达式，将会停止进一步的处理，Lex 将显示一个错误消息。
Erlang的lexer工具就是leex模块。

# 三、yacc（语法分析工具）
是一个用来生成编译器的编译器（编译器代码生成器）。输入词法分析器的流，输出目标语言的代码。
Erlang的yacc工具就是yecc。

# 四、作用
有了这两个工具，我们可以自定义自己的编程语言，自定义这个语言的语法规则，最终生成相同功能的erlang代码。例如领域特定语言(DSL)，我们可以定义个简单类似makefile的mymakefile语法规则，然后生成erlang代码来真正执行功能。

# 五、初识
这次研究leex和yecc也是在研究erlang protocolbuf时候，发现通用的做法就是书写一个协议字段描述文件，然后来生成erlang文件，例如一条TestRequest协议，定义其协议号10000，编解码文件为test_proto.erl，那么描述关系就是10000,TestRequest,test_proto，然后我们代码中传包解包时，都在协议头带上一个16位的协议号，通过协议号路由到真正的编解码文件也就是test_proto.erl。而看了很多实现，都是用riak官方某个rebar插件，来将这种关系描述文件生成erlang文件。

这个功能很简单，不过想是不是能用leex、yecc完成，最近正好也在研究，于是就产生了试一试的想法。

# 六、定义leex的.xrl文件
leex需要.xrl文件来描述自定义源文件的正则匹配规则，例如上面的协议描述关系，我们可以用[0-9]+来匹配出开始的协议号，用[,][a-zA-Z]([0-9a-zA-Z]*_?)*[,]匹配出协议名。

leex文件需要三个部分:Definitions Rules Erlangcode，Definitions表示正则匹配的变量，Rules表示匹配的规则和匹配后的输出，Erlangcode则是作为辅助的erlang函数。

我们可以按照第五步来编写我们的.xrl文件（假设叫test_lexer.xrl）：


    Definitions.


    TypeID = [0-9]+
    MsgName = [,][a-zA-Z]([0-9a-zA-Z]*_?)*[,]
    MsgModule = [a-zA-Z]([0-9a-zA-Z]*_?)*(\n)?
    NoneLine = [\n]

    Rules.

    {TypeID} : {token, {msg_number, TokenLine, list_to_integer(TokenChars)}}.
    {MsgName} : {token, {msg_name, TokenLine, drop_tokens(TokenChars)}}.
    {MsgModule} : {token, {msg_module, TokenLine, drop_tokens(TokenChars)}}.
    {NoneLine} : skip_token.

    Erlang code.

    drop_tokens(TokenChars) ->
        [Chars] = string:tokens(TokenChars, ","),
        [Chars1] = string:tokens(Chars, "\s"),
        [Chars2] = string:tokens(Chars1, "."),
        [Chars3] = string:tokens(Chars2, "\n"),
        Chars3.

上面的代码还是容易看懂的。

使用方法则是先编写一个协议描述的文件test_pb_desc.csv，随便输入个关系：


    10000,fdsfds_232,fdsf_REWr
    10001,fwefsd,terterr

然后通过leex编译出我们的词法分析器：

    leex:file("test_lexer.xrl").

不出意外，当前目录就会多一个test_lexer.erl，然后来分析我们的描述文件：


    {ok, Lines} = file:read_file("test_pb_desc.csv"),
    {_, Tokens, _} = test_lexer:string(binary_to_list(Lines)),

这里得到的Tokens就是我们的词法分析结果，可以输入到下一步的语法分析器里生成erlang代码。

# 七、定义yecc的.yrl文件
yecc需要.yrl文件来描述词法分析的分析方法以及分析的产出。

yecc文件需要四个部分：Nonterminals Terminals Rootsymbol Erlangcode，Nonterminals表示每次输入的流，Terminals表示输入流里面的tokens关键字，Rootsymbol表示输入的根（源），Erlangcode照例是辅助的erlang函数，
我们可以按照第六步的生成来编写yrl文件（例如叫test_parser.yrl）：


    Nonterminals
    combines combine.

    Terminals msg_number msg_name msg_module.

    Rootsymbol combines.

    combines -> combine : '$1'.
    combines -> combine combines : '$1' ++ '$2'.
    combine -> msg_number msg_name msg_module :
        [{{save('$1'), save('$2')}, {save('$2'), save('$1')}, {save('$1'), save('$3')}}].

    Erlang code.


    save({msg_number, _, Value}) -> Value;
    save({msg_name, _, Value}) -> list_to_atom(Value);
    save({msg_module, _, Value}) -> list_to_atom(Value).

这里的combines语法有点像[H | Rest]语法，第一个combines -> combine : '$1'表示整个输入的词法分析流列表只剩一个元素的做法，combines -> combine combines : ... 表示匹配出单个元素combine 以及剩下的combines，而combine又寻找到单个规则combine -> msg_number msg_name ....，这样递归地处理输入源。

使用方法是：


    yecc:file("test_parser.yrl").

然后编译生成的test_parser.erl文件，再调用test_parser:parse(Tokens).就生成erlang的目标代码了，这里我只生成了一个列表。

# 八、完整代码
我将完整代码用来编写了一个rebar3插件，用户定义协议描述文件xxx，然后里面的内容是协议描述关系：数字,协议名,协议编解码文件 这样的内容，插件就能输出成一个erlang文件。

完整代码参照我的git repo：[rebar3_pb_msgdesc][1]




[1]:https://github.com/xlkness/rebar3_pb_msgdesc