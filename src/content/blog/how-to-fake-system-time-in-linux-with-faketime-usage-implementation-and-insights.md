---
author: Kelefat
pubDatetime: 2025-04-27
modDatetime: 2025-04-27
title: 如何优雅地调整时间？faketime的使用和原理分析
tags:
  - faketime
  - libfaketime
description:
  本文介绍在Linux系统中，通过faketime工具优雅地伪造程序时间，详细说明安装、使用方法及原理，重点解析其基于LD_PRELOAD的拦截机制。
---

## 问题
游戏服务器需要经常调整时间，测试各种活动和功能，那么如何优雅地调整时间呢？

## 方案

|方案|优点|缺点|
|-|-|-|
|调整主机时间|简单直接|当前主机所有程序都受到影响|
|使用虚拟机部署，调整虚拟机时间|隔离管理|部署麻烦|
|Faketime|针对程序自身|只支持Linux and macOS|


这里主要介绍Faketime方案， <a href="https://github.com/wolfcw/libfaketime" target="_blank">项目地址</a>

下载源码：
https://github.com/wolfcw/libfaketime/archive/refs/tags/v0.9.10.tar.gz

执行以下命令：
```bash
tar zxvf libfaketime-0.9.10.tar.gz
cd libfaketime-0.9.10
make && make install
```

编译成功后，默认会安装在/usr/local/lib/faketime/

```bash
ll /usr/local/lib/faketime/
```
会看到libfaketime.so.1文件

先做一个简单的测试：
```bash
#当前时间增加一天
LD_PRELOAD=/usr/local/lib/faketime/libfaketime.so.1 FAKETIME="+1d" date
```

```bash
#当前时间减少一天
LD_PRELOAD=/usr/local/lib/faketime/libfaketime.so.1 FAKETIME="-1d" date
```

那么如何改变运行时的时间呢？
```bash
#创建一个文件，faketime将通过监控文件内容来调整时间 
touch faketime.rc
LD_PRELOAD=/usr/local/lib/faketime/libfaketime.so.1  FAKETIME_TIMESTAMP_FILE=faketime.rc  /bin/bash -c 'while true ; do date ; sleep 1 ; done'
```
```bash
#当前时间增加3600秒
echo "+3600" > faketime.rc
```
```bash
#当前时间减少3600秒
echo "-3600" > faketime.rc
```
通过改变faketime.rc的内容，来实现运行时修改时间。

## 原理分析
要了解faktetime的实现机制，就要先了解LD_PRELOAD，这也是为什么faketime只能在linux下运行的原因。

LD_PRELOAD 是一个在 Linux 和其他类 Unix 系统中广泛使用的环境变量，它允许你在程序启动时优先加载指定的动态链接库（.so 文件）。这使得你可以在不修改程序本身的情况下，拦截、替换或扩展程序的行为。

当一个程序启动时，动态链接器会按照一定的顺序搜索并加载程序依赖的动态链接库。通常情况下，系统默认的动态链接库会被优先加载。但是，如果设置了 LD_PRELOAD 环境变量，那么动态链接器会首先加载 LD_PRELOAD 中指定的动态链接库。

faketime就是通过LD_PRELOAD实现拦截。

以date命令为例，我们追踪一下date的运行。在shell下输入：

```bash
ltrace date
```

```bash
__libc_start_main(0x401ac0, 1, 0x7ffcecd9c2b8, 0x4096f0 <unfinished ...>
strrchr("date", '/')                                                                                      = nil
setlocale(LC_ALL, "")                                                                                     = "en_US.UTF-8"
bindtextdomain("coreutils", "/usr/share/locale")                                                          = "/usr/share/locale"
textdomain("coreutils")                                                                                   = "coreutils"
__cxa_atexit(0x402c50, 0, 0, 0x736c6974756572)                                                            = 0
getopt_long(1, 0x7ffcecd9c2b8, "d:f:I::r:Rs:u", 0x60d2a0, nil)                                            = -1
nl_langinfo(0x2006c, 1, 0, 0)                                                                             = 0x7f9bcc8ff955
clock_gettime(0, 0x7ffcecd9c0f0, 0x2474440, 0)                                                            = 0
localtime(0x7ffcecd9c070)                                                                                 = 0x7f9bd20a7d40
strftime(" Tue", 1024, " %a", 0x7f9bd20a7d40)                                                             = 4
fwrite("Tue", 3, 1, 0x7f9bd20a3400)                                                                       = 1
fputc(' ', 0x7f9bd20a3400)                                                                                = 32
strftime(" Jan", 1024, " %b", 0x7f9bd20a7d40)                                                             = 4
fwrite("Jan", 3, 1, 0x7f9bd20a3400)                                                                       = 1
fputc(' ', 0x7f9bd20a3400)                                                                                = 32
fwrite("14", 2, 1, 0x7f9bd20a3400)                                                                        = 1
fputc(' ', 0x7f9bd20a3400)                                                                                = 32
fwrite("15", 2, 1, 0x7f9bd20a3400)                                                                        = 1
fputc(':', 0x7f9bd20a3400)                                                                                = 58
fwrite("46", 2, 1, 0x7f9bd20a3400)                                                                        = 1
fputc(':', 0x7f9bd20a3400)                                                                                = 58
fputc('0', 0x7f9bd20a3400)                                                                                = 48
fwrite("8", 1, 1, 0x7f9bd20a3400)                                                                         = 1
fputc(' ', 0x7f9bd20a3400)                                                                                = 32
strlen("CST")                                                                                             = 3
fwrite("CST", 3, 1, 0x7f9bd20a3400)                                                                       = 1
fputc(' ', 0x7f9bd20a3400)                                                                                = 32
fwrite("2025", 4, 1, 0x7f9bd20a3400)                                                                      = 1
__overflow(0x7f9bd20a3400, 10, 4, 0x35323032Tue Jan 14 15:46:08 CST 2025
)                                                             = 10
exit(0 <unfinished ...>
__fpending(0x7f9bd20a3400, 0, 64, 0x7f9bd20a3eb0)                                                         = 0
fileno(0x7f9bd20a3400)                                                                                    = 1
__freading(0x7f9bd20a3400, 0, 64, 0x7f9bd20a3eb0)                                                         = 0
__freading(0x7f9bd20a3400, 0, 2052, 0x7f9bd20a3eb0)                                                       = 0
fflush(0x7f9bd20a3400)                                                                                    = 0
fclose(0x7f9bd20a3400)                                                                                    = 0
__fpending(0x7f9bd20a31c0, 0, 3328, 0xfbad000c)                                                           = 0
fileno(0x7f9bd20a31c0)                                                                                    = 2
__freading(0x7f9bd20a31c0, 0, 3328, 0xfbad000c)                                                           = 0
__freading(0x7f9bd20a31c0, 0, 4, 0xfbad000c)                                                              = 0
fflush(0x7f9bd20a31c0)                                                                                    = 0
fclose(0x7f9bd20a31c0)                                                                                    = 0
+++ exited (status 0) +++
```

从结果中可以看到date调用了clock_gettime函数，那么我们尝试拦截clock_gettime。

要拦截clock_gettime的实现，我们得知道clock_gettime函数的定义。

clock_gettime为linux的系统调用，我们可以通过man来查看其具体定义。

```bash
man clock_gettime
```

```bash
CLOCK_GETRES(2)                                                        Linux Programmer's Manual                                                       CLOCK_GETRES(2)

NAME
       clock_getres, clock_gettime, clock_settime - clock and time functions

SYNOPSIS
       #include <time.h>

       int clock_getres(clockid_t clk_id, struct timespec *res);

       int clock_gettime(clockid_t clk_id, struct timespec *tp);

       int clock_settime(clockid_t clk_id, const struct timespec *tp);

       Link with -lrt (only for glibc versions before 2.17).

   Feature Test Macro Requirements for glibc (see feature_test_macros(7)):

       clock_getres(), clock_gettime(), clock_settime():
              _POSIX_C_SOURCE >= 199309L

DESCRIPTION
       The  function  clock_getres() finds the resolution (precision) of the specified clock clk_id, and, if res is non-NULL, stores it in the struct timespec pointed
       to by res.  The resolution of clocks depends on the implementation and cannot be configured by a particular process.  If the time value pointed to by the argu‐
       ment tp of clock_settime() is not a multiple of res, then it is truncated to a multiple of res.

       The functions clock_gettime() and clock_settime() retrieve and set the time of the specified clock clk_id.

       The res and tp arguments are timespec structures, as specified in <time.h>:

           struct timespec {
               time_t   tv_sec;        /* seconds */
               long     tv_nsec;       /* nanoseconds */
           };
```

从man的返回结果中，我们可以得知clock_gettime的定义：

<code>int clock_gettime(clockid_t clk_id, struct timespec *tp);</code>

假设现在希望输入date返回指定时间，如2025-10-24 12:00:00，那么我们可以这样实现：

这里使用c实现，先新建文件，如/tmp/fake_clockgettime.c， 输入如下内容：
```c
#include <time.h>

int clock_gettime(clockid_t clk_id, struct timespec *tp)
{
  /** 2025-10-24 12:00:00的时间戳是1761278400 **/
  /** 从man手册可知， tv_sec存放秒数 **/
  tp->tv_sec  = 1761278400;
  return 0;
}
```

编译生成so，输入如何命令：
```bash
gcc -fPIC -shared fake_clockgettime.c  -o fake_clockgettime.so
```

然后使用LD_PRELOAD加载我们刚才编译生成的so文件：
```bash
LD_PRELOAD=/tmp/fake_clockgettime.so date "+%Y-%m-%d %H:%M:%S" 
```

输出
```bash
2025-10-24 12:00:00
```

我们分别看下date的动态链接库加载情况，输入如下命令：
```bash
ldd /bin/date
```

```bash
linux-vdso.so.1 =>  (0x00007ffebd725000)
libc.so.6 => /lib64/libc.so.6 (0x00007f967ed48000)
/lib64/ld-linux-x86-64.so.2 (0x00007f967f116000)
```

使用LD_PRELOAD后的加载情况，输入如下命令：
```bash
export LD_PRELOAD=/tmp/fake_clockgettime.so 
ldd /bin/date
```

```bash
linux-vdso.so.1 =>  (0x00007ffcbb9fc000)
/tmp/fake_clockgettime.so (0x00007f0b03111000)
libc.so.6 => /lib64/libc.so.6 (0x00007f0b02d43000)
/lib64/ld-linux-x86-64.so.2 (0x00007f0b03313000)
```
通过以上对比，我们可以看出date加载了/tmp/fake_clockgettime.so。

以上通过拦截clock_gettime，实现修改时间。那么接下来，试下拦截随机函数。

新建test_rand.c，作为例子调用系统随机函数。
```c
#include <stdio.h>
#include <stdlib.h>
#include <time.h>

int main() {
  srand(time(NULL));
  int random_number = rand();
  printf("rand number: %d\n", random_number);
  return 0;
}
```

编译生成程序，输入如下命令：
```bash
gcc test_rand.c -o test_rand
```

运行test_rand，试下获取随机数，输入如下命令：
```bash
./test_rand
```
输出
```
rand number: 737025517
```


现在尝试拦截rand()，使随机产生的值都为1024

新建fake_rand.c，输入以下内容：
```c
int rand()
{
  return 1024;
}
```

编译生成so，输入如下命令：
```bash
gcc -fPIC -shared fake_rand.c  -o fake_rand.so
```

使用LD_PRELOAD加载我们刚才编译生成的so文件：
```bash
LD_PRELOAD=/tmp/fake_rand.so ./test_rand
```
输出
```bash
rand number: 1024
```

通过以上两个例子，我们了解了LD_PRELOAD的强大。faketime就是使用LD_PRELOAD的一个很好例子，我们可以发挥更多的想象，尝试使用LD_PRELOAD实现更多功能。
