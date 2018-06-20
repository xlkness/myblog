---
title: paxos-simple
date: 2018-06-20 20:25:06
categories:
- 译文
tags:
- 分布式
---

上周给组里分享课程，其中讲到了paxos，觉得没讲好，遂决定看看paxos论文，看的时候有的枯涩的地方就翻译到文本里记录，翻译得越来越多，索性都翻译了吧 ...

# 1. 介绍
在实现一个容错分布式系统时，人们认为Paxos算法理解起来很困难，大概对于很多读者来说最初的陈述是用的希腊语。事实上，它是分布式算法中最简单也最明显的一种。

它本质上是一个共识算法-the “synod” algorithm of。下一节叙述了这种共识算法几乎不可避免地遵循我们想让它满足的属性条件。最后一节解释了完整的Paxos算法：通过简单的状态机的共识程序来构建一个分布式系统-这应该是被广为人知的，因为它在分布式系统理论文章中被引用最多的。

# 2. 一致性算法
## 2.1 问题
假设有一些进程可以提议values。一个共识算法可以保证在这些提议的values中只有一个value被选中。如果没有提议任何value，那么也没有任何value被选中。

如果一个value被选中了，那么这些进程应该能够学习这个被选中value。协商一致的安全性要求如下：

* 只能发起提议的value能被选中
* 只能有一个值被选中
* 并且一个进程只能获取到真正被选中的value

我们不用尝试指定精确的要求。但是目标是保证一些被提议的value最终会被选中，并且当一个value被选中，那么某个进程最终都能获取到这个value。

在共识算法中我们使用三个角色由三个种类代理：proposers、acceptors、learners（提议者、接受者、结果学习者）。在一个具体实现中，单个进程可能同时扮演多种代理，，但是从从代理到进程的映射关系我们并不关心。

假设这些代理能够通过发送消息相互交流。我们使用异步、非拜占庭模型：

* 这些代理的执行是任意速度的，也可能发生故障而停止，也可能重启。因为所有的代理都可能在选中一个value后发生故障并且重启，因此这些代理必须持久化一些信息，即使发生故障和重启（也能恢复）
* 发送的消息可以是任意长度的、可能重复、可能丢失，但是不能被篡改。

## 2.2 选中一个value
最简单的方法是使用单个acceptor代理来选中一个值。某个proposer发送一个提议给acceptor，acceptor选择它接收到的第一条消息的value。虽然简单，但是这种方法不能满足我们的要求，因为假如acceptor发生故障，就会导致接下来的步骤失败。

因此，让我们尝试其它的方式来选择一个value。现在用多个acceptor代理来代替单个acceptor。某个proposer给这些acceptor都发送一个提议value。某个acceptor可能会接受这个value。那么这个value当足够数量的acceptor都接受这个value时，就能通过选中这次value了。多大的数量才足够？为了保证仅仅只有一个value被选中，我们让超过半数的acceptor作为这个足够大的数量。因为对于一个acceptor集合，其中任意两个超过半数的子集合至少有一个公共的acceptor。如果一个acceptor只能选中至多一个值，那么这种方法就是可行的。

在没有故障发生和消息丢失的情况下，我们想要让一个value被选中，哪怕仅仅只有单个proposer发起一次提议value。这就需要满足以下要求：

> P1: An acceptormust accept the first proposal that it receives.
>
> （一个acceptor必须接受它第一个收到的提议的value）

但是这个要求又引起一个问题。多个values可能同时被好几个不同proposers提议，导致了一种情形：每个acceptor都接收到value，但是没有一个value是被超过半数的acceptor所选中。即使只提出了两个提议value，如果每个value各自都被半数acceptor选中，那么任意单个acceptor故障都可能让leaner获取哪一个value被选中变为不可能。

P1规约以及一个value只有被超过半数acceptor所接受才算被选中这两个条件意味着一个acceptor必须能够接受多个提议。我们给不同的提议指定一个编号来追踪这些提议，因此一个提议就由提议号（proposal number）和value组成。为了防止冲突，我们要求不同的提议用不同的提议号。这个不同的提议号如何实现，依赖具体的实现，当前我们先假设它（已经实现）。如果一个提议的value被超过半数的acceptor所接受，那么这个value就被选中。这种情况下，我们就说这个提议被选中。

我们可以允许多个提议被选中，但是我们必须保证这些被选中的提议都拥有相同的value。通过归纳提议号，就足以保证：

>P2: If a proposal with value v is chosen,then every higher-numbered pro-posal that is chosen has value v.
>
>（如果一个value为v的提议被选中，那么其后每一个更高提议号的提议value也要是v）

因为提议号都是有序的，条件P2保证了关键的安全性属性：仅仅只有一个value能被选中。

为了被选中，一个提议必须被至少一个acceptor所接受。因此，我们能通过满足以下条件来满足P2：

>P2a: If aproposal with value v is chosen, then every higher-numbered pro-posal accepted by any acceptor has value v.
>
>（如果一个value为v的提议被选中，那么每个被acceptor所接受的具有更高提议号的提议的value也要是v）

我们依然需要P1来保证有提议能被选中。因为交流是异步的，一个提议可能被一些特殊的acceptor c所选中，但是它们没有接受过其它任何提议。假设一个新的proposer“醒过来（重启或从故障中恢复）”，然后发送了一个带有更高提议号且不同value的提议。P1只要求c接受这个提议，但是却违背了P2a。为了同时满足P1和P2a，需要加强P2a：

>P2b: If a proposal with value v ischosen, then every higher-numbered pro-posal issued by any proposer has value v.
>
>（如果一个value为v的提议被选中，那么所有发送具有更高提议号的proposer的value也是v）

因为一个提议在被acceptor接受之前必须是proposer发出，P2b满足了P2a，也满足了P2。

为了明白如何满足P2b，让我们思考如何证明它。我们假设有一些提议号为m，value为v的提议已经被选中，然后我们证明以后的任何提议号n（n>m）的提议，其value为v。我们可以归纳法到n来简化证明，在这种假设下，我们就证明所有发起的提议号为m..(n-1)的提议，其value都为v，其中i..j表示提议号范围为i-j。既然提议号为m的提议被选中，那么必然有一个超过半数acceptors的集合C接受了它。与归纳假设结合，假设提议m被选中可以推论出：

>Everyacceptor in C has accepted a proposal with number in m ..(n − 1),and every proposal with number in m ..(n − 1) accepted by any acceptor has value v.
>
>（超半数集合C中的每一个acceptor都接受了一个提议号从m..(n-1)的提议，且每一个提议被接受时，value都为v）

因为任意超半数的acceptor集合S中，至少有一个是C的成员，我们可以得出结论，通过确保以下的条件来保持提议n的value为v：

>P2c:For any v and n, if a proposal with value v and number n is issued,then there is a set S consisting of amajority of acceptors such thateither (a) no acceptor in S has acceptedany proposal numbered less than n, or (b) v is the value of the highest-numbered proposal among all proposals numbered less than n acceptedby the acceptors in S.
>
>（对于任意的提议n和value v，如果一个提议号为n，value为v的提议产生了，那么就存在一个包含了超过半数acceptor的集合S，这个集合中要么没有acceptor批准过任意比n小的提议，要么v就是S中的acceptor选中的提议号小于n的最大编号提议所具有的value）

我们可以因此满足P2c来满足P2b。

为了满足P2c，一个proposer想产生一个提议n，它就必须获取提议号小于n且已经或将要被超过半数acceptor接受的最大提议的value。获取已经被接受过的提议是很简单的，但是预测将要被接受的结果就很困难。不用预测未来，proposer只要承诺自己不会获得这样的接受情况就可以了。换句话说，proposer请求所有acceptor不要接受任何比（它自己提议号）n小的提议。这就推导出一下算法来生成提议：

* 一个proposer选择新的提议号n，然后发送请求给所有acceptor，询问它回复如下内容：

    (a). 承诺不再接受比n小的提议

    (b). 如果接受了小于n的提议，就返回接受的提议内容

    我们称这样的请求为带有提议号为n的prepare请求
* 如果proposer接受到超过半数的acceptor响应，接下来它就能发出value为v的提议n，value v要么是响应的消息中acceptor接受过的最大提议号的value，要么是所有acceptor都还没接受value时proposer自己提议的value。

一个proposer发送提议消息给一些acceptor集合，请求它们接受。（这个集合不一定就是初始请求后响应了的acceptor集合）让我们称它为accept请求。

这个描述的是proposer的算法。Acceptor的算法呢？acceptor能接受两种请求：prepare请求和accept请求。一个acceptor能够在不影响安全性的条件下忽略任何请求。所以，我们只需要讨论它可以回复（proposer）请求的情况。它通常都能够响应一个prepare请求。如果acceptor没有承诺不接受某种提议，那么它就能响应这种提议，并接受它。换句话说：

>P1a:An acceptor can accept a proposal numbered n if it has not responded to a prepare request having a numbergreater than n.
>
>（一个acceptor如果没有响应比n大的prepare请求，它就能接受提议n）

可以看到P1a包含P1。

现在我们已经有了一个完整的并能满足安全性的算法来选中一个value—假设使用的都是唯一的提议编号的情况。最终的算法通过一点点的小优化来实现。

假设一个acceptor接收到一个prepare请求n，但是它早已响应过了比n大的prepare请求，因此它承诺不会接受任何提议n。acceptor也没有任何理由响应这次prepare请求，因此它不会接受提议n。我们就让acceptor忽略这种prepare请求。同时我们也让acceptor忽略早就被接受过的prepare请求。

经过这种优化，acceptor仅仅只需要记住它曾接受过的最大编号的提议和它响应过的最大编号的prepare请求。P2c必须被满足哪怕发生了故障，因此acceptor也必须记住这些信息即使发生了故障和重启。注意proposer可以丢弃一个提议然后忘掉所有信息，只要它不会又发送相同编号的提议。

将proposer和acceptor放在一起，我们可以得到算法操作分下面两阶段：

**阶段1：**
* 某个proposer选择一个提议号n，然后用提议号n发送一个prepare请求给超过半数的acceptor；
* 如果一个acceptor接收到提议号为n的prepare请求，且提议号n比它曾经响应过的提议号更大，那么它就响应proposer，响应内容是：保证不会接受任何比n更小的提议以及它曾经接受过的最大编号提议的value（如果有的话）

**阶段2**
* 如果proposer接收到超过半数的acceptor对于提议n的响应，那么它就给每个响应了的acceptor发送一个accept请求，提议号还是n，且value要么是那些acceptor响应消息里带有接受过的最大提议号的value，要么那些acceptor都没有接受任何提议，那value就是proposer自己任选；
* 如果一个acceptor接收到提议号为n的accept请求，它就会接受这次accept请求（并选中请求带有的value）除非它之前又响应过比n更高的prepare请求。

一个proposer可以产生多个提议，只要它能满足算法的每个要求。它也可以在协议进行到任何阶段任何时间丢弃提议。（正确性是能保证的，哪怕某个提议请求或者其响应消息可能在很久之后才到达目的地，而这之前这个提议就被丢弃了。）当有其它proposer已经开始用更高的提议号尝试发起投票了，放弃此次提议是个好主意。因此，如果一个acceptor因为它已经接受过更高提议号的提议而忽略当前prepare或accept请求时，它应该通知那个proposer让它放弃这次提议。这是一个并不影响正确性的优化点。

## 2.3 获取一个被选中的值
（learner）为了习得一个被选中的value，一个learner必须找出被大多数批准者接受的那次提议。明显的算法是让每个acceptor无论什么时候选中了一个提议，就给所有learner响应那次提议的信息。这可以让learners尽快知道选中的value，但是这种方法需要每个acceptor给每个learner发送大量的消息。

非拜占庭模型（non-Byzantine model）错误的假设下，某个learner能够容易地从其它learner那里知道被选中的value。我们可以让acceptor响应批准的投票信息给distinguished learner（主学习者），然后distinguished learner再轮流响应给其它learner。这种方法需要额外一轮操作来让所有learner发现被选中的value。它也是不可靠的，因为distinguished learner可能会发生异常。并且它也需要acceptor的数量加上learner的数量这么多消息响应。

更普遍的做法是，acceptors可以响应它们的批准信息给部分被选中distinguishedlearners，每一个distinguished learner又能通知所有learner。如果distinguished learners的数量很多，这可以保证更好地可靠性，但也带来了交流复杂性上的消耗。

由于分布式中消息可能出现丢失，就可能出现一个提议被选中了但是没有learner知道这次投票结果。learners可以询问acceptors哪一个提案被选中了，假设出现了一个acceptor不知道是哪次提案被选中了或者没有提案出现了超过半数的投票。在这种情况下，只有新的一次提案被选中了，learners才能知道哪一个value被选中。如果一个learner需要知道一个value是否被选中，它可以让一个proposer发起一次这个value的提议。


## 2.4 过程保证
我们可以简单构建一种情形：两个proposer不断地用更高的序号发起投票，但是没有一次投票被选中（活锁）。proposer p用提议号n1完成了阶段1(Phase 1)。接着另一个proposer q又用大于n1的提议号n2完成了阶段1(Phase1)。接着p开始进行阶段2(Phase 2)（因为阶段1用n1收到的超过半数响应因此可以进行阶段2），但是发送的accept请求没有收到响应或者收到的都是拒绝消息，因为n1编号小于此时acceptors维护的编号n2。因此，p又用新的编号n3发起投票，并完成阶段1。再来看proposer q，它当前完成了阶段1，然后发起第二阶段消息请求，但是收到了拒绝的响应（因为n2<n3了），它又用n4发起新投票……如此往复。

为了保证过程（正常进行），必须要选一个distinguished proposer来发起提案。如果这个distinguished proposer能成功地跟超过半数acceptors交流，并且每次都使用比曾经使用的提议编号大的编号，那么它就能成功发起一次提议并被acceptors接受。当distinguished proposer知道自己（发起提议）的编号太低时，通过放弃提议并且重新（用更高的编号）尝试，最终选择一个足够高的编号。

## 2.5 实现
Paxos算法假定一个有多个进程的网络。在它的共识算法中，每个进程同时扮演proposer、acceptor和learner。通过算法选择一个leader来作为distinguished proposer和distinguished learner。Paxos一致性算法正是上面描述的消息和请求都作为普通消息发送的确切实现算法。（响应消息也一样被标记上一致的提议编号来防止混淆。）当异常发生时，稳定的存储用来持久化批准者必须记住的信息。响应proposer之前，acceptor在存储系统中记录它准备要响应的消息。

最后剩下要做的就是描述如何避免两个提议用了两个相同的提议编号。如果不同的proposer从不相交的数字集合里选择他们的提议编号，那么两个提议绝不会产生相同编号的提议。每个proposer持久化它用过的最高的提案编号，然后在开始Paxos一阶段时就使用一个比用过的最高的提案编号更高的编号。

# 3. 实现一个状态机
实现一个分布式系统的简单方式是（将分布式系统的节点）当做一批客户端，都产生命令到一个中央服务器。这个中央服务器能被描述为按某种顺序来执行客户端命令的一个确定性状态机。这个状态机有一个当前状态；它执行一步可以看作：输入一个命令->产生一个输出结果->切换到新状态。举个例，分布式银行系统的客户端可能是出纳员，系统里的状态机维护的状态信息可能包含所有用户的账户余额。那么一次提款就能描述为：仅当余额比提款额大时，执行“减少账户余额”的状态机命令->产生一个包含旧余额和新余额的结果。

仅使用单个中央服务器的实现会面临单点失效问题。因此用一批服务器来代替一个中央服务器，每个服务器都是一个独立的状态机。因为这些状态机是确定性的，如果它们接收并执行的指令序列都是相同的话，那么它们也会产生相同状态和输出结果。一个客户端产生一个状态机命令时就能使用任意一个中央服务器的结果。

为了保证所有的中央服务器执行相同序列的状态机命令，我们实现了由Paxos共识算法的单独实例组成的序列：序列中第i个投票的值就作为第i个状态机命令。每个服务器都扮演所有角色（proposer、acceptor、learner）。现在，我假定一组中央服务器已经确定，因此所有一致性算法的实例都使用一样的集合。

正常操作时，在所有这个一致性算法实例中一个节点被选为leader（唯一的一个能产生提议的节点）。客户端发送多条命令给这个leader，leader来决定这些命令该放在实例集合中的位置。如果leader从这众多命令中决定了一个确切的客户端命令应该作为第135条命令，它就会尝试将这条命令作为这个共识算法的第135个实例。这个尝试通常会成功。以下情况可能会失败：leader出现异常情况、或者有一个其它节点服务器也坚信自己是leader并且对于哪条客户端命令作为第135条命令有不同看法。但是这个共识算法保证至多一个命令能被选择为第135条命令。

这个方法效率的关键是：在这个Paxos共识算法中，value只能到第二阶段(Phase 2)才能被批准。

回忆一下，proposer的算法在完成阶段1之后，要么proposer已经知道acceptor选中了一个value，要么所有acceptors还没有经历过阶段2并同意过任何值。

我现在要描述Paxos算法提到的状态机在正常操作中如何实现。稍后，我会讨论异常情况。我思考当前一个leader崩溃了而新leader被选出时会发生什么。（系统初始运行时还没有命令被投票产生，这是一种特殊情形。）

选出的新的leader也作为这个共识算法所有实例的learner，应该要知道大多数被选择的命令。假设它知道第1-134、138和139个命令，也就是这些实例1-134、138和139选择的value。（稍后我们会看到这种情况如何产生的。）leader接着开始执行第135-137实例的阶段1(Phase 1)以及所有比139大的实例。假设这次的执行结果决定了实例135和140选中的value，但是不影响其它实例。然后leader对实例135和140执行阶段2(Phase 2)，从而为135和140实例选中的客户端命令。

Leader服务器和其它所有服务器此时可以获取leader能够知道的所有命令，那么现在它们可以执行1-135实例的命令。然而，它们不能执行138-140，尽管138-140处于实例集合中能够知道它们的内容，因为命令136和137还没有选出。Leader可以将接下来的两条客户端命令请求作为命令136和137。但是我们不这样做，我们发起一个特殊的”no-op”命令的提议来作为136和137的内容，这样让状态机状态不改变并且也能填充命令实例集合的空缺。（通过执行共识算法实例136和137的阶段2）一旦这些”no-op”命令被选中，命令138-140就能被执行。

  现在命令1-140已经选择完毕了。Leader也完成了共识算法中所有大于140的实例的阶段1，并且可以在这些实例的阶段2提议任何value。它将接下来的客户端请求赋值为141，然后提议这个客户端请求作为共识算法实例141在阶段2的value。它又继续提议接下来的客户端请求为142，如此循环往复。

  Leader能够在还不清楚提议141是否被选中之前就提议命令142。可能提议141命令的所有消息都丢失了，也可能在命令141还没被任何learner知道被选为141命令之前，142命令就被选中。当leader没有接收到期望的关于141实例提议的阶段2响应消息，它就会重新发送这些消息。如果所有顺利进行，它的提议命令将会被选中。然而，它可能一开始就发生故障，使选择的命令序列出现了空缺。通常，假设一个leader能够提前获得α条命令，那也是说，在1-i的命令被选出的情况下，它能提议i+1-i+α的命令。因此空缺可能会有α-1那么大。

  一个新的leader可以执行无数个共识算法实例的阶段1—即在上面的场景中，135-137以及所有大于139的实例。所有实例使用同一个提议号，leader可以给其它服务器发送单个合理的短消息。在阶段1，一个acceptor如果早已接收过其它proposer的阶段2的消息，那么（对于这条prepare请求）它就不仅仅回复个OK。（这种情况仅仅是场景中的135和140实例。）因此，一个服务器（扮演acceptor的）可以用单个合理的短消息响应所有实例。执行无数个阶段1的实例不会产生任何问题。

  leader遭遇故障然后选择一个新的leader是小概率事件，因此执行状态机命令的消耗--即，实现共识算法的command/value—就是共识算法阶段2执行的消耗。可以看到Paxos一致性算法的阶段2在任何达成共识的算法中消耗最小的。因此，Paxos算法是最优的。

  系统正常操作中总是假定有一个leader，除非当前leader发生故障并且正在选择一个新leader。在意外情况中，leader选举可能发生故障。如果没有服务器扮演leader，那么也就没有命令会被提议。如果有多个服务器认为他们是leaders，它们都能对一致性算法的同一实例的提议value，这会导致没有任何value被选中。然而安全性总是保留着—两个不同的服务器永远不会不同意一个value被选为第i个状态机命令。单个leader的选举仅仅是为了保证过程的进行。

  如果服务器的集合可以改变，那么必须要有方法来确定哪些服务器实现了哪些一致性算法实例。最简单的办法就是让状态机自己来做。当前服务器的集合可以作为状态的一部分，并且可以被一般的状态机命令改变。在执行完第i条状态机命令后，我们通过让这个服务器集合执行一致性算法的i-i+α条实例来指定状态的方式，以允许leader提前获取α个命令。这点就可以完成一个任意复杂的重构算法的简单实现。















