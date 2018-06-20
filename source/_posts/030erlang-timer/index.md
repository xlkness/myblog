---
title: erlang底层c定时器设计-Erlang源码学习二
date: 2018-06-20 20:09:25
categories:
- 开发
tags:
- Erlang
---

Erlang底层的定时器实现位于源码的erts/emulator/beam/time.c文件，用时间轮的方式动态添加和删除定时器，结构体名为`typedef struct ErtsTimerWheel_ ErtsTimerWheel`，每一个定时器的结构体名为`typedef struct erl_timer ErtsTWheelTimer`，看结构体实现大体上可以知道定时器的设计。

# 定时器 ErtsTWheelTimer

    typedef struct erl_timer {
        struct erl_timer* next; /* next entry tiw slot or chain */
        struct erl_timer* prev; /* prev entry tiw slot or chain */
        union {
        struct {
            void (*timeout)(void*); /* called when timeout */
            void (*cancel)(void*);  /* called when cancel (may be NULL) */
            void* arg;              /* argument to timeout/cancel procs */
        } func;
        ErtsThrPrgrLaterOp cleanup;
        } u;
        ErtsMonotonicTime timeout_pos; /* Timeout in absolute clock ticks */
        int slot;
    } ErtsTWheelTimer;

每个定时器维护了前后向指针，有定时器到时作为回调的函数、取消定时器所调用的函数（可能做参数销毁用）和函数参数，还有定时器的到时点，以及此定时器位于时间轮的槽。


# 时间轮 ErtsTimerWheel

    struct ErtsTimerWheel_ {
        ErtsTWheelTimer *w[ERTS_TIW_SIZE];
        ErtsMonotonicTime pos;
        Uint nto;
        struct {
        ErtsTWheelTimer *head;
        ErtsTWheelTimer *tail;
        Uint nto;
        } at_once;
        int yield_slot;
        int yield_slots_left;
        int yield_start_pos;
        ErtsTWheelTimer sentinel;
        int true_next_timeout_time;
        ErtsMonotonicTime next_timeout_time;
    };

时间轮维护了一个ERTS_TIW_SIZE大小的定时器指针数组，看头文件定义可以得到ERTS_TIW_SIZE在小内存机器上是 1<<13的大小，大内存机器为1<<16=2^16=2^6*1024=65535大小，这里只看大内存机器；接着有一个pos字段，类型为ErtsMonotonicTime，这是一个long long的别名，顾名思义就是erlang的monotonic时间，简单说就是一个精确到纳秒的单调递增时间；接着有一个at_once空间，有头head、尾tail指针，至于数据结构可能为链表，可能为数组实现的栈或队列等；然后的字段光看名字也无法推断了。进入时间轮操作函数。

#

time.c的函数只有几个，先罗列简单的:

    ErtsTimerWheel *
    erts_create_timer_wheel(ErtsSchedulerData *esdp)
    {
        ErtsMonotonicTime mtime;
        int i;
        ErtsTimerWheel *tiw;
        tiw = erts_alloc_permanent_cache_aligned(ERTS_ALC_T_TIMER_WHEEL,
                             sizeof(ErtsTimerWheel));
        for(i = 0; i < ERTS_TIW_SIZE; i++)
        tiw->w[i] = NULL;

        mtime = erts_get_monotonic_time(esdp);
        tiw->pos = ERTS_MONOTONIC_TO_CLKTCKS(mtime);
        tiw->nto = 0;
        tiw->at_once.head = NULL;
        tiw->at_once.tail = NULL;
        tiw->at_once.nto = 0;
        tiw->yield_slot = ERTS_TWHEEL_SLOT_INACTIVE;
        tiw->true_next_timeout_time = 0;
        tiw->next_timeout_time = mtime + ERTS_MONOTONIC_DAY;
        tiw->sentinel.next = &tiw->sentinel;
        tiw->sentinel.prev = &tiw->sentinel;
        tiw->sentinel.u.func.timeout = NULL;
        tiw->sentinel.u.func.cancel = NULL;
        tiw->sentinel.u.func.arg = NULL;
        return tiw;
    }

看操作是先分配内存，然后初始化w定时器指针数组为NULL，接着获取一次当前的monotonic时间，将它转换为时间轮滴答后赋给pos字段，monotonic时间是精确到纳秒，宏ERTS_MONOTONIC_TO_CLKTCKS将它除以了1000*1000，从这里我们可以知道时间轮每一次走动是1ms，即时间轮的粒度就是1ms了，接下来的操作就是常规的初始化了，到tiw->sentinel.next = $tiw->sentinel语句开始，是将一个sentinel（哨兵）变量变为一个指向自己的循环双向链表。

**结论：**

时间轮的pos字段初始值为创建时间轮时的monotonic时间，但时间轮的精度为ms，故需要将monotonic时间转换为ms（除以1000*1000），pos字段为时间轮的当前指针（想象成钟的分针）。

# 插入定时器 insert_timer_into_slot

    static ERTS_INLINE void
    insert_timer_into_slot(ErtsTimerWheel *tiw, int slot, ErtsTWheelTimer *p)
    {
        ERTS_TW_ASSERT(slot >= 0);
        ERTS_TW_ASSERT(slot < ERTS_TIW_SIZE);
        p->slot = slot;
        if (!tiw->w[slot]) {
        tiw->w[slot] = p;
        p->next = p;
        p->prev = p;
        }
        else {
        ErtsTWheelTimer *next, *prev;
        next = tiw->w[slot];
        prev = next->prev;
        p->next = next;
        p->prev = prev;
        prev->next = p;
        next->prev = p;
        }
    }

先看插入的第1、2两句，断言slot要介于0-ERTS_TIW_SIZE之间：定时器要插到时间轮的槽上，因此必须介于这个范围。然后开始插入，先判断待插入的槽有没有定时器，如果没有，就直接将w[slot]指针指向这个定时器，并且赋值next、prev指针保证循环双向链表特性；如果槽上已经有了别的定时器，那么看else的操作是将待插入的定时器头插到链表中。

于是看完这个函数，知道了时间轮的主要逻辑如图：

![erlang_timer1]

**结论：**

时间轮的槽大小为65535；每个槽是一个定时器指针，指针又维护了一个定时器双向循环链表，跟链式散列表很像；定时器是头插。


# 去除定时器 remove_timer

    static ERTS_INLINE void
    remove_timer(ErtsTimerWheel *tiw, ErtsTWheelTimer *p)
    {
        int slot = p->slot;
        ERTS_TW_ASSERT(slot != ERTS_TWHEEL_SLOT_INACTIVE);

        if (slot >= 0) {
            /*
             * Timer in wheel or in circular
             * list of timers currently beeing
             * triggered (referred by sentinel).
             */
            ERTS_TW_ASSERT(slot < ERTS_TIW_SIZE);
            if (p->next == p) {
                ERTS_TW_ASSERT(tiw->w[slot] == p);
                tiw->w[slot] = NULL;
            }
            else {
                if (tiw->w[slot] == p)
                tiw->w[slot] = p->next;
                p->prev->next = p->next;
                p->next->prev = p->prev;
            }
        }
        else {
            /* Timer in "at once" queue... */
            ERTS_TW_ASSERT(slot == ERTS_TWHEEL_SLOT_AT_ONCE);
            if (p->prev)
                p->prev->next = p->next;
            else {
                ERTS_TW_ASSERT(tiw->at_once.head == p);
                tiw->at_once.head = p->next;
            }
            if (p->next)
                p->next->prev = p->prev;
            else {
                ERTS_TW_ASSERT(tiw->at_once.tail == p);
                tiw->at_once.tail = p->prev;
            }
            ERTS_TW_ASSERT(tiw->at_once.nto > 0);
            tiw->at_once.nto--;
        }

        p->slot = ERTS_TWHEEL_SLOT_INACTIVE;
        tiw->nto--;
    }

先看第一个断言slot != ERTS_TWHEEL_SLOT_INACTIVE，这个宏值为-2，前面的函数知道槽数一定是介于0-65535之间，所以猜测如果槽数为-2了，表示定时器未激活。

往后看，如果槽存在，又分两种情况，一种是这个定时器所处的槽只有它一个定时器，那么需要将槽指针w[slot]置为空，另一种是槽上还有很多定时器，则从循环双向链表中取下一个结点。
如果槽不存在，且看else的slot为宏值ERTS_TWHEEL_SLOT_AT_ONCE，那么就从at_once队列中去除定时器，并且nto字段减1。

将定时器的slot字段置为ERTS_TWHEEL_SLOT_INACTIVE，时间轮的nto字段减1。


**结论：**

定时器有三种状态分别为正常、at_once、未激活；at_once队列实则为不循环双向链表；at_once的nto字段记录这个队列上的定时器个数；tiw的nto字段记录所有定时器包括at_once队列上的定时器个数。

# 定时器到时回调 timeout_timer
回调就很简单，将定时器的slot字段设置为未激活，然后调用回调函数

# 取消定时器 erts_twheel_cancel_timer
逻辑与4的到时回调差不多，判断了定时器的slot不能为未激活状态，然后调用remove去除定时器，接着调用定时器的cancel回调函数

# 创建定时器 erts_twheel_set_timer

    void
    erts_twheel_set_timer(ErtsTimerWheel *tiw,
                  ErtsTWheelTimer *p, ErlTimeoutProc timeout,
                  ErlCancelProc cancel, void *arg,
                  ErtsMonotonicTime timeout_pos)
    {
        ErtsMonotonicTime timeout_time;
        ERTS_MSACC_PUSH_AND_SET_STATE_M_X(ERTS_MSACC_STATE_TIMERS);

        p->u.func.timeout = timeout;
        p->u.func.cancel = cancel;
        p->u.func.arg = arg;

        ERTS_TW_ASSERT(p->slot == ERTS_TWHEEL_SLOT_INACTIVE);

        if (timeout_pos <= tiw->pos) {
        tiw->nto++;
        tiw->at_once.nto++;
        p->next = NULL;
        p->prev = tiw->at_once.tail;
        if (tiw->at_once.tail) {
            ERTS_TW_ASSERT(tiw->at_once.head);
            tiw->at_once.tail->next = p;
        }
        else {
            ERTS_TW_ASSERT(!tiw->at_once.head);
            tiw->at_once.head = p;
        }
        tiw->at_once.tail = p;
        p->timeout_pos = tiw->pos;
        p->slot = ERTS_TWHEEL_SLOT_AT_ONCE;
        timeout_time = ERTS_CLKTCKS_TO_MONOTONIC(tiw->pos);
        }
        else {
        int slot;

        /* calculate slot */
        slot = (int) (timeout_pos & (ERTS_TIW_SIZE-1));

        insert_timer_into_slot(tiw, slot, p);

        tiw->nto++;

        timeout_time = ERTS_CLKTCKS_TO_MONOTONIC(timeout_pos);
        p->timeout_pos = timeout_pos;
        }

        if (timeout_time < tiw->next_timeout_time) {
        tiw->true_next_timeout_time = 1;
        tiw->next_timeout_time = timeout_time;
        }
        ERTS_MSACC_POP_STATE_M_X();
    }

逻辑很清楚：传入一个时间轮、定时器、以及定时器要用的相关函数、时间轮上的超时位置（monotonic time / 1000*1000）。

然后判断超时位置是否小于等于时间轮当前的指针pos，如果是，就把它加入到at_once链表，pos的精度为ms，这个at_once的意思就是加入的定时器差1ms就要到时，而针对这种定时器，再把它插入到槽里做管理和到时是没有意义的，因为马上就到时了。

正常的定时器则可以插入到槽里了，槽的计算是用到时位置与槽总大小做与运算，举个例子：当前monotonic时间为10,000,000,000，表示开始或者erlang虚拟机开启了10s， 此时创建了一个时间轮，它的pos就该为10,000，然后插入一个5,000,000,000纳秒后到时的定时器，因为时间轮精度为ms，顾折算为(10,000,000,000 + 5,000,000,000)/1000*1000=15,000，即timeout_pos就为15000，那么timeout_pos & ERTS_TIW_SIZE = 15000，那么槽就是15000位置，此时槽还在10000位置，要走5000个滴答才到，同理，如果插入一个距现在65536ms后到时的定时器，则65536超出了65535，但与运算，又变为了0，实现了定时器的循环相加。

相应nto计数加一，然后判断加入的定时器的到时时间是否小于等于时间轮的下一次到时时间，如果是，就更新时间轮的相应到时值。

**总结：**

定时器如果马上（差1ms）到时的，会加入到at_once队列，否则加入到时间槽里做管理；定时器的到时时间为一个精度为ms的值，然后用这个值跟ERTS_TIW_SIZE做与运算，保证了槽的循环；时间轮还有字段用来表示下一次最近的到时时间，true_next_timeout_time为1表示存在这个时间（即槽上至少存在一个激活的定时器还没到时）。

# 寻找下一个最近到时时间 find_next_timeout

    static ERTS_INLINE ErtsMonotonicTime
    find_next_timeout(ErtsSchedulerData *esdp,
              ErtsTimerWheel *tiw,
              int search_all,
              ErtsMonotonicTime curr_time,       /* When !search_all */
              ErtsMonotonicTime max_search_time) /* When !search_all */
    {
        int start_ix, tiw_pos_ix;
        ErtsTWheelTimer *p;
        int true_min_timeout = 0;
        ErtsMonotonicTime min_timeout, min_timeout_pos, slot_timeout_pos;

        if (tiw->nto == 0) { /* no timeouts in wheel */
            if (!search_all)
                min_timeout_pos = tiw->pos;
            else {
                curr_time = erts_get_monotonic_time(esdp);
                tiw->pos = min_timeout_pos = ERTS_MONOTONIC_TO_CLKTCKS(curr_time);
            }
            min_timeout_pos += ERTS_MONOTONIC_TO_CLKTCKS(ERTS_MONOTONIC_DAY);
            goto found_next;
        }

        slot_timeout_pos = min_timeout_pos = tiw->pos;
        if (search_all)
           min_timeout_pos += ERTS_MONOTONIC_TO_CLKTCKS(ERTS_MONOTONIC_DAY);
        else
           min_timeout_pos = ERTS_MONOTONIC_TO_CLKTCKS(curr_time + max_search_time);

        start_ix = tiw_pos_ix = (int) (tiw->pos & (ERTS_TIW_SIZE-1));

        do {
            if (++slot_timeout_pos >= min_timeout_pos)
                break;

            p = tiw->w[tiw_pos_ix];

            if (p) {
                ErtsTWheelTimer *end = p;

                do  {
                ErtsMonotonicTime timeout_pos;
                timeout_pos = p->timeout_pos;
                if (min_timeout_pos > timeout_pos) {
                    true_min_timeout = 1;
                    min_timeout_pos = timeout_pos;
                    if (min_timeout_pos <= slot_timeout_pos)
                    goto found_next;
                }
                p = p->next;
                } while (p != end);
            }

            tiw_pos_ix++;
            if (tiw_pos_ix == ERTS_TIW_SIZE)
                tiw_pos_ix = 0;
        } while (start_ix != tiw_pos_ix);

    found_next:

        min_timeout = ERTS_CLKTCKS_TO_MONOTONIC(min_timeout_pos);
        tiw->next_timeout_time = min_timeout;
        tiw->true_next_timeout_time = true_min_timeout;

        return min_timeout;
    }


函数作用是寻找时间轮所处指针到当前时间curr_time之间最近的一个定时器到时时间。

函数逻辑分两种情况，一种是时间轮上没有定时器，则判断search_all的值是否要将时间轮的指针拨到当前时间点，然后最小超时时间就为明天的这个时候（因为没有定时器，自然不存在下一个到时的定时器时间）；另一种是时间轮上有定时器，则判断search_all的值是，如果为1，寻找的间隔就是一天(24*60*60*1000)，否则间隔就是时间轮当前指针到curr_time+max_search_time的距离，然后从时间轮当前指针处开始循环判断每个槽链表，有无定时器的到时时间小于curr_time+max_search_time，如果找了一圈（即走过的距离为ERTS_TIW_SIZE）没找到，就退出，并设置时间轮的下一次到时时间。

**结论：**

时间轮维护了一个下一次到时时间，避免了一段连续的槽上都没有定时器，而在做到时判断时空循环破坏效率。

# 时间轮嘀嗒 erts_bump_timers

    void
    erts_bump_timers(ErtsTimerWheel *tiw, ErtsMonotonicTime curr_time)
    {
        int tiw_pos_ix, slots, yielded_slot_restarted, yield_count;
        ErtsMonotonicTime bump_to, tmp_slots, old_pos;
        ERTS_MSACC_PUSH_AND_SET_STATE_M_X(ERTS_MSACC_STATE_TIMERS);

        yield_count = ERTS_TWHEEL_BUMP_YIELD_LIMIT;

        /*
         * In order to be fair we always continue with work
         * where we left off when restarting after a yield.
         */

        if (tiw->yield_slot >= 0) {
            yielded_slot_restarted = 1;
            tiw_pos_ix = tiw->yield_slot;
            slots = tiw->yield_slots_left;
            bump_to = tiw->pos;
            old_pos = tiw->yield_start_pos;
            goto restart_yielded_slot;
        }

        do {

            yielded_slot_restarted = 0;

            bump_to = ERTS_MONOTONIC_TO_CLKTCKS(curr_time);

            while (1) {
                ErtsTWheelTimer *p;

                old_pos = tiw->pos;

                if (tiw->nto == 0) {
                    empty_wheel:
                    ERTS_DBG_CHK_SAFE_TO_SKIP_TO(tiw, bump_to);
                    tiw->true_next_timeout_time = 0;
                    tiw->next_timeout_time = curr_time + ERTS_MONOTONIC_DAY;
                    tiw->pos = bump_to;
                    tiw->yield_slot = ERTS_TWHEEL_SLOT_INACTIVE;
                            ERTS_MSACC_POP_STATE_M_X();
                    return;
                }

                p = tiw->at_once.head;
                while (p) {
                    if (--yield_count <= 0) {
                        ERTS_TW_ASSERT(tiw->nto > 0);
                        ERTS_TW_ASSERT(tiw->at_once.nto > 0);
                        tiw->yield_slot = ERTS_TWHEEL_SLOT_AT_ONCE;
                        tiw->true_next_timeout_time = 1;
                        tiw->next_timeout_time = ERTS_CLKTCKS_TO_MONOTONIC(old_pos);
                                ERTS_MSACC_POP_STATE_M_X();
                        return;
                    }

                    ERTS_TW_ASSERT(tiw->nto > 0);
                    ERTS_TW_ASSERT(tiw->at_once.nto > 0);
                    tiw->nto--;
                    tiw->at_once.nto--;
                    tiw->at_once.head = p->next;
                    if (p->next)
                        p->next->prev = NULL;
                    else
                        tiw->at_once.tail = NULL;

                    timeout_timer(p);

                    p = tiw->at_once.head;
                }

                if (tiw->pos >= bump_to) {
                    ERTS_MSACC_POP_STATE_M_X();
                    break;
                }

                if (tiw->nto == 0)
                    goto empty_wheel;

                if (tiw->true_next_timeout_time) {
                    ErtsMonotonicTime skip_until_pos;
                    /*
                     * No need inspecting slots where we know no timeouts
                     * to trigger should reside.
                     */

                    skip_until_pos = ERTS_MONOTONIC_TO_CLKTCKS(tiw->next_timeout_time);
                    if (skip_until_pos > bump_to)
                        skip_until_pos = bump_to;

                    skip_until_pos--;

                    if (skip_until_pos > tiw->pos) {
                        ERTS_DBG_CHK_SAFE_TO_SKIP_TO(tiw, skip_until_pos);

                        tiw->pos = skip_until_pos;
                    }
                }

                tiw_pos_ix = (int) ((tiw->pos+1) & (ERTS_TIW_SIZE-1));
                tmp_slots = (bump_to - tiw->pos);
                if (tmp_slots < (ErtsMonotonicTime) ERTS_TIW_SIZE)
                  slots = (int) tmp_slots;
                else
                  slots = ERTS_TIW_SIZE;

                tiw->pos = bump_to;

                while (slots > 0) {

                    p = tiw->w[tiw_pos_ix];
                    if (p) {

                        if (p->next == p) {
                            ERTS_TW_ASSERT(tiw->sentinel.next == &tiw->sentinel);
                            ERTS_TW_ASSERT(tiw->sentinel.prev == &tiw->sentinel);
                        } else {
                            tiw->sentinel.next = p->next;
                            tiw->sentinel.prev = p->prev;
                            tiw->sentinel.next->prev = &tiw->sentinel;
                            tiw->sentinel.prev->next = &tiw->sentinel;
                        }

                        tiw->w[tiw_pos_ix] = NULL;

                        while (1) {

                            if (p->timeout_pos > bump_to) {
                                /* Very unusual case... */
                                ++yield_count;
                                insert_timer_into_slot(tiw, tiw_pos_ix, p);
                            } else {
                                /* Normal case... */
                                timeout_timer(p);
                                tiw->nto--;
                            }

                            restart_yielded_slot:

                            p = tiw->sentinel.next;

                            if (p == &tiw->sentinel) {
                                ERTS_TW_ASSERT(tiw->sentinel.prev == &tiw->sentinel);
                                break;
                            }

                            if (--yield_count <= 0) {
                                tiw->true_next_timeout_time = 1;
                                tiw->next_timeout_time = ERTS_CLKTCKS_TO_MONOTONIC(old_pos);
                                tiw->yield_slot = tiw_pos_ix;
                                tiw->yield_slots_left = slots;
                                tiw->yield_start_pos = old_pos;
                                ERTS_MSACC_POP_STATE_M_X();
                                return; /* Yield! */
                            }

                            tiw->sentinel.next = p->next;
                            p->next->prev = &tiw->sentinel;
                        }
                    }
                    tiw_pos_ix++;
                    if (tiw_pos_ix == ERTS_TIW_SIZE)
                        tiw_pos_ix = 0;
                    slots--;
                }
            }

        } while (yielded_slot_restarted);

        tiw->yield_slot = ERTS_TWHEEL_SLOT_INACTIVE;
        tiw->true_next_timeout_time = 0;
        tiw->next_timeout_time = curr_time + ERTS_MONOTONIC_DAY;

        /* Search at most two seconds ahead... */
        (void) find_next_timeout(NULL, tiw, 0, curr_time, ERTS_SEC_TO_MONOTONIC(2));
        ERTS_MSACC_POP_STATE_M_X();
    }


这是最重要的一个函数，erlang虚拟机启动后，有一个线程做周期性调用，来检测有无定时器到时。

函数接收一个curr_time形参，将时间轮上小于等于此时间的定时器都视为到时，所以估计是1ms调用一次。

函数定义了yield_count=100，如果at_once或者某个槽上大于100个定时器，就丢弃多的。

这个函数写得很恶心，又是do while{}，又是while(1)，又是while，但剥离开，真正的逻辑就一段：循环将at_once链表的定时器全部到时，则at_once链表清空了；开始判断时间槽，先利用下一个最近的到时时间next_timeout_time跳过一段槽，然后开始遍历从时间轮的当前指针pos到curr_time之间的间隔槽，再遍历每个槽上的链表，对每个结点判断是否大于等于curr_time，即判断是否到时，如果到时就可以去掉定时器，并执行回调任务。

以上步骤就做完了到时任务，调用一下find_next_timeout寻找一次最近到时时间。

#

在看erts_bump_timers函数时候看到一段goto的代码形如：

    goto test_label:

    int a = 0;

    test_label:

        a = 1;

当时很诧异，a不是没定义吗？激动得不行，摩拳擦掌准备提bug，抱着谨慎的态度还是查了一下，这种用法是可以的，真是菜得不行 …… 自己猜想一下可能是编译期已经将a加入了符号表，goto只影响运行时。

[erlang_timer1]:/img/erlang_timer1.png ""

