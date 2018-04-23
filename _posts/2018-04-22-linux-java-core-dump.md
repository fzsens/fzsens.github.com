---
layout: post
title: Java Linux Core Dump分析
date: 2018-04-22
categories: java
tags: [java]
description: 使用Linux的Core Dump技术分析疑难杂症
---


# Java Core Dump 分析  
Core Dump，当一个进程出现严重的错误导致奔溃或者异常退出的之前，操作系统会创建一个Core Dump。Core Dump文件提供了系统的内存和进程信息，对于我们定位和处理问题，可以提供非常多的帮助。  
对于Java开发而言，一般我们会添加` -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/home/vagrant/export/dump.hprof `参数，用于发送OOM的时候，将堆栈信息导出，当如果非OOM的异常，则无法顺利得到这些信息。在出现一些极端的异常的时候，可以通过配置生成Core Dump来进行分析和问题排查。  
### 设置  
默认在Linux的各个发行版本中，是禁止生成Core Dump的，  
`ulimit -c`  
默认情况下返回 0 ，表示不允许创建Core Dump，如果要调整为允许创建  
`ulimit -c unlimited`  
`unlimited` 标识不限制Core Dump文件的大小，为了不丢失重要的信息，一般设置为`unlimited`。并且这个设置只会对当前session的程序生效，当session关闭退出只会，就会失效，如果要长期开启，可以在`/etc/profile`中写入`ulimit -c unlimited >/dev/null 2>&1 `，或者在程序启动脚本中加入`ulimit -c unlimited >/dev/null 2>&1`，并在启动只会恢复。更加建议使用后者，这样可以有针对性地对程序进行Core Dump生成，而不会影响其他的程序，可以查看`/proc/${PID}/limits`确定指定进程的参数设置  
````java
    vagrant@precise64:~/opt$ cat /proc/1778/limits
    Limit                     Soft Limit           Hard Limit           Units
    Max cpu time              unlimited            unlimited            seconds
    Max file size             unlimited            unlimited            bytes
    Max data size             unlimited            unlimited            bytes
    Max stack size            8388608              unlimited            bytes
    Max core file size        0                    unlimited            bytes
    Max resident set          unlimited            unlimited            bytes
    Max processes             2782                 2782                 processes
    Max open files            1024                 4096                 files
    Max locked memory         65536                65536                bytes
    Max address space         unlimited            unlimited            bytes
    Max file locks            unlimited            unlimited            locks
    Max pending signals       2782                 2782                 signals
    Max msgqueue size         819200               819200               bytes
    Max nice priority         0                    0
    Max realtime priority     0                    0
    Max realtime timeout      unlimited            unlimited            us
````  
查看Core Dump生成的位置  
````shell  
    cat /proc/sys/kernel/core_pattern
````  
可以通过修改`/etc/sysctl.conf`的方式修改生成Core Dump存储的位置
````shell
    kernel.core_uses_pid = 1
    kernel.core_pattern = /tmp/core-%e-%s-%u-%g-%p-%t
    fs.suid_dumpable = 2
````  
使用`sysctl -p`使其生效  
### 模拟
在完成了以上的配置之后，为了模拟应用程序，可以使用  
````shell
    kill -ABRT PID
    kill -11 PID
````  
这两个指令中的一个来关闭进程，并生成Core Dump，不能使用 -9 ，因为-9 直接将进程关闭，不会给系统留下 Dump的时间。  
### 分析  
首先是`hs_err_pidxxx.log`日志，程序的异常关闭一般都会生成一个这样的异常日志，主要记录了一些系统和`JVM`配置和栈信息，可以提供一些解决问题的切入点。但是日志文件提供的信息一般比较简略，有时候甚至不会生成，因此我们重点分析Core Dump文件。  
````shell
    gdb $JAVA_HOME/bin/java core-java-6-1000-1000-2109-1523173609
````  
使用`where`指令可以查看进程`abort()`的调用栈，可以查看在什么地方触发了异常关闭。  
````shell  
    (gdb) where
    #0  0x00007f05d66f8445 in raise () from /lib/x86_64-linux-gnu/libc.so.6
    #1  0x00007f05d66fbbab in abort () from /lib/x86_64-linux-gnu/libc.so.6
    #2  0x00007f05d5ff1295 in os::abort(bool) () from /home/vagrant/opt/jdk1.8.0_161/jre/lib/amd64/server/libjvm.so
    #3  0x00007f05d6194d53 in VMError::report_and_die() () from /home/vagrant/opt/jdk1.8.0_161/jre/lib/amd64/server/libjvm.so
    #4  0x00007f05d5ff6f4f in JVM_handle_linux_signal () from /home/vagrant/opt/jdk1.8.0_161/jre/lib/amd64/server/libjvm.so
    #5  0x00007f05d5fecff3 in signalHandler(int, siginfo*, void*) () from /home/vagrant/opt/jdk1.8.0_161/jre/lib/amd64/server/libjvm.so
    #6  <signal handler called>
    #7  0x00007f05d6ea2146 in pthread_join () from /lib/x86_64-linux-gnu/libpthread.so.0
    #8  0x00007f05d6c8e775 in ContinueInNewThread0 () from /home/vagrant/opt/jdk1.8.0_161/bin/../lib/amd64/jli/libjli.so
    #9  0x00007f05d6c89e9a in ContinueInNewThread () from /home/vagrant/opt/jdk1.8.0_161/bin/../lib/amd64/jli/libjli.so
    #10 0x00007f05d6c8cf18 in JLI_Launch () from /home/vagrant/opt/jdk1.8.0_161/bin/../lib/amd64/jli/libjli.so
    #11 0x0000000000400696 in main ()
````  
仅仅知道关闭异常的调用栈，可能还不足以帮助我们排查问题。我们也可以使用JDK自带的分析工具，来进一步分析Core Dump中的内容|  
````shell
    jstack -J-d64 $JAVA_HOME/bin/java ./core-java-6-1000-1000-2842-1523164931
````  
可以得到对应Java进程的栈详细信息，`$JAVA_HOME/bin/java ./core-java-6-1000-1000-2842-1523164931`在这里可以等同于一个运行时的进程  
````shell
    jmap -dump:format=b,file=./2842.hprof $JAVA_HOME/bin/java ./core-java-6-1000-1000-2842-1523164931
````
可以得到对应Java进程的Heap Dump文件，使用MAT等进行分析  