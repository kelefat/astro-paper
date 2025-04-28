---
author: Kelefat
pubDatetime: 2024-12-18
modDatetime: 2024-12-18
title: OpenSSL导致Erlang调用crypto崩溃问题
tags:
  - Erlang
description:
  OpenSSL bug导致Erlang在windows下调用crypto:hash(sha, <<1>>)直接崩溃
---

## 问题：
在windows 10下， Erlang OTP 22, 项目在启动过程中闪退，没有任何crash日志。

## 过程：
由于没有任何crash日志，就在启动过程中打日志，最终定位问题在crypto模块的hash方法调用上。
打开werl，直接执行crypto:hash(sha, <<1>>). 马上闪退。

查找相关资料，最终发现是由于OpenSSL的bug导致。具体原因参考：[https://www.intel.com/content/www/us/en/developer/articles/troubleshooting/openssl-sha-crash-bug-requires-application-update.html]

>OpenSSL* 1.0.2 beta (Jun 2014) to OpenSSL 1.0.2k (Jan 2017) contain bugs that either cause a crash or bad SHA (Secure Hash Algorithm) values on processors with the SHA extensions, such as the recently released 10th Generation processor. Both bugs were fixed years ago; however, any application that uses the old version directly, or as one of its dependencies, will fail. 

OpenSSL版本为 1.0.2 beta (Jun 2014) ~   1.0.2k (Jan 2017) 处理SHA时，会crash。

查看当前Erlang crypto模块使用的OpenSSL版本：
执行
```erlang
crypto:info_lib().
[{<<"OpenSSL">>,268443727,<<"OpenSSL 1.0.2d 9 Jul 2015">>}]
```



## 解决：
>OpenSSL provides an environment variable control for enabling features, including one that modifies the Intel® CPU identification, OPENSSL_ia32cap:

`set OPENSSL_ia32cap=:~0x20000000`

>This disables the OpenSSL code check for SHA extensions and runs a different code path that does not contain the crashing bug.

添加windows环境变量：`OPENSSL_ia32cap` 值为：`~0x20000000`


## 补充：
后来在windows的事件日志里，发现了Erlang的崩溃日志，如下：
 ```
 werl.exe 
   0.0.0.0 
   5e6eb06c 
   crypto.dll 
   0.0.0.0 
   5e6eb229 
   c0000005 
   0000000000019063 
    
   \\?\C:\ProgramData\Microsoft\Windows\WER\Temp\WER3771.tmp.WERInternalMetadata.xml 
   \\?\C:\ProgramData\Microsoft\Windows\WER\ReportQueue\AppCrash_werl.exe_3dbf3263f8677be7a286a16bc0681c2f6549a2d4_754a5540_c23a5024-4dad-4ad1-ba12-491dbc731f4e 
```
   从日志中，可以发现crypto.dll模块异常。
