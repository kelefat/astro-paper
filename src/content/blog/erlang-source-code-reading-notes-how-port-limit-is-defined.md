---
author: Kelefat
pubDatetime: 2025-03-26
modDatetime: 2025-03-26
title: Erlang 源码阅读笔记：端口数（port_limit）设置
tags:
  - Erlang
  - Erlang源码
description:
  Erlang 源码阅读笔记：端口数（port_limit）设置
---


本文基于Erlang OTP 27

Erlang中有一个端口数限制，当超出端口限制，部分服务将不能正常使用。

如创建网络进程、打开文件等等，常见的错误是返回`system_limit`。

现在我们来看看如何设置port_limit，并通过阅读源码看下其设置过程。

启动erlang，看看port_limit默认值是多少？
```bash
$erl
1> erlang:system_info(port_limit).
65536
2> 
```

通过`erlang:system_info(port_limit).`，我们得知port_limit的默认值为65536。

接下来尝试设置port_limit的值，使用`+Q`参数设置，输入命令`erl +Q 1`
```bash
$erl +Q 1
bad number of ports 1
Usage: beam.smp [flags] [ -- [init_args] ]
The flags are:
...省略...
-Q number      set maximum number of ports on this node;
               valid range is [1024-134217727]
```
这里通过`+Q 1`，尝试将prot_limit设为1，但erlang提示`bad number of ports 1`，并且给出`Q`参数值范围`[1024-134217727]`，最小为1024，最大为134217727。

那么尝试将port_limit设为1025，输入命令`erl +Q 1025`
```bash
$erl +Q 1025
Erlang/OTP 27 [erts-15.2.3] [source] [64-bit] [smp:2:2] [ds:2:2:10] [async-threads:1]

Eshell V15.2.3 (press Ctrl+G to abort, type help(). for help)
1> 
```

正常进入控制台，查询一下是否生效。输入`erlang:system_info(port_limit).`
```bash
1> erlang:system_info(port_limit).
2048
2>
```

这里显示的prot_limit为2048，为什么不是1025？

接下来尝试设置port_limit为最大值134217727，输入命令`erl +Q 134217727`

```bash
$erl +Q 134217727
Could not write tty mode to domain socket in spawn_init: 32
Aborted
```

Erlang直接退出，又是为什么？

我们来看看Erlang源码，为什么会这样！

erl_start为erl启动入口函数，通过暴力查找`bad number of ports`关键词，也可以定位到。

`bad number of ports`就是刚才我们执行`erl +Q 1`时，返回的错误信息。

erl_start所在文件路径为：`/erts/emulator/beam/erl_init.c`

```c
void
erl_start(int argc, char **argv)
{
//...省略部分代码...
        case 'Q': /* set maximum number of ports */	// +Q进入分支
            arg = get_arg(argv[i]+2, argv[i+1], &i);
            if (sys_strcmp(arg, "legacy") == 0)
                legacy_port_tab = 1;
            else {
                errno = 0;
                port_tab_sz = strtol(arg, NULL, 10);	// 变量port_tab_sz保存+Q设置的参数值
                if (errno != 0
                    || port_tab_sz < ERTS_MIN_PORTS
                    || ERTS_MAX_PORTS < port_tab_sz) { // 这就是为什么输入+Q 1时，会提示错误的原因
                    erts_fprintf(stderr, "bad number of ports %s\n", arg); // 如果不在可用范围内，则提示错误
                    erts_usage();	// 打印参数使用说明
                }
                port_tab_sz_ignore_files = 1;
            }
            break;
//...省略部分代码...
// 进行初始化
    erl_init(ncpu,
             proc_tab_sz,
             legacy_proc_tab,
             port_tab_sz,	// 上面已经赋了值
             port_tab_sz_ignore_files,
             legacy_port_tab,
             sys_proc_outst_req_lim,
             time_correction,
             time_warp_mode,
             node_tab_delete_delay,
             db_spin_count);
```

接下来看下erl_init函数。

```c
static void
erl_init(int ncpu,
         int proc_tab_sz,
         int legacy_proc_tab,
         int port_tab_sz,
         int port_tab_sz_ignore_files,
         int legacy_port_tab,
         Uint sys_proc_outst_req_lim,
         int time_correction,
         ErtsTimeWarpMode time_warp_mode,
         int node_tab_delete_delay,
         ErtsDbSpinCount db_spin_count)
{
    init_constant_literals();
    erts_monitor_link_init();
    erts_bif_unique_init();
    erts_proc_sig_queue_init(); /* Must be after erts_bif_unique_init(); */
    erts_init_time(time_correction, time_warp_mode);
    erts_init_sys_common_misc();
    erts_init_process(ncpu, proc_tab_sz, legacy_proc_tab);
    erts_init_scheduling(no_schedulers,
                         no_schedulers_online,
                         erts_no_poll_threads,
                         no_dirty_cpu_schedulers,
                         no_dirty_cpu_schedulers_online,
                         no_dirty_io_schedulers
                         );
//...省略部分代码...
//调用erts_init_io函数，将port_tab_sz传入
    erts_init_io(port_tab_sz, port_tab_sz_ignore_files, legacy_port_tab);
    init_load();
```

再看看erts_init_io函数。
所在文件路径为`erts/emulator/beam/io.c`
```c
void erts_init_io(int port_tab_size,
                  int port_tab_size_ignore_files,
                  int legacy_port_tab)
{
//...省略部分代码...
    if (!port_tab_size_ignore_files) {
        int max_files = sys_max_files();
        if (port_tab_size < max_files)
            port_tab_size = max_files;
    }

    if (port_tab_size > ERTS_MAX_PORTS)	// 这里做了容错处理，防止prot_tab_size过大或过小
        port_tab_size = ERTS_MAX_PORTS;
    else if (port_tab_size < ERTS_MIN_PORTS)
        port_tab_size = ERTS_MIN_PORTS;

    erts_rwmtx_init_opt(&erts_driver_list_lock, &drv_list_rwmtx_opts, "driver_list", NIL,
        ERTS_LOCK_FLAGS_PROPERTY_STATIC | ERTS_LOCK_FLAGS_CATEGORY_IO);
    driver_list = NULL;
    erts_tsd_key_create(&driver_list_lock_status_key,
                            "erts_driver_list_lock_status_key");
    erts_tsd_key_create(&driver_list_last_error_key,
                            "erts_driver_list_last_error_key");

//调用erts_ptab_init_table函数	
    erts_ptab_init_table(&erts_port,
                         ERTS_ALC_T_PORT_TABLE,
                         NULL,
                         (ErtsPTabElementCommon *) &erts_invalid_port.common,
                         port_tab_size,
                         common_element_size, /* Doesn't need to be exact */
                         "port_table",
                         legacy_port_tab,
                         1);
```

继续看erts_ptab_init_table函数。
所在文件路径为`erts/emulator/beam/erl_ptab.c`

```c
void
erts_ptab_init_table(ErtsPTab *ptab,
                     ErtsAlcType_t atype,
                     void (*release_element)(void *),
                     ErtsPTabElementCommon *invalid_element,
                     int size,
                     UWord element_size,
                     char *name,
                     int legacy,
                     int atomic_refc)
{
    size_t tab_sz, alloc_sz;
    Uint32 bits, cl, cli, ix, ix_per_cache_line, tab_cache_lines;
    char *tab_end;
    erts_atomic_t *tab_entry;
    erts_rwmtx_opt_t rwmtx_opts = ERTS_RWMTX_OPT_DEFAULT_INITER;
    rwmtx_opts.type = ERTS_RWMTX_TYPE_EXTREMELY_FREQUENT_READ;
    rwmtx_opts.lived = ERTS_RWMTX_LONG_LIVED;

    erts_rwmtx_init_opt(&ptab->list.data.rwmtx, &rwmtx_opts, name, NIL,
        ERTS_LOCK_FLAGS_PROPERTY_STATIC | ERTS_LOCK_FLAGS_CATEGORY_GENERIC);
    erts_atomic32_init_nob(&ptab->vola.tile.count, 0);
    last_data_init_nob(ptab, ~((Uint64) 0));

    // 这里的size就是上面传入的proc_tab_size变量，即我们要设置的端口限制数
    /* A size that is a power of 2 is to prefer performance wise */
    bits = erts_fit_in_bits_int32(size-1);	// 英文注释写得很清楚“选择 2 的幂次方对于性能更有利”
    size = 1 << bits;				// 处理成2 的幂次方，因此当我们使用+Q 1025的时候，最终会变成2048。最接近 1025 的 2 的幂次方 是 2048。
    if (size > ERTS_PTAB_MAX_SIZE) {		// 这里的ERTS_PTAB_MAX_SIZE为134217728, 而ERTS_MAX_PORTS的定义为(ERTS_PTAB_MAX_SIZE-1)，即134217727
        size = ERTS_PTAB_MAX_SIZE;
        bits = erts_fit_in_bits_int32((Sint32) size - 1);
    }

    ptab->r.o.element_size = element_size;
    ptab->r.o.max = size;			// 最终生效值，而erlang:system_info(port_limit)返回的值正是这个ptab->r.o.max

    tab_sz = ERTS_ALC_CACHE_LINE_ALIGN_SIZE(size*sizeof(erts_atomic_t));	// 假设我们的size是134217727，那么tab_sz = 134217727 * 8 = 1073741816
    alloc_sz = tab_sz;								// alloc_sz = tab_sz = 1073741816
    if (!legacy)								// 是否兼容旧代码，这个默认值为0，所以alloc_sz由1073741816变成2147483632，新版本的优化？
        alloc_sz += ERTS_ALC_CACHE_LINE_ALIGN_SIZE(size*sizeof(erts_atomic_t));	// alloc_sz += 1073741816，即2147483632
    ptab->r.o.tab = erts_alloc_permanent_cache_aligned(atype, alloc_sz);	// 申请内存
    tab_end = ((char *) ptab->r.o.tab) + tab_sz;				// tab_end = 起始位置 + 1073741816
    tab_entry = ptab->r.o.tab;
    while (tab_end > ((char *) tab_entry)) {					//  循环tab_end次即循环1073741816次
        erts_atomic_init_nob(tab_entry, ERTS_AINT_NULL);			//  初始化，最终占内存粗略计算为1073741816 / 1024 / 1024 = 1023.99999, 约1G内存
        tab_entry++;								
    }
```

而刚才尝试`erl +Q 134217727`时， erlang直接退出了。原因就是测试机器只有1.6G可用内存，而这里已经占用了1G内存, 接下来因内存不足导致启动失败。

经测试如果将port_limit设为134217727，最终erlang进程约占1.5G内存。

通过阅读源码，我们了解到port_limit的设置过程。也解释了为什么设置1025时，实际生效值是2048，因为“选择 2 的幂次方对于性能更有利”，至于为什么有利，留给大家思考了。

另外就是port_limit越大，占用内存也越多。
