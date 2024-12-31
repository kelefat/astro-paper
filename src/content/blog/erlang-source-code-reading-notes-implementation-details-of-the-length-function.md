---
author: Kelefat
pubDatetime: 2024-12-31T16:40:00Z
modDatetime: 2024-12-31T16:40:00Z
title: Erlang 源码阅读笔记：length 函数的实现细节
tags:
  - Erlang
  - Erlang源码
description:
  Erlang 源码阅读笔记：length 函数的实现细节
---


本文基于Erlang OTP 27

源码路径：otp_src_27.2\erts\emulator\beam\erl_bif_guard.c

## 问题：
在erlang下，我们要获取列表的长度，可以使用length函数
```erlang
length([1,2,3]).
```
那么length是如何计算长度的呢？

## 分析：
length函数为bif调用，在erl_bif_guaid.c下有如下实现：
```c
/*
 * This version of length/1 is called from native code and apply/3.
 */

BIF_RETTYPE length_1(BIF_ALIST_1)
{
    Eterm args[3];

    /*
     * Arrange argument registers the way expected by
     * erts_trapping_length_1(). We save the original argument in
     * args[2] in case an error should signaled.
     */

    args[0] = BIF_ARG_1; /* 需要计算长度的列表，如[1,2,3] */
    args[1] = make_small(0);
    args[2] = BIF_ARG_1;
    return erlang_length_trap(BIF_P, args, A__I);
}
```

我们看看erlang_length_trap做了什么？
```c
static BIF_RETTYPE erlang_length_trap(BIF_ALIST_3)
{
    Eterm res;

    res = erts_trapping_length_1(BIF_P, BIF__ARGS);  /* 主要实现逻辑*/
    if (is_value(res)) {        /* Success. */ 
        BIF_RET(res);		/* 如果计算完成，返回结果 */
    } else {                    /* Trap or error. */
        if (BIF_P->freason == TRAP) {	//否则等待下次调度*/
            /*
             * The available reductions were exceeded. Trap.
             */
            BIF_TRAP3(&erlang_length_export, BIF_P, BIF_ARG_1, BIF_ARG_2, BIF_ARG_3);
        } else {
            /*
             * Signal an error. The original argument was tucked away in BIF_ARG_3.
             */
            ERTS_BIF_ERROR_TRAPPED1(BIF_P, BIF_P->freason,
                                    BIF_TRAP_EXPORT(BIF_length_1), BIF_ARG_3);
        }
    }
}
```

我们看看erts_trapping_length_1做了什么？
```c
/*
 * Trappable helper function for calculating length/1.
 *
 * When calling this function, entries in args[] should be set up as
 * follows:
 *
 *   args[0] = List to calculate length for.
 *   args[1] = Length accumulator (tagged integer).
 *
 * If the return value is a tagged integer, the length was calculated
 * successfully.
 *
 * Otherwise, if return value is THE_NON_VALUE and p->freason is TRAP,
 * the available reductions were exceeded and this function must be called
 * again after rescheduling. args[0] and args[1] have been updated to
 * contain the next part of the list and length so far, respectively.
 *
 * Otherwise, if return value is THE_NON_VALUE, the list did not end
 * in an empty list (and p->freason is BADARG).
 */
/* 注释已经写得十分清楚 */
Eterm erts_trapping_length_1(Process* p, Eterm* args)
{
    Eterm list;
    Uint i;
    Uint max_iter;
    Uint saved_max_iter;

#if defined(DEBUG) || defined(VALGRIND) /* 如果为debug模式，最大迭代50次*/
    max_iter = 50;	
#else
    max_iter = ERTS_BIF_REDS_LEFT(p) * 16; /* 否则最大迭代为剩余reductions * 16 */
#endif
    saved_max_iter = max_iter;
    ASSERT(max_iter > 0);

    list = args[0];		/* 需要计算长度的变量 */
    i = unsigned_val(args[1]);	/* 上次计算的结果 */
    while (is_list(list) && max_iter != 0) {	/* 进入循环 */
	list = CDR(list_val(list));	/* 去掉列表第一个值，返回剩余内容 */
	i++, max_iter--;	/* 结果加1， 迭代加1 */
    }

    if (is_list(list)) {	/* 如果list还是列表的情况下，说明还没有计算完成，那么记录当前执行的结果，等待下一次调度 */
        /*
         * We have exceeded the allotted number of iterations.
         * Save the result so far and signal a trap.
         */
        args[0] = list;	/* 保存剩余的list */
        args[1] = make_small(i);	/* 保存当前的结果 */
        p->freason = TRAP; 	/* 标记为trap
        BUMP_ALL_REDS(p);
        return THE_NON_VALUE;
    } else if (is_not_nil(list))  {
        /* Error. Should be NIL. */
	BIF_ERROR(p, BADARG);
    }

    /*
     * We reached the end of the list successfully. Bump reductions
     * and return result.
     */
    BUMP_REDS(p, (saved_max_iter - max_iter) / 16);	/* 如果计算完成，计算这次消耗的reductions */
    BIF_RET(make_small(i));	/* 返回结果 */
}
```

## 总结
length的实现就是遍历整个list，时间复杂度为：O(n)，n为列表元素数量，所以使用的时候注意列表数量；
另外从源码中可以看出length函数的实现对进程reductions的消耗，从而影响进程的调度。

## 引申思考
length的实现为什么要遍历整个list？因为erlang的列表实现是基于链表。
