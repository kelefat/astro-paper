---
author: Kelefat
pubDatetime: 2025-03-26
modDatetime: 2025-03-26
title: Erlang 源码阅读笔记：端口数（port_limit）是如何作用的
tags:
  - Erlang
  - Erlang源码
description:
  Erlang 源码阅读笔记：端口数（port_limit）是如何作用的
---


本文基于Erlang OTP 27

输入命令`erl +Q 1024 -name server@127.0.0.1 -setcookie mycookie`，进入Erlang控制台，并设置port_limit为1024。
```bash
$erl +Q 1024 -name server@127.0.0.1 -setcookie mycookie
Erlang/OTP 27 [erts-15.2.3] [source] [64-bit] [smp:2:2] [ds:2:2:10] [async-threads:1]

Eshell V15.2.3 (press Ctrl+G to abort, type help(). for help)
(server@127.0.0.1)1> 
```

输入`erlang:system_info(port_limit).`，查看当前端口限制数

```bash
(server@127.0.0.1)1> erlang:system_info(port_limit).
1024
(server@127.0.0.1)2> 
```

输入`erlang:system_info(port_count).`，查看当前占用端口数
```bash
(server@127.0.0.1)2> erlang:system_info(port_count).
4
(server@127.0.0.1)3>
```
当前port_limit为1024，port_count为4

现在尝试创建一个tcp连接，输入`gen_tcp:connect({127,0,0,1}, 4369, [{active, false}]).`
```bash
(server@127.0.0.1)3> gen_tcp:connect({127,0,0,1}, 4369, [{active, false}]).
{ok,#Port<0.6>}
(server@127.0.0.1)4> erlang:system_info(port_count).
5
```

创建连接成功后，再一次查询port_count，由4变成5了，说明建立tcp连接占用了一个port。

现在尝试将端口数耗尽，spawn一个进程，创建1024-5=1019个进程，输入如下代码：
```erlang
spawn(fun() -> [begin gen_tcp:connect({127,0,0,1}, 4369, [{active, false}]), timer:sleep(100) end || _ <- lists:duplicate(1019, 'x')], receive Msg -> Msg end end).
```
```bash
(server@127.0.0.1)5> spawn(fun() -> [begin gen_tcp:connect({127,0,0,1}, 4369, [{active, false}]), timer:sleep(100) end || _ <- lists:duplicate(1019, 'x')], receive Msg -> Msg end end).
<0.97.0>
(server@127.0.0.1)6> erlang:system_info(port_count).
729
(server@127.0.0.1)7> erlang:system_info(port_count).
853
(server@127.0.0.1)8> erlang:system_info(port_count).
1024
(server@127.0.0.1)9> erlang:system_info(port_count).
1024
```

从控制台信息中，可以看出，端口一直被占用，最终达到1024。

现在尝试再建立一个tcp连接，看看会如何？

输入`gen_tcp:connect({127,0,0,1}, 4369, [{active, false}]).`
```bash
(server@127.0.0.1)10> gen_tcp:connect({127,0,0,1}, 4369, [{active, false}]).
{error,system_limit}
(server@127.0.0.1)11> 
```

提示`{error,system_limit}`，说明已经无法再创建新的连接。

假设现在是生产环境，端口数已经占满，这个时候还能否从远程连接这个节点呢？我们试一下。

新开一个shell，输入命令`erl -name client@127.0.0.1 -setcookie mycookie -remsh server@127.0.0.1`，连接刚才创建的erlang节点。

```bash
$erl -name client@127.0.0.1 -setcookie mycookie -remsh server@127.0.0.1
Could not connect to "server@127.0.0.1"
```

连接不上，提示`Could not connect to "server@127.0.0.1"`

回到刚才的erlang控制台，尝试关闭一个端口，再看看能不能远程连接上。

输入`gen_tcp:close(#Port<0.6>).`，`#Port<0.6>`这个port是我们第一次创建tcp连接时返回的。
```bash
(server@127.0.0.1)12> gen_tcp:close(#Port<0.6>).
ok
(server@127.0.0.1)13> erlang:system_info(port_count).
1023
```

再查询，当前的port_count为1023，说明关闭端口成功。

再次尝试远程连接此节点。

```bash
$erl -name client@127.0.0.1 -setcookie mycookie -remsh server@127.0.0.1
Erlang/OTP 27 [erts-15.2.3] [source] [64-bit] [smp:2:2] [ds:2:2:10] [async-threads:1]

Eshell V15.2.3 (press Ctrl+G to abort, type help(). for help)
(server@127.0.0.1)1> erlang:system_info(port_count).
1024
```
这次成功连接上。

再次查询port_count为1024。说明远程连接节点同样会占用端口。

到这里我们看到了port_limit带来的影响，因此port_limit要适当设置。

接下来，我们通过阅读源码，了解一下port_limit是如何作用的。

以gen_tcp:connect函数为入口，一步一步分析。

`gen_tcp:connect -> inet_tcp:do_connect -> inet:open -> inet:open_opts -> prim_net:open`

从`gen_tcp:connect`开始调用，最终来到`prim_net:open`。

具体看下`prim_net:open`的实现。
```erlang
open(Protocol, Family, Type, Opts, Req, Data) ->
    Drv = protocol2drv(Protocol),
    AF = enc_family(Family),
    T = enc_type(Type),
    try erlang:open_port({spawn_driver,Drv}, [binary]) of %% 这里调用了open_port
        S ->
            case setopts(S, Opts) of
                ok ->
                    case ctl_cmd(S, Req, [AF,T,Data]) of
                        {ok,_} -> {ok,S};
                        {error,_}=E1 ->
                            close(S),
                            E1
                    end;
                {error,_}=E2 ->
                    close(S),
                    E2
            end
    catch
        %% The only (?) way to get here is to try to open
        %% the sctp driver when it does not exist (badarg)
        error:badarg       -> {error, eprotonosupport};
        %% system_limit if out of port slots
        error:system_limit -> {error, system_limit}	%% error为system_limit就返回{error, system_limit}
    end.
```

erlang:open_port对应的函数名为`erts_internal_open_port_2`

所在源码文件`erts/emulator/beam/erl_bif_port.c`

```c
BIF_RETTYPE erts_internal_open_port_2(BIF_ALIST_2)
{
    BIF_RETTYPE ret;
    Port *port;
    Eterm res;
    char *str;
    int err_type, err_num;
    ErtsLink *proc_lnk, *port_lnk;

// 调用open_port
    port = open_port(BIF_P, BIF_ARG_1, BIF_ARG_2, &err_type, &err_num);
    if (!port) {
        if (err_type == -4) {
            /* Invalid settings arguments. */
            return am_badopt;
        } else if (err_type == -3) {
            ASSERT(err_num == BADARG || err_num == SYSTEM_LIMIT);
            if (err_num == BADARG)
                res = am_badarg;
            else if (err_num == SYSTEM_LIMIT)
                res = am_system_limit;	// 返回给erl的system_limit错误信息
            else
                /* this is only here to silence gcc, it should not happen */
                BIF_ERROR(BIF_P, EXC_INTERNAL_ERROR);
        } else if (err_type == -2) {
            str = erl_errno_id(err_num);
            res = erts_atom_put((byte *) str, sys_strlen(str), ERTS_ATOM_ENC_LATIN1, 1);
        } else {
            res = am_einval;
        }
        BIF_RET(res);
    }
```

看看open_port的实现

```c
static Port *
open_port(Process* p, Eterm name, Eterm settings, int *err_typep, int *err_nump)
{
//...省略部分代码...

// 调用erts_open_driver
    port = erts_open_driver(driver, p->common.id, name_buf, &opts, err_typep, err_nump);
#ifdef USE_VM_PROBES
    if (port && DTRACE_ENABLED(port_open)) {
        DTRACE_CHARBUF(process_str, DTRACE_TERM_BUF_SIZE);
        DTRACE_CHARBUF(port_str, DTRACE_TERM_BUF_SIZE);

        dtrace_proc_str(p, process_str);
        erts_snprintf(port_str, sizeof(DTRACE_CHARBUF_NAME(port_str)), "%T", port->common.id);
        DTRACE3(port_open, process_str, name_buf, port_str);
    }
#endif

    if (port && ERTS_IS_P_TRACED_FL(port, F_TRACE_PORTS))
        trace_port(port, am_getting_linked, p->common.id, F_TRACE_PORTS);

    erts_proc_lock(p, ERTS_PROC_LOCK_MAIN);

    if (ERTS_IS_P_TRACED_FL(p, F_TRACE_SCHED_PROCS)) {
        trace_sched(p, ERTS_PROC_LOCK_MAIN, am_in, F_TRACE_SCHED_PROCS);
    }

// 如果申请port失败，就执行do_return
    if (!port) {
        DEBUGF(("open_driver returned (%d:%d)\n",
                err_typep ? *err_typep : 4711,
                err_nump ? *err_nump : 4711));
        goto do_return;
    }

    if (linebuf && port->linebuf == NULL){
        port->linebuf = allocate_linebuf(linebuf);
        sflgs |= ERTS_PORT_SFLG_LINEBUF_IO;
    }

    if (sflgs)
        erts_atomic32_read_bor_relb(&port->state, sflgs);

 do_return:
    erts_osenv_clear(&opts.envir);
    if (name_buf)
        erts_free(ERTS_ALC_T_TMP, (void *) name_buf);
    if (opts.argv) {
        free_args(opts.argv);
    }
    if (opts.wd && opts.wd != ((char *)dir)) {
        erts_free(ERTS_ALC_T_TMP, (void *) opts.wd);
    }
    return port;
//...省略部分代码...
}
```

看看erts_open_driver的实现，所在源码文件`erts/emulator/beam/io.c`
```c
/*
   Opens a driver.
   Returns the non-negative port number, if successful.
   If there is an error, -1 or -2 or -3 is returned. -2 means that
   there is valid error information in *error_number_ptr.
   Returning -3 means that an error in the given options was detected
   (*error_number_ptr must contain either BADARG or SYSTEM_LIMIT).
   The driver start function must obey the same conventions.
*/
Port *
erts_open_driver(erts_driver_t* driver, /* Pointer to driver. */
                 Eterm pid,             /* Current process. */
                 char* name,            /* Driver name. */
                 SysDriverOpts* opts,   /* Options. */
                 int *error_type_ptr,   /* error type */
                 int *error_number_ptr) /* errno in case of error type -2 */
{

#undef ERTS_OPEN_DRIVER_RET
#define ERTS_OPEN_DRIVER_RET(Prt, EType, ENo)   \
    do {                                        \
        if (error_type_ptr)                     \
            *error_type_ptr = (EType);          \
        if (error_number_ptr)                   \
            *error_number_ptr = (ENo);          \
        return (Prt);                           \
    } while (0)

    ErlDrvData drv_data = 0;
    Port *port;
    int error_type, error_number;
    int port_errno = 0;
    erts_mtx_t *driver_lock = NULL;
    int cprt_flgs = 0;

//...省略部分代码...
//调用create_port创建端口
    port = create_port(name, driver, driver_lock, cprt_flgs, pid, &port_errno);
    if (!port) {//如果port为false，则返回错误
        if (driver->handle) {
            erts_rwmtx_rlock(&erts_driver_list_lock);
            erts_ddll_decrement_port_count(driver->handle);
            erts_rwmtx_runlock(&erts_driver_list_lock);
            erts_ddll_dereference_driver(driver->handle);
        }
        if (port_errno)
            ERTS_OPEN_DRIVER_RET(NULL, -2, port_errno); // 这里返回具体的错误码
        else
            ERTS_OPEN_DRIVER_RET(NULL, -3, SYSTEM_LIMIT);// 没有错误码的情况，返回SYSTEM_LIMIT错误
    }
```

看看create_port的实现

```c
static Port *create_port(char *name,
                         erts_driver_t *driver,
                         erts_mtx_t *driver_lock,
                         int create_flags,
                         Eterm pid,
                         int *enop)
{
    ErtsPortTaskBusyPortQ *busy_port_queue;
    Port *prt;
    char *p;
    size_t port_size, busy_port_queue_size, size;
    erts_aint32_t state = ERTS_PORT_SFLG_CONNECTED;
    erts_aint32_t x_pts_flgs = 0;
//...省略部分代码...
//  如果erts_ptab_new_element返回0，则return
    if (!erts_ptab_new_element(&erts_port,
                               &prt->common,
                               (void *) prt,
                               insert_port_struct)) {

#if !ERTS_PORT_INIT_INSTR_NEED_ID
        port_init_instr_abort(prt);
#endif
        if (driver_lock)
            erts_mtx_unlock(driver_lock);
        if (enop)
            *enop = 0;
        erts_free(ERTS_ALC_T_PORT, prt);
        return NULL;
    }
...
```

重点看看erts_ptab_new_element的实现，所在源码文件`erts/emulator/beam/erl_ptab.c`
```c
int
erts_ptab_new_element(ErtsPTab *ptab,
                      ErtsPTabElementCommon *ptab_el,
                      void *init_arg,
                      void (*init_ptab_el)(void *, Eterm))
{
    Uint32 pix, ix;
    Uint data;
    erts_aint32_t count;
    erts_aint_t invalid = (erts_aint_t) ptab->r.o.invalid_element;

    erts_ptab_rlock(ptab);

    count = erts_atomic32_inc_read_acqb(&ptab->vola.tile.count);	// 端口计数+1
    if (count > ptab->r.o.max) {					// 超出port_limit限制，我们测试的例子是1025 > 1024
        while (1) {
            erts_aint32_t act_count;

            act_count = erts_atomic32_cmpxchg_relb(&ptab->vola.tile.count,	// 回滚端口计数
                                                       count-1,
                                                       count);
            if (act_count == count) {	// 回滚成功，返回0
                erts_ptab_runlock(ptab);
                return 0;
            }
            count = act_count;
            if (count <= ptab->r.o.max)
                break;
        }
    }
```

到这里，我们已经了解port_limit如何发挥作用。

同时我们也看到port_limit带来的影响，如果端口数给占满，会导致无法远程登录，所以在生产环境上要注意。

