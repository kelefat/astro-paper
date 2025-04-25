---
author: Kelefat
pubDatetime: 2025-03-24T22:40:00Z
modDatetime: 2025-03-24T22:40:00Z
title: 如何使用 GDB 调试 Erlang
tags:
  - Erlang
  - GDB
  - Erlang debug
  - etp-commands 
description:
  如何使用 GDB 调试 Erlang：从编译到 etp-commands
---

在阅读Erlang源码的过程中，如果能直接断点调试，将有助于进一步了解Erlang的运行机制。

接下来以Erlang otp 27为例，我们看下如何对Erlang进行断点调试。

按以下步骤，正常编译Erlang，这里指定安装目录为`/usr/local/erlang_27_debug`
```shell
$cd /usr/local
$wget https://github.com/erlang/otp/releases/download/OTP-27.3/otp_src_27.3.tar.gz
$tar zxvf otp_src_27.3.tar.gz
$cd otp_src_27.3
$export ERL_TOP=`pwd`
$./configure --prefix=/usr/local/erlang_27_debug/ --without-javac
$make && make install
```


安装完成后，尝试运行erl。
```bash
$/usr/local/erlang_27_debug/bin/erl
```

出现以下控制台信息，说明安装成功。
```
Erlang/OTP 27 [erts-15.2.3] [source] [64-bit] [smp:2:2] [ds:2:2:10] [async-threads:1]

Eshell V15.2.3 (press Ctrl+G to abort, type help(). for help)
1>
```

接下来，使用debug模式编译
```shell
$cd $ERL_TOP/erts/emulator && make debug
```

编译成功后，执行以下命令。
```shell
$$ERL_TOP/bin/cerl -debug  -gdb -break main
```

第一次运行时，提示是否安装gdb-tools，输入Y

```shell
Do you want to install gdb-tools? (Y/n) Y
Downloading gdb-tools for Erlang/OTP 27
git clone --origin gdb_tools https://github.com/erlang/otp-gdb-tools /usr/local/otp_src_27.3/erts/etc/unix/gdb-tools
 MAKE   opt
gmake[1]: Entering directory `/usr/local/otp_src_27.3/erts/etc/common'
 CC     /usr/local/otp_src_27.3/erts/obj/x86_64-pc-linux-gnu/jit-reader.o
 LD     /usr/local/otp_src_27.3/bin/x86_64-pc-linux-gnu/jit-reader.so
gmake[1]: Leaving directory `/usr/local/otp_src_27.3/erts/etc/common'
Excess command line arguments ignored. (main") ...)
GNU gdb (GDB) Red Hat Enterprise Linux 7.6.1-120.el7
Copyright (C) 2013 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-redhat-linux-gnu".
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>...
(insert-string: No such file or directory.
/usr/local/otp_src_27.3/bin/"break: No such file or directory.
%---------------------------------------------------------------------------
% Use etp-help for a command overview and general help.
%
% To use the Erlang support module, the environment variable ROOTDIR
% must be set to the toplevel installation directory of Erlang/OTP,
% so the etp-commands file becomes:
%     $ROOTDIR/erts/etc/unix/etp-commands
% Also, erl and erlc must be in the path.
%---------------------------------------------------------------------------
etp-set-max-depth 20
etp-set-max-string-length 100
--------------- System Information ---------------
OTP release: 27
ERTS version: 15.2.3
Arch: x86_64-pc-linux-gnu
Endianness: Little
Word size: 64-bit
BeamAsm support: no
SMP support: yes
Thread support: yes
Kernel poll: Supported
Debug compiled: yes
Lock checking: yes
Lock counting: no
System not initialized
--------------------------------------------------
```

尝试断点，以常见的length函数为例，在控制台输入命令`break length_1`。
```bash
(gdb) break length_1
Breakpoint 1 at 0x5270f4: file beam/erl_bif_guard.c, line 218.
```

执行成功，按`r`运行，进入erlang控制台
```bash
(gdb) r
Starting program: /usr/local/otp_src_27.3/bin/x86_64-pc-linux-gnu/beam.debug.smp -- -root /usr/local/otp_src_27.3 -bindir /usr/local/otp_src_27.3/bin/x86_64-pc-linux-gnu -progname /usr/local/otp_src_27.3/bin/cerl -debug -- -home /root -- -emu_type debug -- 
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib64/libthread_db.so.1".
[New Thread 0x7fffb699f700 (LWP 13211)]
[New Thread 0x7fffa9abc700 (LWP 13212)]
[New Thread 0x7ffff7e33700 (LWP 13213)]
[Detaching after fork from child process 13214]
[New Thread 0x7fffb411b700 (LWP 13215)]
[New Thread 0x7fffb4098700 (LWP 13216)]
[New Thread 0x7fffa92bb700 (LWP 13217)]
[New Thread 0x7fffa9290700 (LWP 13218)]
[New Thread 0x7fffa9265700 (LWP 13219)]
[New Thread 0x7fffa923a700 (LWP 13220)]
[New Thread 0x7fffa920f700 (LWP 13221)]
[New Thread 0x7fffa91e4700 (LWP 13222)]
[New Thread 0x7fffa91b9700 (LWP 13223)]
[New Thread 0x7fffa918e700 (LWP 13224)]
[New Thread 0x7fffa9163700 (LWP 13225)]
[New Thread 0x7fffa9138700 (LWP 13226)]
[New Thread 0x7fffa910d700 (LWP 13227)]
[New Thread 0x7fffa90e2700 (LWP 13228)]
[New Thread 0x7fffa90b7700 (LWP 13229)]
[New Thread 0x7fffa908c700 (LWP 13230)]
Erlang/OTP 27 [erts-15.2.3] [source] [64-bit] [smp:2:2] [ds:2:2:10] [async-threads:1] [type-assertions] [debug-compiled] [lock-checking]

Eshell V15.2.3 (press Ctrl+G to abort, type help(). for help)
1>
```

调用length函数，`erlang:length([1,2,3]).`
```bash
1> erlang:length([1,2,3]).
[Switching to Thread 0x7fffb411b700 (LWP 13215)]

Breakpoint 1, length_1 (A__p=0xd19c98, BIF__ARGS=0x7ffff7e38240, A__I=0x7fffb546c948) at beam/erl_bif_guard.c:218
```

看到Breakpoint说明已经断点成功，接下来按`n`，单步执行，一步一步看erlang的执行。
```bash
218         args[0] = BIF_ARG_1;
(gdb) n
219         args[1] = make_small(0);
(gdb) n
220         args[2] = BIF_ARG_1;
```

从上面的信息，我们看到了args的赋值过程，我们看下args[0]具体是什么值？

输入命令`p args[0]`
```bash
(gdb) p args[0]
$4 = 140735133207249
```

显示值为：140735133207249，那么这个值代表什么意思呢？这个时候我们可以使用erlang提供的etp-commands。

输入命令`etp args[0]`
```bash
(gdb) etp args[0]
[1,2,3].
```

显示值为：[1,2,3]，通过etp这个命令，我们看到了一个可读值，也就是我们传入的`[1,2,3]`。

按`c`键，继续运行。

这个时候回到了erlang控制台，控制台返回了`erlang:length([1,2,3]).`的执行结果`3`。

```bash
(gdb) c
Continuing.

3
3> q().
ok
4> [Thread 0x7ffff7e33700 (LWP 13213) exited]
[Thread 0x7fffb4098700 (LWP 13216) exited]
[Thread 0x7fffa92bb700 (LWP 13217) exited]
[Thread 0x7fffa9290700 (LWP 13218) exited]
[Thread 0x7fffa9265700 (LWP 13219) exited]
[Thread 0x7fffa9163700 (LWP 13225) exited]
[Thread 0x7fffa9138700 (LWP 13226) exited]
[Thread 0x7fffa90b7700 (LWP 13229) exited]
[Thread 0x7fffa908c700 (LWP 13230) exited]
[Thread 0x7fffa9abc700 (LWP 13212) exited]
[Thread 0x7fffb411b700 (LWP 13215) exited]
[Thread 0x7fffa920f700 (LWP 13221) exited]
[Thread 0x7fffa923a700 (LWP 13220) exited]
[Thread 0x7fffb699f700 (LWP 13211) exited]
[Thread 0x7fffa91b9700 (LWP 13223) exited]
[Thread 0x7fffa91e4700 (LWP 13222) exited]
[Thread 0x7fffa918e700 (LWP 13224) exited]
[Thread 0x7fffa910d700 (LWP 13227) exited]
[Thread 0x7ffff7fed740 (LWP 13207) exited]
[Inferior 1 (process 13207) exited normally]
(gdb) quit
```

通过上面的例子，我们可以看到erlang为了方便调试，提供了专门的工具库：etp-commands。

使用etp-commands可以方便地查看erlang的内部数据和相关的调试信息，更多的etp-commands用法可以通过`etp-help`命令查询。


## 接下来，我们看下如何通过进程pid进行调试。


首先，启动erlang。
```shell
$/usr/local/otp_src_27.3/bin/cerl -debug
```

通过ps命令查看erlang的进程PID
```bash
$ps aux | grep beam

root     15294  5.1  2.1 2232708 39580 pts/2   Sl+  16:18   0:00 /usr/local/otp_src_27.3/bin/x86_64-pc-linux-gnu/beam.debug.smp -- -root /usr/local/otp_src_27.3 -bindir /usr/local/otp_src_27.3/bin/x86_64-pc-linux-gnu -progname /usr/local/otp_src_27.3/bin/cerl -debug -- -home /root -- -emu_type debug --
```

这里查询出来的进程PID为15294

直接使用gdb调试进程， 输入命令`gdb -p 15294`
```shell
$gdb -p 15294

GNU gdb (GDB) Red Hat Enterprise Linux 7.6.1-120.el7
Copyright (C) 2013 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-redhat-linux-gnu".
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Attaching to process 15294
Reading symbols from /usr/local/otp_src_27.3/bin/x86_64-pc-linux-gnu/beam.debug.smp...done.
Reading symbols from /lib64/libutil.so.1...(no debugging symbols found)...done.
Loaded symbols for /lib64/libutil.so.1
Reading symbols from /lib64/libdl.so.2...(no debugging symbols found)...done.
Loaded symbols for /lib64/libdl.so.2
Reading symbols from /lib64/libm.so.6...(no debugging symbols found)...done.
Loaded symbols for /lib64/libm.so.6
Reading symbols from /lib64/libtinfo.so.5...Reading symbols from /lib64/libtinfo.so.5...(no debugging symbols found)...done.
(no debugging symbols found)...done.
Loaded symbols for /lib64/libtinfo.so.5
Reading symbols from /lib64/libpthread.so.0...(no debugging symbols found)...done.
[New LWP 15350]
[New LWP 15349]
[New LWP 15348]
[New LWP 15347]
[New LWP 15346]
[New LWP 15345]
[New LWP 15344]
[New LWP 15343]
[New LWP 15342]
[New LWP 15341]
[New LWP 15340]
[New LWP 15339]
[New LWP 15338]
[New LWP 15337]
[New LWP 15336]
[New LWP 15335]
[New LWP 15333]
[New LWP 15332]
[New LWP 15331]
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib64/libthread_db.so.1".
Loaded symbols for /lib64/libpthread.so.0
Reading symbols from /lib64/librt.so.1...(no debugging symbols found)...done.
Loaded symbols for /lib64/librt.so.1
Reading symbols from /lib64/libc.so.6...(no debugging symbols found)...done.
Loaded symbols for /lib64/libc.so.6
Reading symbols from /lib64/ld-linux-x86-64.so.2...(no debugging symbols found)...done.
Loaded symbols for /lib64/ld-linux-x86-64.so.2
Reading symbols from /lib64/libnss_files.so.2...(no debugging symbols found)...done.
Loaded symbols for /lib64/libnss_files.so.2
0x00007f44382f0b43 in select () from /lib64/libc.so.6
Missing separate debuginfos, use: debuginfo-install glibc-2.17-326.el7_9.3.x86_64 ncurses-libs-5.9-14.20130511.el7_4.x86_64
```


同样以length函数为例，尝试断点length函数，输入命令`break length_1`。
```bash
(gdb) break length_1
Breakpoint 1 at 0x5270f4: file beam/erl_bif_guard.c, line 218.
```

按`c`继续
```bash
(gdb) c
Continuing.
[Switching to Thread 0x7f43f4dd8700 (LWP 15336)]
```

然后在刚才运行的erlang控制台调用`erlang:length([1,2,3]).`。

断点成功，按`n`单步执行。

```bash
Breakpoint 1, length_1 (A__p=0x1cb4bc0, BIF__ARGS=0x7f4439380300, A__I=0x7f43f69b0948) at beam/erl_bif_guard.c:218
218         args[0] = BIF_ARG_1;
(gdb) n
219         args[1] = make_small(0);
(gdb) n
220         args[2] = BIF_ARG_1;
```


尝试打印args[0], 输入命令`p args[0]`。
```bash
(gdb) p args[0]
$1 = 139929653075417
```

显示值为：139929653075417

调用etp-commands提供的方法，输入命令`etp args[0]`。
```bash
(gdb) etp args[0]
Undefined command: "etp".  Try "help".
```

这里提示`Undefined command: "etp". `，说明没有加载etp-commands。

通过source命令，加载etp-commands，这里etp-commands的路径为`/usr/local/otp_src_27.3/erts/etc/unix/etp-commands.in`

```bash
(gdb) source /usr/local/otp_src_27.3/erts/etc/unix/etp-commands.in

%---------------------------------------------------------------------------
% Use etp-help for a command overview and general help.
%
% To use the Erlang support module, the environment variable ROOTDIR
% must be set to the toplevel installation directory of Erlang/OTP,
% so the etp-commands file becomes:
%     $ROOTDIR/erts/etc/unix/etp-commands
% Also, erl and erlc must be in the path.
%---------------------------------------------------------------------------
etp-set-max-depth 20
etp-set-max-string-length 100
--------------- System Information ---------------
OTP release: 27
ERTS version: 15.2.3
Arch: x86_64-pc-linux-gnu
Endianness: Little
Word size: 64-bit
BeamAsm support: no
SMP support: yes
Thread support: yes
Kernel poll: Supported and used
Debug compiled: yes
Lock checking: yes
Lock counting: no
Node name: nonode@nohost
Number of schedulers: 2
Number of async-threads: 1
--------------------------------------------------

```

再次输入命令`etp args[0]`，显示值为：[1,2,3]
```bash
(gdb) etp args[0]
[1,2,3].

```

通过以上两个调试例子，简单地介绍了如何对erlang进行调试。总的来说，erlang已经提供了丰富的调试工具。

参考地址:

https://www.erlang.org/doc/system/install#advanced-configuration-and-build-of-erlang-otp_building_how-to-build-a-debug-enabled-erlang-runtime-system
