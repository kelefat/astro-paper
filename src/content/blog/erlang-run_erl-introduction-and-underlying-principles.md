---
author: Kelefat
pubDatetime: 2025-03-28T22:40:00Z
modDatetime: 2024-03-28T22:40:00Z
title: Erlang 工具run_erl使用介绍和原理
tags:
  - Erlang
  - Erlang源码
description:
  Erlang 工具run_erl使用介绍和原理
---


Erlang提供了run_erl工具，可以用来启动erlang；同时也提供了to_erl工具，用来连接由run_erl启动的erlang进程。

用AI翻译了官网手册介绍，原文地址(https://www.erlang.org/doc/apps/erts/run_erl_cmd.html)：
```
run_erl 程序是特定于 Unix 系统的工具。它用于重定向标准输入和标准输出流，以便记录所有输出。此外，它还允许 to_erl 程序连接到 Erlang 控制台，从而可以远程监视和调试嵌入式系统。

有关详细的使用信息，请参阅系统文档中的《嵌入式系统用户指南》。

运行方式：run_erl [-daemon] pipe_dir/ log_dir "exec command arg1 arg2 ..."

参数说明：
-daemon —— 强烈推荐使用此选项。它使 run_erl 在后台运行，完全脱离任何控制终端，并立即返回给调用方。如果不使用该选项，则需要使用多种 shell 技巧来完全将 run_erl 与启动它的终端分离。该选项必须作为 run_erl 命令行上的第一个参数。

pipe_dir —— 命名管道的目录，通常是 /tmp/。必须以 /（斜杠）结尾，例如 /tmp/epipes/，而不能是 /tmp/epipes。

log_dir —— 日志文件目录，包括：

一个日志文件 run_erl.log，用于记录 run_erl 本身的进度和警告信息。

最多五个日志文件（默认每个最大 100 KB），记录标准输入和标准输出的内容。（日志数量和大小可以通过环境变量修改，详见下文“环境变量”部分。）

当日志文件满后，run_erl 会删除最旧的日志文件，并重新使用它。

"exec command arg1 arg2 ..." —— 用于指定要执行的程序的字符串，通常第二个字段是类似 erl 的命令名称。
``

按手册提示，我们先创建目录，然后启动run_erl。

```bash
$mkdir -p /tmp/erlang_pipe /tmp/erlang_log
$run_erl -daemon /tmp/erlang_pipe/ /tmp/erlang_log/ "exec erl"
```

输入`ps aux | grep erl`，看看有没有启动成功。
```bash
$ps aux | grep erl
root     18002  0.0  0.0  14140   688 ?        S    15:40   0:00 run_erl -daemon /tmp/erlang_pipe/ /tmp/erlang_log/ exec erl
root     18003  0.3  0.9 2207008 18352 pts/2   Ssl+ 15:40   0:00 /usr/local/lib/erlang/erts-10.7.2.19/bin/beam.smp -- -root /usr/local/lib/erlang -progname erl -- -home /root --
```

以上所示，启动成功。

查看/tmp/erlang_pipe/
```bash
$ll /tmp/erlang_pipe/
total 0
prw------- 1 root root 0 Mar 27 15:45 erlang.pipe.1.r
prw------- 1 root root 0 Mar 27 15:45 erlang.pipe.1.w
```
从文件名字来看，应该是管道1负责读，管道2负责写。`prw-------`中的p说明是命名管道（pipe）

再看看/tmp/erlang_log/
```bash
$ll /tmp/erlang_log/
total 8
-rw-r--r-- 1 root root 195 Mar 27 15:45 erlang.log.1
-rw-r--r-- 1 root root 236 Mar 27 15:45 run_erl.log
```
如手册所说，erlang.log.1为erlang日志，run_erl.lgo为工具日志。


现在尝试用to_erl连接这个erlang控制台。

先看官网说明，原文地址(https://www.erlang.org/doc/system/embedded#to_erl)
```
该程序用于连接（attach）到一个由 run_erl 启动的正在运行的 Erlang 运行时系统。

用法：to_erl [pipe_name | pipe_dir]
其中，pipe_name 默认为 /tmp/erlang.pipe.N。

说明：
to_erl 允许你连接到 run_erl 启动的 Erlang 进程，方便进行交互、监控和调试。

断开连接（但不退出 Erlang 系统），请按 Ctrl+D。
```

按手册指示，输入`to_erl /tmp/erlang_pipe/erlang.pipe.1`
```bash
$to_erl /tmp/erlang_pipe/erlang.pipe.1
Attaching to /tmp/erlang_pipe/erlang.pipe.1 (^D to exit)

1> 
```
连接成功，这里要注意是是按`Ctrl+D`退出to_erl，否则会退出Erlang。

看完run_erl和to_erl的用法后，我们看看与平时erlang放后台运行的方式有什么不同？并且为什么需要run_erl这种工具？

erl使用参数-detached，可以将erlang放到后台运行，输入`erl -name server@127.0.0.1 -setcookie mycookie -detached`。
然后可通过remsh模式远程登录节点，如`erl -name client@127.0.0.1 -setcookie mycookie -remsh server@127.0.0.1`。

以这种方式启动的erlang控制台，只能够通过远程连接节点的方式进入控制台。假设出现类似system_limit之类的问题，这种方式会连接失败，详见[Erlang 源码阅读笔记：端口数（port_limit）是如何作用的]这篇文章。而run_erl方式启动的，通过to_erl依然能连接上。为什么to_erl还能连接上，我们来看看run_erl的实现原理。

run_erl所在源码文件`erts/etc/unix/run_erl.c`

```c
/* 
 * Module: run_erl.c
 * 
 * This module implements a reader/writer process that opens two specified 
 * FIFOs, one for reading and one for writing; reads from the read FIFO
 * and writes to stdout and the write FIFO.
 *
  ________                            _________ 
 |        |--<-- pipe.r (fifo1) --<--|         |
 | to_erl |                          | run_erl | (parent)
 |________|-->-- pipe.w (fifo2) -->--|_________|
                                          ^ master pty
                                          |
                                          | slave pty
                                      ____V____ 
                                     |         |
                                     |  "erl"  | (child)
                                     |_________|
*/
```
从源码中的注释，可以知道run_erl的工作流程，我们看看源码细节。
```c
int main(int argc, char **argv)
{
  int childpid;
  int sfd = -1;
  int fd;
  char *p, *ptyslave=NULL;
  int i = 1;
  int off_argv;
  int calculated_pipename = 0;
  int highest_pipe_num = 0;
  int sleepy_child = 0;
//...省略部分代码...
  /*
   * Open master pseudo-terminal
   */

  if ((mfd = open_pty_master(&ptyslave, &sfd)) < 0) {	//创建pty， mfd为主设备文件描述符， sfd为从设备文件描述符， ptyslave为终端名称,如/dev/pts/2
    ERRNO_ERR0(LOG_ERR,"Could not open pty master");
    exit(1);
  }

  /* 
   * Now create a child process
   */

  if ((childpid = fork()) < 0) { // fork一个子进程，用来运行erlang
    ERRNO_ERR0(LOG_ERR,"Cannot fork");
    exit(1);
  }
  if (childpid == 0) {	// childpid 等于 0，即新开的子进程
      if (sleepy_child)
          sleep(1);

    /* Child */	// 注释也标明是子进程
    sf_close(mfd);
//......
#if defined(HAVE_OPENPTY) && defined(TIOCSCTTY)
    else {
        /* sfd is from open_pty_master 
         * openpty -> fork -> login_tty (forkpty)
         * 
         * It would be preferable to implement a portable 
         * forkpty instead of open_pty_master / open_pty_slave
         */
        /* login_tty(sfd);  <- FAIL */
        ioctl(sfd, TIOCSCTTY, (char *)NULL); // 将终端输绑定sfd，那么终端的输出输入都由sfd负责
    }
#endif
    }
//......
    if (dup(sfd) != 0 || dup(sfd) != 1 || dup(sfd) != 2) {
      status("Cannot dup\n");
    }
    sf_close(sfd);
	// 执行command，运行erl
    exec_shell(argv+off_argv); /* exec_shell expects argv[2] to be */
                        /* the command name, so we have to */
                        /* adjust. */
} else {
    /* Parent */	//父进程，即run_erl进程
    /* Ignore the SIGPIPE signal, write() will return errno=EPIPE */
    struct sigaction sig_act;
    sigemptyset(&sig_act.sa_mask);
    sig_act.sa_flags = 0;
    sig_act.sa_handler = SIG_IGN;
    sigaction(SIGPIPE, &sig_act, (struct sigaction *)NULL);

    sigemptyset(&sig_act.sa_mask);
    sig_act.sa_flags = SA_NOCLDSTOP;
    sig_act.sa_handler = catch_sigchild;
    sigaction(SIGCHLD, &sig_act, (struct sigaction *)NULL);

    /*
     * read and write: enter the workloop
     */

    pass_on(childpid);//监听键盘输入
  }
  return 0;
} /* main() */
```

具体看看open_pty_master的实现
```c
/* open_pty_master()
 * Find a master device, open and return fd and slave device name.
 */

#ifdef HAVE_WORKING_POSIX_OPENPT
   /*
    * Use openpty() on OpenBSD even if we have posix_openpt()
    * as there is a race when read from master pty returns 0
    * if child has not yet opened slave pty.
    * (maybe other BSD's have the same problem?)
    */
#  if !(defined(__OpenBSD__) && defined(HAVE_OPENPTY))
#    define TRY_POSIX_OPENPT
#  endif
#endif

static int open_pty_master(char **ptyslave, int *sfdp)
{
  int mfd;
//......
  {
      static char slave[SLAVE_SIZE];
#  undef SLAVE_SIZE
      if (openpty(&mfd, sfdp, slave, NULL, NULL) == 0) {//调用系统提供的openpty方法, 分别返回主设备的文件描述符，从设备的文件描述符
          *ptyslave = slave;	// 终端名字，如/dev/pts/2
          return mfd;		// 设备的文件描述符
      }
  }
```
```c
/* pass_on()
 * Is the work loop of the logger. Selects on the pipe to the to_erl
 * program erlang. If input arrives from to_erl it is passed on to
 * erlang.
 */
static void pass_on(pid_t childpid)
{
    int len;
    fd_set readfds;
    fd_set writefds;
    fd_set* writefds_ptr;
    struct timeval timeout;
    time_t last_activity;
    char buf[BUFSIZ];
    char log_alive_buffer[ALIVE_BUFFSIZ+1];
    int lognum;
    int rfd, wfd=0, lfd=0;
    int maxfd;
    int ready;
    int got_some = 0; /* from to_erl */
//...省略部分代码...

 /*
         * Write any pending output first.
         */
        if (FD_ISSET(wfd, &writefds)) {
            int written;
            char* buf = outbuf_first();	// 这里缓存了erlang的返回数据，现在先取出，然后往wfd写入，发给to_erl

            len = outbuf_size();
            written = sf_write(wfd, buf, len);	// 往wfd写入，发给to_erl
            if (written < 0 && errno == EAGAIN) {
                /*
                 * Nothing was written - this is really strange because
                 * select() told us we could write. Ignore.
                 */
            } else if (written < 0) {
                /*
                 * A write error. Assume that to_erl has terminated.
                 */
                clear_outbuf();
                sf_close(wfd);
                wfd = 0;
            } else {
                /* Delete the written part (or all) from the buffer. */
                outbuf_delete(written);
            }
        }
        /*
         * Read master pty and write to FIFO.	// 从mfd读出erlang返回数据然后往wfd写入，to_erl再读取
         */
        if (FD_ISSET(mfd, &readfds)) {
#ifdef DEBUG
            status("Pty master read; ");
#endif      
            if ((len = sf_read(mfd, buf, BUFSIZ)) <= 0) {	// 从mfd读取数据，即上面通过open_pty_master返回的master pty 设备文件描述符
                int saved_errno = errno;
                sf_close(rfd);
                if(wfd) sf_close(wfd);
                sf_close(mfd);
                unlink(fifo1);
                unlink(fifo2);
                if (len < 0) {
                    errno = saved_errno;
                    if(errno == EIO)
                        ERROR0(LOG_ERR,"Erlang closed the connection.");
                    else
                        ERRNO_ERR0(LOG_ERR,"Error in reading from terminal");
                    exit(1);
                }
                exit(0);
            }
            
            write_to_log(&lfd, &lognum, buf, len);
            
            /*
             * Save in the output queue.
             */
            
            if (wfd) {
                outbuf_append(buf, len);	// 将mfd返回的数据缓存起来，下次再发送
            }
        }
//...省略部分代码...
       /*
         * Read from FIFO, write to master pty	// 从rfd里读取数据，然后往mfd写入
         */
        if (FD_ISSET(rfd, &readfds)) {
#ifdef DEBUG
            status("FIFO read; ");
#endif
            if ((len = sf_read(rfd, buf, BUFSIZ)) < 0) {	// 从rfd读取to_erl写入的指令，保存在buf
                sf_close(rfd);
                if(wfd) sf_close(wfd);
                sf_close(mfd);
                unlink(fifo1);
                unlink(fifo2);
                ERRNO_ERR0(LOG_ERR,"Error in reading from FIFO.");
                exit(1);
            }
//...省略部分代码...

                /* Write the message */
#ifdef DEBUG
                status("Pty master write; ");
#endif
                len = extract_ctrl_seq(buf, len);

                if(len==1 && buf[0] == '\003') {
                    kill(childpid,SIGINT);
                }
                else if (len>0 && write_all(mfd, buf, len) != len) { // 往mfd写入数据，发送给erlang
                    ERRNO_ERR0(LOG_ERR,"Error in writing to terminal.");
                    sf_close(rfd);
                    if(wfd) sf_close(wfd);
                    sf_close(mfd);
                    exit(1);
                }
            }
```

通过源码，我们总结一下，run_erl调用系统提供的openpty函数，分别生成mfd和sfd，两个文件描述符。然后fork一个子进程，子进程的输入输出由sfd负责，而父进程则对mfd进行读写。
mfd和sfd会自动同步数据。to_erl与run_erl通过rfd和wfd两个文件描述符进行交互，然后run_erl将to_erl的输入写入mfd，然后将mfd的数据返回给to_erl。

再重温一下源码开头的注释所示的流程图。

```
  ________                            _________
 |        |--<-- pipe.r (fifo1) --<--|         |
 | to_erl |                          | run_erl | (parent)
 |________|-->-- pipe.w (fifo2) -->--|_________|
                                          ^ master pty
                                          |
                                          | slave pty
                                      ____V____
                                     |         |
                                     |  "erl"  | (child)
                                     |_________|
```




我们回顾一下上面的两个问题？

run_erl与平时erlang放后台运行的方式有什么不同？

run_erl是使用pty建立erlang控制台，模拟终端操作，通过pty可以与erlang控制台交互。

而使用`erl -detached`是直接以后台进程形式运行，只能通过远程节点的方式进入控制台。

类似的工具有screen，为什么还需要run_erl这种工具？

像erlang这种系统，要考虑嵌入式系统环境，嵌入式系统为了稳定安全工作，可能没有安装太多的三方工具，甚至不需要联网，就是一个简单的本地部署。

因此将run_erl这种工具直接集成到erlang里，可以在类似的系统环境里提供更可靠的维护方式。

补充：
从run_erl的源码中，我们可以看出run_erl不局限于只运行erlang。是一个独立的工具。

尝试运行`python`，输入`run_erl -daemon /tmp/erlang_pipe/ /tmp/erlang_log/ "python"`
```bash
$run_erl -daemon /tmp/erlang_pipe/ /tmp/erlang_log/ "python" 
```
再输入`to_erl /tmp/erlang_pipe/erlang.pipe.3`
```bash
$to_erl /tmp/erlang_pipe/erlang.pipe.3
Attaching to /tmp/erlang_pipe/erlang.pipe.3 (^D to exit)
>>> print("hello")
hello
>>>
```
成功进入python控制台。
