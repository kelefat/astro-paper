---
author: Kelefat
pubDatetime: 2025-04-03T22:40:00Z
modDatetime: 2025-04-03T22:40:00Z
title: PTY介绍和应用实践
tags:
  - Erlang
  - GDB
  - Erlang debug
  - etp-commands
description:
  PTY介绍和应用实践
---

从run_erl的实现原理中，我们看到了一个概念pty。那么pty是什么呢？看看AI的回答：
```
PTY（Pseudo Terminal，伪终端） 是 Linux/Unix 系统中提供的一种特殊的终端设备，它可以让程序像在真实终端（TTY）中运行一样，但实际数据的输入输出是通过程序进行控制的。

换句话说，PTY 让一个程序可以充当“虚拟用户”，与另一个程序进行交互，就像一个真实用户在终端里输入命令一样。

PTY 由 两个部分 组成：

Master 端（主端，通常叫 master_fd）

Slave 端（从端，通常叫 slave_fd）

它们成对存在，数据从 master_fd 读出，会出现在 slave_fd 里，反之亦然。
```

有了概念后，我们再重新看看run_erl和`erl -detached`的表现。

分别输入

`run_erl -daemon /tmp/erlang_pipe/ /tmp/erlang_log/ "erl"`

 `erl -name server@127.0.0.1 -setcookie mycookie -detached`

然后ps看一下进程信息，输入`ps aux| grep beam`

```bash
$ps aux| grep beam
root     18381  0.0  1.0 2207560 19340 pts/2   Ssl+ Mar27   0:01 /usr/local/lib/erlang/erts-10.7.2.19/bin/beam.smp -- -root /usr/local/lib/erlang -progname erl -- -home /root --
root     24781  0.8  0.8 2201888 16640 ?       Sl   14:54   0:00 /usr/local/lib/erlang/erts-10.7.2.19/bin/beam.smp -- -root /usr/local/lib/erlang -progname erl -- -home /r
oot -- -name server@127.0.0.1 -setcookie mycookie -noshell -noinput --
```

由run_erl启动的erlang进程有一个标识“pts/2”，而使用detached的erlang进程则显示“?”。

`pts/2`就是设备名称，对应`/dev/pts/2`

`?`意味着该进程不依赖于任何终端输入输出设备。常见的情况是后台进程、系统服务。

那么如果我是python开发者，并没有run_erl这种工具，那么怎样才能实现这种模拟终端效果呢？

Linux上最常见的工具就是screen。

我们试下使用screen启动python，输入`screen -S my_erlang_session python`
```bash
$screen -S my_erlang_session python
Python 2.7.5 (default, Nov 14 2023, 16:14:06) 
[GCC 4.8.5 20150623 (Red Hat 4.8.5-44)] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> 
```
成功启动python，按Ctrl + A，然后按 D，退出screen。

ps看一下进程信息，输入`ps aux | grep python`
```bash
$ps aux | grep python
root     29041  0.0  0.0 125884  1424 ?        Ss   15:42   0:00 SCREEN -S my_erlang_session python
root     29042  0.0  0.2 130728  4828 pts/6    Ss+  15:42   0:00 python
```

从进程信息中，我们再次看到了`pts/6`和`?`，说明screen是以后台进程的形式运行，而python则是pty形式。

输入`screen -r my_erlang_session`，回到python控制台。
```bash
$screen -r my_erlang_session
Python 2.7.5 (default, Nov 14 2023, 16:14:06)
[GCC 4.8.5 20150623 (Red Hat 4.8.5-44)] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> 
```

run_erl和screen都是采用pty形式，实现原理差不多。

这里我们试试使用python结合pty实现一个小工具，通过telnet与erlang交互。
新建server.py文件，输入以下代码（AI加持）：
```python
# -*- coding: utf-8 -*-
import sys
import os
import socket
import select
import threading
import pty
import subprocess

HOST = "0.0.0.0"  # 监听所有 IP
PORT = 4000       # 端口号

def handle_client(conn, master_fd):
    try:
        while True:
            ready, _, _ = select.select([conn, master_fd], [], [])

            # 处理客户端输入
            if conn in ready:
                data = conn.recv(1024)
                if not data:
                    break
                os.write(master_fd, data)  # 发送到 Erlang 进程

            # 读取 Erlang 的输出
            if master_fd in ready:
                try:
                    output = os.read(master_fd, 1024)
                    if output:
                        conn.sendall(output)  # 发送回客户端
                except OSError:
                    break
    finally:
        conn.close()

def run_server(cmd):
    master_fd, slave_fd = pty.openpty()
    process = subprocess.Popen([cmd], stdin=slave_fd, stdout=slave_fd, stderr=slave_fd, close_fds=True)

    server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

    server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    server_socket.bind((HOST, PORT))
    server_socket.listen(5)
    print("Run Cmd: {}".format(cmd))
    print("Server started. Host: {} Port:{}".format(HOST, PORT))

    try:
        while True:
            conn, addr = server_socket.accept()
            print("Connection from {}".format(addr))
            threading.Thread(target=handle_client, args=(conn, master_fd)).start()
    except KeyboardInterrupt:
        print("\nServer shutting down.")
    finally:
        server_socket.close()
        process.terminate()

if __name__ == "__main__":
    run_server(sys.argv[1])
```

保存然后运行，输入`python server.py "/usr/local/bin/erl"`
```bash
$python server.py "/usr/local/bin/erl"
Run Cmd: /usr/local/bin/erl
Server started. Host: 0.0.0.0 Port:4000
```

启动成功，ps看下进程信息，输入`ps aux | grep erl`

```bash
$ps aux | grep erl
root       442  0.4  0.4 141960  8076 pts/1    S+   16:35   0:00 python server.py /usr/local/bin/erl
root       443  2.4  0.9 2206496 17792 pts/1   Sl+  16:35   0:00 /usr/local/lib/erlang/erts-10.7.2.19/bin/beam.smp -- -root /usr/local/lib/erlang -progname erl -- -home /root --
```
再次看到`pts/1`，但没有`?`，因为这里没有实现python放后台运行。

现在通过telnet连接erlang，输入`telnet 127.0.0.1 4000`
```bash
$telnet 127.0.0.1 4000
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
Erlang/OTP 22 [erts-10.7.2.19] [source] [64-bit] [smp:2:2] [ds:2:2:10] [async-threads:1]

Eshell V10.7.2.19  (abort with ^G)
1>  
```

看到熟悉的Erlang shell，输入`self().`
```bash
1> self().
self().
<0.79.0>
2> 
```

正常运行。


