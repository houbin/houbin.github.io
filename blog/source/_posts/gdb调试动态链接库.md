title: gdb调试动态链接库
date: 2016-02-29 14:58:39
tags: gdb 调试 动态链接库
---

## 起源

今天发现一个服务进程占用cpu为100%，已经无法提供服务。

## 查找问题线程

1. 查找进程中的所有线程

``` powershell
[root@ross _posts]# cd  /proc/2750/task/
[root@ross task]# ls
2750  2751  2752  2753
```

<!-- more -->

2. 查看所有的线程文件，找到使用cpu较多的线程

``` powershell
[root@ross task]# cat 2750/stat
2750 (data_collection) T 1 2750 2750 0 -1 8256 266 0 0 0 0 0 0 0 20 0 4 0 2296438 259633152 469 18446744073709551615 4194304 4517477 140737488346640 140737488346160 140737329459757 0 0 16781312 2560 18446744071579695367 0 0 17 1 0 0 0 0 0

参数含义为
pid = 2750
comm = data_collection
task_state = T	任务运行状态：T为tracing stop，S为sleeping，R为running
ppid = 1 	父进程ID为1
pgid = 2750	线程组号
sid = 2750 该任务所在的会话组ID
tty_nr = 0 该任务的tty终端的设备号
tty_pgrp = -1 终端的进程组号
task->flags = 8256 进程标志位
min_flt=266     该任务不需要从硬盘拷数据而发生的缺页（次缺页）的次数
cmin_flt=0     累计的该任务的所有的waited-for进程曾经发生的次缺页的次数目
maj_flt=0      该任务需要从硬盘拷数据而发生的缺页（主缺页）的次数
cmaj_flt=0     累计的该任务的所有的waited-for进程曾经发生的主缺页的次数目
utime=0        该任务在用户态运行的时间，单位为jiffies
stime=0        该任务在核心态运行的时间，单位为jiffies
```
由最后两项可以得出哪个线程占用CPU严重，这里假设为2753。

## gdb调试so

### gdb挂载运行进程

``` powershell
[root@ross bin]# gdb -p 2750
(gdb) info thread
  4 Thread 0x7fffee502700 (LWP 2751)  0x00007ffff687a5bc in pthread_cond_wait@@GLIBC_2.3.2 () from /lib64/libpthread.so.0
  3 Thread 0x7fffedb01700 (LWP 2752)  0x00007ffff687a5bc in pthread_cond_wait@@GLIBC_2.3.2 () from /lib64/libpthread.so.0
* 2 Thread 0x7fffed100700 (LWP 2753)  event_base_loop (base=0x655a20, flags=<value optimized out>) at event.c:1878
  1 Thread 0x7ffff7fda7e0 (LWP 2750)  0x00007ffff687722d in pthread_join () from /lib64/libpthread.so.0
```

### 查看所有动态库

``` powershell
gdb) info share
From                To                  Syms Read   Shared Object Library
0x00007ffff7b9b850  0x00007ffff7bc9c78  Yes         /usr/local/lib/libevent-2.1.so.5
0x00007ffff7960380  0x00007ffff797f358  Yes (*)     /usr/lib64/libmysqlpp.so.3

```

### 加载指定目录源码进行调试

现在源码被拷贝到/root/src/work/data_collection_server_back
查询加载的源码目录

``` ```
(gdb) c
Continuing.

Breakpoint 1, event_base_loop (base=0x123b9f0, flags=<value optimized out>) at event.c:1898
1898	event.c: No such file or directory.
	in event.c
(gdb) show dir
Source directories searched: $cdir:$cwd
(gdb) dir /root/src/libevent-2.1.5-beta_back
Source directories searched: /root/src/libevent-2.1.5-beta_back:$cdir:$cwd
(gdb) c
Continuing.

Breakpoint 1, event_base_loop (base=0x123b9f0, flags=<value optimized out>) at event.c:1898
1898			res = evsel->dispatch(base, tv_p);
(gdb) 

```

由上面可知，通过dir命令加载对应的源码路径，就可以单步调试源码

