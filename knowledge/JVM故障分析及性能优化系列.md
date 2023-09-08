---
title: JVM故障分析及性能优化系列
categories: []
tags: []
halo:
  site: http://blog.jamesyoung94.top:8081
  name: 386d8c15-fa60-4262-b754-b1f0671bed75
  publish: true
---
# JVM 故障分析及性能优化系列

## [JVM 故障分析及性能优化系列之一：使用 jstack 定位线程堆栈信息](https://www.javatang.com/archives/2017/10/19/33151873.html)

- thread dump 主要记录 JVM 在某一时刻各个线程执行的情况，以栈的形式显示，是一个文本文件。通过对 thread dump 文件可以分析出程序的问题出现在什么地方，从而定位具体的代码然后进行修正。thread dump 需要结合占用系统资源的**线程 id**进行分析才有意义。
- heap dump 主要记录了在某一时刻 JVM 堆中对象使用的情况，即某个时刻 JVM 堆的快照，是一个二进制文件，主要用于分析哪些对象占用了太多的堆空间，从而发现导致内存泄漏的对象。

上面两种 dump 文件都具有**实时性**，因此需要在服务器出现问题的时候生成，并且**多生成几个文件**，方便进行对比分析。下面我们先来说一下如何生成 thread dump。

当服务器出现高 CPU 的时候，首先执行 `top -c` 命令动态显示进程及占用资源的排行.

需要将占用 CPU 最高进程中的线程打印出来，可以用 `top -bn1 -H -p <pid>` 命令.我个人请喜欢用 `ps -mp <pid> -o THREAD,tid,time | sort -k2r` 命令查看，后面的 sort 参数根据线程占用的 cpu 比例进行排序.
`ps -mp 1023 -o THREAD,tid,time | sort -k2r`

```sh
#!/bin/bash
if [ $# -le 0 ]; then
    echo "usage: $0 <pid> [line-number]"
    exit 1
fi

# java home
if test -z $JAVA_HOME
then
    JAVA_HOME='/usr/local/jdk'
fi

#pid
pid=$1
# checking pid
if test -z "$($JAVA_HOME/bin/jps -l | cut -d ' ' -f 1 | grep $pid)"
then
    echo "process of $pid is not exists"
    exit
fi

#line number
linenum=$2
if test -z $linenum
then
    linenum=10
fi

stackfile=stack$pid.dump
threadsfile=threads$pid.dump

# generate java stack
$JAVA_HOME/bin/jstack -l $pid >> $stackfile
ps -mp $pid -o THREAD,tid,time | sort -k2r | awk '{if ($1 !="USER" && $2 != "0.0" && $8 !="-") print $8;}' | xargs printf "%x\n" >> $threadsfile
tids="$(cat $threadsfile)"
for tid in $tids
do
    echo "------------------------------ ThreadId ($tid) ------------------------------"
    cat $stackfile | grep 0x$tid -A $linenum
done

rm -f $stackfile $threadsfile
```

## [JVM 故障分析及性能优化系列之二：jstack 生成的 Thread Dump 日志结构解析](https://www.javatang.com/archives/2017/10/19/51301886.html)

一个典型的 thread dump 文件主要由一下几个部分组成：
![Alt text](http://blog.jamesyoung94.top:8081/upload/2021-06-01-15-56-09.png)

### 第一部分：Full thread dump identifier

这一部分是内容最开始的部分，展示了快照文件的生成时间和 JVM 的版本信息。

```log
2021-06-01 00:24:04
Full thread dump Java HotSpot(TM) 64-Bit Server VM (25.91-b14 mixed mode):
```

### 第二部分：Java EE middleware, third party & custom application Threads

这是整个文件的核心部分，里面展示了 JavaEE 容器（如 tomcat、resin 等）、自己的程序中所使用的线程信息。这一部分详细的含义见 Java 内存泄漏分析系列之四：jstack 生成的 Thread Dump 日志线程状态分析。

### 第三部分：HotSpot VM Thread

### 第四部分：HotSpot GC Thread

### 第五部分：JNI global references count

## [JVM 故障分析及性能优化系列之三：jstat 命令的使用及 VM Thread 分析](https://www.javatang.com/archives/2017/10/20/12131956.html)

使用 `jstat -gc <pid> <period> <times>` 命令查看 gc 的信息.

```sh
[webedit@vm130-67-84 ~]$ jstat -gc 29595
 S0C    S1C    S0U    S1U      EC       EU        OC         OU         MC       MU       CCSC    CCSU    YGC    YGCT       FGC    FGCT     GCT
 0.0   163840.0  0.0   163840.0 819200.0 573440.0 17891328.0 13482132.9 347544.0 322186.9 34712.0 30260.5 101407 11292.120   3     61.653 11353.773

```

结果中每个项目的含义可以参考官方对 [jstat 的文档](https://docs.oracle.com/javase/1.5.0/docs/tooldocs/share/jstat.html)，简单翻译如下：

S0C: Young Generation 第一个 survivor space 的内存大小 (kB).
S1C: Young Generation 第二个 survivor space 的内存大小 (kB).
S0U: Young Generation 第一个 Survivor space 当前已使用的内存大小 (kB).
S1U: Young Generation 第二个 Survivor space 当前已经使用的内存大小 (kB).
EC: Young Generation 中 eden space 的内存大小 (kB).
EU: Young Generation 中 Eden space 当前已使用的内存大小 (kB).
OC: Old Generation 的内存大小 (kB).
OU: Old Generation 当前已使用的内存大小 (kB).
PC: Permanent Generation 的内存大小 (kB)
PU: Permanent Generation 当前已使用的内存大小 (kB).
YGC: 从启动到采样时 Young Generation GC 的次数
YGCT: 从启动到采样时 Young Generation GC 所用的时间 (s).
FGC: 从启动到采样时 Old Generation GC 的次数.
FGCT: 从启动到采样时 Old Generation GC 所用的时间 (s).
GCT: 从启动到采样时 GC 所用的总时间 (s).
JDK8 的结果稍微有所不同，结果含义可以参考：http://docs.oracle.com/javase/8/docs/technotes/tools/unix/jstat.html。

## [JVM 故障分析及性能优化系列之四：jstack 生成的 Thread Dump 日志线程状态](https://www.javatang.com/archives/2017/10/25/36441958.html)

```log
"NettyClientWorkerThread_4" #23451 daemon prio=5 os_prio=0 tid=0x00007fca7407f800 nid=0x45d4 waiting on condition [0x00007fc992405000]
   java.lang.Thread.State: TIMED_WAITING (parking)
        at sun.misc.Unsafe.park(Native Method)
        - parking to wait for  <0x000000034a750ef0> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
        at java.util.concurrent.locks.LockSupport.parkNanos(LockSupport.java:215)
        at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.awaitNanos(AbstractQueuedSynchronizer.java:2078)
        at java.util.concurrent.LinkedBlockingQueue.poll(LinkedBlockingQueue.java:467)
        at io.netty.util.concurrent.SingleThreadEventExecutor.takeTask(SingleThreadEventExecutor.java:250)
        at io.netty.util.concurrent.DefaultEventExecutor.run(DefaultEventExecutor.java:64)
        at io.netty.util.concurrent.SingleThreadEventExecutor$5.run(SingleThreadEventExecutor.java:858)
        at java.lang.Thread.run(Thread.java:745)
```

以上依次是：

- "NettyClientWorkerThread_4" 线程名称：如果使用 java.lang.Thread 类生成一个线程的时候，线程名称为 Thread-(数字) 的形式；
- daemon 线程类型：线程分为守护线程 (daemon) 和非守护线程 (non-daemon) 两种，通常都是守护线程；
- prio=10 线程优先级：默认为 5，数字越大优先级越高；
- tid=0x00007fca7407f800 JVM 线程的 id：JVM 内部线程的唯一标识，通过 java.lang.Thread.getId()获取，通常用自增的方式实现；
- nid=0x45d4 系统线程 id：对应的系统线程 id（Native Thread ID)，可以通过 top 命令进行查看，现场 id 是十六进制的形式；
- waiting on condition 系统线程状态：这里是系统的线程状态，具体的含义见下面 [系统线程状态](###系统线程状态 "Native Thread Status") 部分；
- [0x00007fc992405000] 起始栈地址：线程堆栈调用的其实内存地址；
- java.lang.Thread.State: TIMED_WAITING (parking) JVM 线程状态：这里标明了线程在代码级别的状态，详细的内容见下面的 JVM 线程运行状态 部分。
- 线程调用栈信息：下面就是当前线程调用的详细栈信息，用于代码的分析。堆栈信息应该从下向上解读，因为程序调用的顺序是从下向上的。

### 系统线程状态 (Native Thread Status)
