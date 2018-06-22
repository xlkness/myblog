---
title: 【从零开始构建erlang服务器】-04协议层
date: 2018-06-22 20:06:51
categories:
- 开发
tags:
- Erlang
---

# 一、简介
协议的作用很重要，通信协议可以理解为两个节点之间为了协同工作实现信息交换，协商一定的规则和约定，例如规定字节序，各个字段类型，使用什么压缩算法或加密算法等。常 见的有tcp，udo，http，sip等常见协议。协议有流程规范和编码规范。流程如呼叫流程等信令流程，编码规范规定所有信令和数据如何打包/解包。编码规范就是我们通常所说的编解码，序列化。不光是用在通信工作上，在存储工作上我们也经常用到。如我们经常想把内存中对象存放到磁盘上，就需要对对象进行数据序列化工作。

本文主要使用google protocol buffer协议。协议就不多介绍了，网上搜索有很多讲解。

# 二、编写proto文件
协议规定了两方交流的方式，因此要有统一的双方都认识的东西来规定协议，例如json、xml，对于protocolbuf，就要定义proto文件了。

协议一般都跟服务器逻辑相关，因此我们先明确服务器要提供什么功能，再来编写协议。

这里我们可以做一个五子棋游戏服务器，那么游戏提供以下功能：

（先不做登陆功能，假设只要是客户端跟我们服务器完成三次握手就能进入游戏）


1、大厅，每隔3s刷新显示所有房间人数和玩家准备状态（先不提供按等级划分的大厅例如菜鸟场、新手场、中级场等）;
2、大厅提供直选房间加入方式，也提供匹配方式加入房间（优先匹配有一个人落座的房间）；
3、房间，最多坐两个人，多余的人会进入观战席，座位上的玩家可以准备，可以取消准备，玩家从进入房间开始计时，计时到了还没准备就踢出房间，不管中途多少次准备、未准备，一旦双方准备，则游戏开始，随机分配先手方；
4、对战，无禁手，无交换，无放弃落子（往简单做），双方各有2分钟总的思考时间，用完则其每一步均只有10s思考时间；
5、对战中异常检测，3s一次心跳，如果客户端下线或者退出房间，则另一方赢；
6、房间预先创建20个，如果只剩5个空房间，则多创建5个保留；
7、点选加入房间如果人满，返回人满错误提示玩家重新选房间

根据需求我们可以制定协议了。

创建协议文件gobang.proto，复制以下内容到协议（协议只是随便定义了下，没精力了，后续写代码再优化吧）：


    enum ChessPieceColor {
        BALCK = 1;
        RED = 2;
    }


    // 心跳
    message HeartBeatReq {}
    message HeartBeatResp {}


    // 获取大厅房间人数，客户端可以自己间隔获取
    message GetRoomsInfoReq {}
    message GetRoomsInfoResp {
        repeated roomInfoType rooms = 1;
    }


    // 加入房间
    message JoinRoomReq {
        required uint32 join_type = 1;      // 1:随机匹配 2:选房加入
        required string nick_name = 2;      // 未做注册、登陆，直接给昵称
        required uint32 sex = 3;            // 性别
        optional uint32 room_id = 4;        // 选房加入时的房间号
    }
    message JoinRoomResp {
        required bool audience = 1;         // 是否观众席位（加入瞬间座位满了）
        required roomInfoType room = 2;     // 加入的房间信息
        optional uint32 remain_sec = 3;     // 不准备的剩余秒数，到时即踢
    }


    // 准备
    message PrepareReq {
        required uint32 type = 1;           // 1:准备 2:取消准备
    }
    message PrepareResp {
        required uint32 remain_sec = 1;     // 不准备的剩余秒数，到时即踢
    }


    // 游戏开始-服务器主动推送给客户端
    message StartGamePush {
        required ChessPieceColor color = 1; // 我方颜色，黑棋先（如果是围观群众，不读这个字段）
        required uint32 remain_sec = 2;     // 思考的剩余时长，如果超时了，每一次落子只有per_round_remain_sec
        required uint32 per_round_remain_sec = 3;   // 思考的总时间花费完了，之后每一步只有这么长的时间思考，超过即输
        repeated startDetailType detail = 4;
        message startDetailType {       // 推送给围观群众的
            required ChessPieceColor color = 1;
            required userInfoType user = 2;
        }
    }


    // 游戏进行
    message PlayGameReq {
        required uint32 x_pos = 1;
        required uint32 y_pos = 2;
    }
    message PlayGameResp {
        required uint32 err_code = 1;
        required uint32 remain_sec = 2;
    }


    // 服务器推送其他玩家落子
    message PlayBroadcastPush {
        required ChessPieceColor color = 1;         // 棋子颜色
        required uint32 x_pos = 2;
        required uint32 y_pos = 3;
    }


    // 游戏结束
    message EndGamePush {
        required userInfoType winner = 1;           // 赢家
    }


    // 玩家加入房间、退出房间推送
    message UserInOutPush {
        required uint32 type = 1;                   // 1.进入座位、2.进入观众席、3.退出座位、4.退出观众席
        optional seatInfoType seat = 2;             // 座位变动才有
        optional userInfoType user = 3;                 // 观众席变动才有
    }


    // 座位玩家加入观众席
    message JoinAudienceReq {}
    message JoinAudienceResp {
        required uint32 err_code = 1;
    }


    // 观众席加入座位
    message JoinSeatReq {}
    message JoinSeatResp {
        required uint32 err_code = 1;
    }






    /*内部声明*/
    // 房间信息
    message roomInfoType {
        required uint32 room_id = 1;        // 房间id
        repeated seatInfoType seats = 2;    // 座位信息
        repeated userInfoType audiences = 3;// 围观群众
    }
    // 座位信息
    message seatInfoType {
        required uint32 index = 1;          // 座位编号
        optional userInfoType user = 2;     // 座位用户信息，空座位没有用户
    }
    // 用户信息
    message userInfoType {
        required uint64 user_id = 1;        // 用户id
        required string nick_name = 2;      // 昵称
    }


# 三、编译协议文件
## 1、添加协议描述文件
目前我看到的erlang protocolbuf通信，都是将proto结构体编码成二进制，然后再在二进制头上补一个16位的协议号，这个协议号是前端和后端自定义的，也可以说是公司层面的协议号，例如这个项目组使用100-999，那个项目组使用1000-1999这样子。那么协议流发出去就是一个<<MsgID:16, Bin/binary>>这样一个二进制流，然后接收端再提取前16位的协议号。通过协议号找到真正的协议名，然后路由到真正的协议编解码文件进行处理，例如有多个.proto文件，经过pb工具生成编解码工具文件就有多个，如何路由到哪个编解码文件，就需要手工写一个描述关系。

将自定义的协议描述文件生成对应的erlang文件的工具我看到的很多都是用riak官方的一个rebar插件，这里我自己在学习lexer/yecc编译工具的时候写了一个rebar3插件rebar3_pb_msgdesc，功能一样。我就使用自己的插件来生成了，具体用法参考插件的readme文件。

针对第二步的协议内容，我们编写一个erlserver_pb_desc.csv文件，格式 协议号，协议名，协议文件 输入以下内容：


    1,HeartBeatReq,erlserver_gobang_pb
    2,HeartBeatResp,erlserver_gobang_pb
    100,GetRoomsInfoReq,erlserver_gobang_pb
    101,GetRoomsInfoResp,erlserver_gobang_pb
    102,JoinRoomReq,erlserver_gobang_pb
    103,JoinRoomResp,erlserver_gobang_pb
    104,PrepareReq,erlserver_gobang_pb
    105,PrepareResp,erlserver_gobang_pb
    106,StartGamePush,erlserver_gobang_pb
    107,PlayGameReq,erlserver_gobang_pb
    108,PlayGameResp,erlserver_gobang_pb
    109,PlayBroadcastPush,erlserver_gobang_pb
    110,EndGamePush,erlserver_gobang_pb
    111,UserInOutPush,erlserver_gobang_pb
    112,JoinAudienceReq,erlserver_gobang_pb
    113,JoinAudienceResp,erlserver_gobang_pb
    114,JoinSeatReq,erlserver_gobang_pb
    115,JoinSeatResp,erlserver_gobang_pb

然后按照我的插件readme配置，执行rebar3 compile就能生成文件了，注意配置生成的描述文件可以放在src/proto/目录。

## 2、生成协议编解码文件
生成协议编解码文件现在都推荐gpb库了，而rebar3插件rebar3_gpb_plugin就可以自动使用gpb作为插件并生成协议编解码文件，按照插件readme的配置，我们将编解码文件生成到src/proto/目录，配置好了rebar3钩子(hook)，直接执行compile就能生成协议编解码文件。
（如还是很模糊，可直接参考这个项目[github repo][1]）

# 四、测试协议收发
在项目根目录创建一个test目录，然后添加一个proto_tst.erl文件，输入以下内容：

    %%%-------------------------------------------------------------------
    %%% @author lkness
    %%% @copyright (C) 2018, <COMPANY>
    %%% @doc
    %%%
    %%% @end
    %%% Created : 14. Apr 2018 6:27 PM
    %%%-------------------------------------------------------------------
    -module(proto_tst).


    %% API
    -export([
        start/0
    ]).


    start() ->
        {ok, Fd} = gen_tcp:connect({127,0,0,1}, 8888, [binary, {packet, raw}, {active, once}]),
        GetRoomsInfoReq = #{},
        MsgName = erlserver_pb_desc:msg_type(100),
        B = erlserver_gobang_pb:encode_msg(GetRoomsInfoReq, MsgName),
        SendMsg = <<100:16, B/binary>>,
        gen_tcp:send(Fd, SendMsg),
        receive
            {tcp, _Socket, Packet} ->
                <<Cmd:16, Bin/binary>> = Packet,
                RecvMsgName = erlserver_pb_desc:msg_type(Cmd),
                Msg = erlserver_gobang_pb:decode_msg(Bin, RecvMsgName),
                io:format("sender recv msg:~w/~p~n", [Cmd, Msg])
        after 5000 ->
            io:format("error, not receive from server~n")
        end,


        ok.


然后在我们的user处理模块里，接收socket消息里面添加响应逻辑：


    handle_info({tcp, Socket, <<Cmd:16, Binary/binary>> = Data}, State=#state{
    socket=Socket, transport=Transport})
    when byte_size(Data) > 1 ->
    Transport:setopts(Socket, [{active, once}]),
    {ok, PeerName} = inet:peername(Socket),
    MsgName = erlserver_pb_desc:msg_type(Cmd),
    Msg = erlserver_gobang_pb:decode_msg(Binary, MsgName),
    lager:info("receive from client[~w], msg:~p/~p~n", [PeerName, Cmd, Msg]),
    SendMsg = #{rooms => [#{room_id => 123, seats => [], audiences => []}]},
    SendBin = erlserver_gobang_pb:encode_msg(SendMsg, erlserver_pb_desc:msg_type(101)),
    Transport:send(Socket, <<101:16, SendBin/binary>>),
    {noreply, State, ?TIMEOUT};

可以看到，就是个简单的收发交互，主要是查看协议是否正常编解码和协议描述文件是否正常生成和对应。

一切就绪，我们可以在测试环境运行rebar shell（因为我们的测试文件放在test目录，default环境启动的话默认不加载test目录），


    rebar3 shell
    (erlserver@127.0.0.1)6> test_pb:start().
    sender recv msg:101/#{rooms =>
                              [#{audiences => [],room_id => 123,seats => []}]}
    2018-04-16/20:42:22.774[info][erlserver_user:handle_info:55]|receive from client[{{127,0,0,1},36069}], msg:100/#{}
    ok

好，不出意外，就打印了收发消息，我们也看到了收到的字节流正常解码为了协议字段信息。

# 五、总结
本节根据具体需求定义了协议文件，然后能将协议文件生成编解码工具，最后还测试了协议收发。

后面就可以开始编写我们的五子棋游戏了（本来想做个聊天室服务器，感觉太烂大街了，并且im都用ejjabered，自己随手写太low了）。

（项目[repo][1]。。。



[1]:https://github.com/xlkness/erlserver