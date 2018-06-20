---
title: 【从零开始构建erlang服务器】-01网络库
date: 2018-06-20 20:38:16
categories:
- 开发
tags:
- Erlang
---

# 简介
 网络库是服务器的基础。有了网络库，服务器就能接收外界消息，提供服务。因此开始就从网络库入手。这里先构建基础的TCP通信网络库。至于UDP、WebSocket这些通信手段暂不说。
# gen_tcp
 erlang的底层是c语言封装的，因此其socket通信在beam层也是经过封装的，更易使用。告别跟系统绑定的select/poll/epoll/iocp/kqueue的复杂操作，我们只需要根据gen_tcp的官方文档，就能构建简单好用的tcp服务器。
# 惊群效应
 惊群效应就是多进程或多线程同时阻塞等待相同事件时，假如事件不阻塞而是返回结果了，那么所有阻塞的进程或线程就会唤醒，但是却只有一个进程或线程能获得这个时间的“控制权”，对该事件进行处理，而其它竞争失败的就只能重新休眠。这样多个进程或线程同时唤醒但大部分都做无用功，cpu上下文白切换。

比较典型的就是accept函数。

例如linux的socket监听套接字创建流程是：
1. int socket(int domain, int type, int protocol) //指定协议版本(IPv4/IPv6)，指定通信类型(TCP/UDP/RAW)
2. int bind(int socket, const struct sockaddr *address, socket_t address_len) // 将socket套接字绑定到一个网卡
3. int listen(int socket, int backlog) // 将socket套接字转变为连接套接字
4. int accept(int socket, struct sockaddr *, socklen_t *) // 监听一个客户端的连接

 第1、2步没什么好说，第3步的listen，需要指定一个backlog参数，何谓backlog？linux内核对于tcp三次握手，维护了两个队列，一个用于客户端请求握手的队列，一个用于完成了握手的队列。客户端完成了握手，就从第一个队列转移到第二个队列如果accept接受了请求，就将握手的客户端从第二个队列删除，那么backlog就指定第二个队列的长度。可见，如果客户端connect并发太多间隔太短，而accept的接受速度如果太慢，backlog到了最大长度，就会对新到来的客户端响应RST分节，可见的抓包握手消息就是（三次握手在第一步“你好在吗”就返回错误了）：

 client->server : SYN

 server->client : RST

**个人猜测，未去证实：**

因此可以增大backlog的数量，也可以多几个acceptor来做监听。才有了用多进程、多线程来做accept操作。不过linux2.6内核以后，就改善了accept的惊群问题，多个acceptor。


# gen_tcp:accept/2
gen_tcp的核心函数我觉得就是accept，正如官方文档上写的：
>The accept call does not have to be issued from the socket owner process. Using version 5.5.3 and higher of the emulator, multiple simultaneous accept calls can be issued from different processes, which allows for a pool of acceptor processes handling incoming connections.

文档明确说了可以用多个acceptor来针对同一个监听套接字做客户端监听。

（底层也只有一个OS线程做accept，可能给多个做accept的erlang轻进程做竞争，accept返回即选择一个轻进程分发clientfd，参见https://github.com/alibaba/erlang_multi_pollset)
# 简单tcp服务器代码
官方文档的一个简单实现：

    start(Num,LPort) ->
        case gen_tcp:listen(LPort,[{active, false},{packet,2}]) of
            {ok, ListenSock} ->
                start_servers(Num,ListenSock),
                {ok, Port} = inet:port(ListenSock),
                Port;
            {error,Reason} ->
                {error,Reason}
        end.


    start_servers(0,_) ->
        ok;
    start_servers(Num,LS) ->
        spawn(?MODULE,server,[LS]),
        start_servers(Num-1,LS).


    server(LS) ->
        case gen_tcp:accept(LS) of
            {ok,S} ->
                loop(S),
                server(LS);
            Other ->
                io:format("accept returned ~w - goodbye!~n",[Other]),
                ok
        end.


    loop(S) ->
        inet:setopts(S,[{active,once}]),
        receive
            {tcp,S,Data} ->
                Answer = process(Data), % Not implemented in this example
                gen_tcp:send(S,Answer),
                loop(S);
            {tcp_closed,S} ->
                io:format("Socket ~w closed [~w]~n",[S,self()]),
                ok
        end.

 这个代码就是启动Num数量个acceptors来做客户端连接监听，但是这个代码效率不高，这里的acceptor即做accept工作，又做客户端任务处理的逻辑，既是acceptors pool，也是worker pool，如果客户端请求的服务器任务处理时间太长的话，当前处理进程的accept工作也一起阻塞在那里。

# 优化accept
 这里要提到erlang socket套接字的归属权，初始时只有做accept的进程有权处理返回的socket消息，因此我们上面第四步才能直接在loop函数里receive {tcp, S, Data}处理，但也因此带来了阻塞acceptor的风险。

如果可以转让归属权，是不是我们就能解放acceptor的工作，它只需要做单一的accept工作，一旦有客户端连接，返回了客户端套接字，我们就将归属权转让给其它进程去处理今后所有客户端的请求服务器？

是的。这个函数就是gen_tcp:controlling_process/2。有了这个函数，我们就可以解放acceptor的工作。将socketFd交给其它worker进程去处理。
# tcp调参
经过上面步骤，初步的监听框架有了，并且可以启动很多acceptors来做accept工作，beam以上的accept性能似乎已经没有问题了（只有beam虚拟机用单个OS thread做connect事件监听可能会影响速度，不过我们只需要做好我们代码能做到的就可以了，multi-pollset会在20.3之后某个版本发布）。

余下的工作就是tcp调参了。

众所周知，tcp是个全双工可靠通信，为了实现这个特性，tcp底层实现用了很多算法来支撑、优化tcp。

例如小块数据发送策略-Nagle算法、成块数据流发送策略-滑动窗口、慢启动、超时与重传、发送定时器、keepalive定时器、tcp状态转换等等。

因此，在os系统调用层就能优化很多操作，而beam又在系统api上面封装了socket操作，也会加一些erlang的tcp特性参数。

这里大概说一下重要的一些参数，其它的请参考官方文档：

http://erlang.org/doc/search/?q=meck&x=0&y=0?q=meck&x=0&y=0
## backlog
上文有提到，是一个完成三次握手但还没有被应用层accept的客户端请求最大数量
## nodelay
tcp对待小块数据，是先缓存起来不立即发送（如果小块数据也默认发送，可能会造成广域网消息拥塞），等到数据量多一点再一起发，
这个就是tcp的Nagle算法。而要不延迟发送的服务器可以设置为true，例如telnet客户端、ssh客户端等，按一个键盘按键就要发送一个
按键字母。
## active
分为三种模式：

* true-非阻塞，tcp消息会以{tcp, Socket, Data}的进程普通消息发到socket套接字归属进程
* false-阻塞，必须显式调用gen_tcp:recv/2来接收消息
* once-半阻塞（官方推荐的方式），这种模式是非阻塞的接收一个消息，但是在收到一个消息后，
		必须再次调用inet:setopts(Socket, [{active, once}])来重新设置才能再接收到消息
		（这个的作用是，单纯用true模式太粗暴，如果收速小于发速，beam要盲目接收外界消息
		  然后转换成进程消息，很容易被大量的数据撑爆进程邮箱，而如果用false模式阻塞接收，
		  进程必须自己做recv后的逻辑处理，处理完毕又自己做轮询recv，让消息处理变得复杂，
		  因此有了once模式，即有true模式的消息事件触发方式，又有false模式的消息限速安全性）
## reuseaddr
TCP四次挥手过程中，先发起关闭连接的一方会进入TIME_WAIT状态。TIME_WAIT是指四次挥手中主动方最后一次接收到对方的关闭请求后，自己响应了ACK消息后等待2MSL(最大报文生存时间Maximum Segment Lifetime，一个MSL可能是30s 1m 2m)才能关闭本端。原因是：tcp作为全双工的管道，关闭时也要实现全双工，即双方明确知道都关闭了管道。因此在先发起关闭的一方最后一次响应ACK后要等待对方告诉自己它已经收到ACK消息，而不是说这个ACK消息在还没被对方收到就失去了，如果对方没有收到，就要重新发送ACK，这才是可靠关闭的实现。而这个2MSL期间，再想重新启动这个端口的监听模式是错误的。

举个例， 如果设置不能重用地址， 服务器程序打开了端口8888，这时候遇到一个错误，服务器宕了，监控程序立马检测到服务器宕了，执行重启服务器逻辑，却发现一直重启不了，直到2MSL过了。

# 编写网络库
根据以上总结，我们可以容易编写一个简单网络库，支持大量连接，但一个工业强度网络库的诞生，必然伴随着大量测试的千锤百炼。

这里我实现了一个简单的网络库，还没有测过客户端异常关闭等等socket套接字管理问题：
https://github.com/xlkness/erlnet

因此我们编写网络库，只是做一个学习作用，知道网络库的作用，如何并发连接，消息并发接收等等，因为网络库的重要性，有了网络库，才能进行接下来的协议层，才能针对客户端请求消息提供对应的服务，有了服务才会根据服务器进行更深的业务和方案选择。

真正的使用还是寻找一些开源的网络库，例如ranch：

https://github.com/ninenines/ranch