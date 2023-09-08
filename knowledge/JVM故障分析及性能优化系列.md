---
title: JVM故障分析及性能优化系列
categories: []
tags:
  - JVM
  - Java
  - GC
halo:
  site: http://blog.jamesyoung94.top:8081
  name: 386d8c15-fa60-4262-b754-b1f0671bed75
  publish: true
---
# JVM 故障分析及性能优化系列

## GC相关
### 常用命令

GC 统计信息

```sh
jstat -gc ${pid}
```

使用 `jstat -gcutil ${pid} 1000` 每隔一秒打印一次 GC 统计信息。
[参考文章](https://cloud.tencent.com/developer/article/1740841)

```log
YGCT：Young GC 总时间，单位为秒
YGC：Young GC 次数
FGCT：Full GC 总时间，单位为秒
FGC：Full GC 次数
GCT：GC 总时间，是 YGCT 和 FGCT 之和
```

### GC 日志

GC 日志中的三个时间：user, sys, real
这些时间是 GC 花费的时间，

- user 是用户态耗费的时间，
- sys 是内核态耗费的时间，
- real 是整个过程实际花费的时间。

user+sys 是 CPU 时间，每个 CPU core 单独计算，所以这个时间可能会是 real 的好几倍。
[CSDN 文章](https://blog.csdn.net/u010325193/article/details/88324143)

### gc日志分析

[gceasy](https://gceasy.io/gc-index.jsp)：免费的gc日志在线分析工具，但是有文件大小和次数限制，测试50M可以。

g1日志示例：
```log
2023-09-08T19:26:53.841+0800: 95391.844: [GC concurrent-root-region-scan-start]
2023-09-08T19:26:53.926+0800: 95391.928: [GC concurrent-root-region-scan-end, 0.0841083 secs]
2023-09-08T19:26:53.926+0800: 95391.928: [GC concurrent-mark-start]
{Heap before GC invocations=27865 (full 0):
 garbage-first heap   total 18874368K, used 14434304K [0x0000000369000000, 0x000000036a002400, 0x00000007e9000000)
  region size 16384K, 92 young (1507328K), 11 survivors (180224K)                                                                   2211937,3     99%
 garbage-first heap   total 18874368K, used 14434304K [0x0000000369000000, 0x000000036a002400, 0x00000007e9000000)
  region size 16384K, 92 young (1507328K), 11 survivors (180224K)
 Metaspace       used 201644K, capacity 210651K, committed 210944K, reserved 1236992K
  class space    used 21233K, capacity 22922K, committed 23040K, reserved 1048576K
2023-09-08T19:26:59.304+0800: 95397.306: [GC pause (G1 Evacuation Pause) (young), 0.1161456 secs]
   [Parallel Time: 113.3 ms, GC Workers: 8]
      [GC Worker Start (ms):  95397306.9  95397306.9  95397306.9  95397307.0  95397307.0  95397307.0  95397307.7  95397307.7
       Min: 95397306.9, Avg: 95397307.1, Max: 95397307.7, Diff: 0.8]
      [Ext Root Scanning (ms):  3.8  4.0  3.4  4.0  4.0  3.9  3.1  2.7
       Min: 2.7, Avg: 3.6, Max: 4.0, Diff: 1.3, Sum: 28.9]
         [Thread Roots (ms):  0.0  4.1  1.4  4.0  4.0  3.9  0.7  0.7
          Min: 0.0, Avg: 2.3, Max: 4.1, Diff: 4.0, Sum: 18.8]
         [StringTable Roots (ms):  0.5  0.0  2.3  0.0  0.0  0.0  0.1  2.4
          Min: 0.0, Avg: 0.7, Max: 2.4, Diff: 2.4, Sum: 5.3]
         [Universe Roots (ms):  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0
          Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
         [JNI Handles Roots (ms):  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0
          Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
         [ObjectSynchronizer Roots (ms):  0.0  0.0  0.2  0.0  0.0  0.0  0.0  0.0
          Min: 0.0, Avg: 0.0, Max: 0.2, Diff: 0.2, Sum: 0.2]
         [FlatProfiler Roots (ms):  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0
          Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
         [Management Roots (ms):  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0
          Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
         [SystemDictionary Roots (ms):  0.0  0.0  0.0  0.0  0.0  0.0  2.3  0.0
          Min: 0.0, Avg: 0.3, Max: 2.3, Diff: 2.3, Sum: 2.3]
         [CLDG Roots (ms):  3.4  0.0  0.0  0.0  0.0  0.0  0.0  0.0
          Min: 0.0, Avg: 0.4, Max: 3.4, Diff: 3.4, Sum: 3.4]
         [JVMTI Roots (ms):  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0
          Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
         [CodeCache Roots (ms):  14.8  14.8  15.0  14.8  14.7  14.8  15.0  14.9
          Min: 14.7, Avg: 14.9, Max: 15.0, Diff: 0.2, Sum: 118.9]
         [CM RefProcessor Roots (ms):  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0
          Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
         [Wait For Strong CLD (ms):  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0
          Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
         [Weak CLD Roots (ms):  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0
          Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
         [SATB Filtering (ms):  0.2  0.0  0.0  0.0  0.0  0.0  0.0  0.0
          Min: 0.0, Avg: 0.0, Max: 0.2, Diff: 0.2, Sum: 0.2]
      [Update RS (ms):  14.6  14.2  14.4  14.3  14.2  14.3  14.4  14.4
       Min: 14.2, Avg: 14.4, Max: 14.6, Diff: 0.5, Sum: 114.8]
         [Processed Buffers:  21  31  38  19  19  29  8  22
          Min: 8, Avg: 23.4, Max: 38, Diff: 30, Sum: 187]
      [Scan RS (ms):  0.2  0.6  0.5  0.5  0.6  0.6  0.5  0.6
       Min: 0.2, Avg: 0.5, Max: 0.6, Diff: 0.4, Sum: 4.0]
      [Code Root Scanning (ms):  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0
      [Object Copy (ms):  94.2  94.2  94.6  94.2  94.2  94.2  94.2  94.6                                                            2211936,2     99%
       Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Object Copy (ms):  94.2  94.2  94.6  94.2  94.2  94.2  94.2  94.6
       Min: 94.2, Avg: 94.3, Max: 94.6, Diff: 0.4, Sum: 754.4]
      [Termination (ms):  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0
       Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.1]
         [Termination Attempts:  12  12  8  9  8  11  11  10
          Min: 8, Avg: 10.1, Max: 12, Diff: 4, Sum: 81]
      [GC Worker Other (ms):  0.1  0.0  0.0  0.0  0.0  0.0  0.1  0.0
       Min: 0.0, Avg: 0.0, Max: 0.1, Diff: 0.1, Sum: 0.3]
      [GC Worker Total (ms):  113.1  113.1  113.0  113.0  113.0  112.9  112.3  112.3
       Min: 112.3, Avg: 112.8, Max: 113.1, Diff: 0.8, Sum: 902.8]
      [GC Worker End (ms):  95397420.0  95397420.0  95397420.0  95397419.9  95397420.0  95397419.9  95397420.0  95397420.0
       Min: 95397419.9, Avg: 95397420.0, Max: 95397420.0, Diff: 0.1]
   [Code Root Fixup: 0.0 ms]
   [Code Root Purge: 0.0 ms]
   [Clear CT: 0.7 ms]
   [Other: 2.2 ms]
      [Choose CSet: 0.0 ms]
      [Ref Proc: 0.7 ms]
      [Ref Enq: 0.0 ms]
      [Redirty Cards: 0.6 ms]
         [Parallel Redirty:  0.4  0.4  0.4  0.4  0.4  0.4  0.4  0.4
          Min: 0.4, Avg: 0.4, Max: 0.4, Diff: 0.1, Sum: 3.3]
         [Redirtied Cards:  55175  55013  51861  49643  49252  47807  47194  43930
          Min: 43930, Avg: 49984.4, Max: 55175, Diff: 11245, Sum: 399875]
      [Humongous Register: 0.0 ms]
         [Humongous Total: 1]
         [Humongous Candidate: 0]
      [Humongous Reclaim: 0.0 ms]
         [Humongous Reclaimed: 0]
      [Free CSet: 0.1 ms]
         [Young Free CSet: 0.1 ms]
         [Non-Young Free CSet: 0.0 ms]
   [Eden: 1296.0M(1296.0M)->0.0B(784.0M) Survivors: 176.0M->192.0M Heap: 13.8G(18.0G)->12.8G(18.0G)]
Heap after GC invocations=27866 (full 0):
 garbage-first heap   total 18874368K, used 13426688K [0x0000000369000000, 0x000000036a002400, 0x00000007e9000000)
  region size 16384K, 12 young (196608K), 12 survivors (196608K)
 Metaspace       used 201644K, capacity 210651K, committed 210944K, reserved 1236992K
  class space    used 21233K, capacity 22922K, committed 23040K, reserved 1048576K
}
 [Times: user=0.89 sys=0.00, real=0.12 secs]
2023-09-08T19:26:59.672+0800: 95397.674: [GC concurrent-mark-end, 5.7463805 secs]
2023-09-08T19:26:59.674+0800: 95397.676: [GC remark 2023-09-08T19:26:59.674+0800: 95397.676: [Finalize Marking, 0.0006308 secs] 2023-09-08T19:26:59.675+0800: 95397.677: [GC ref-proc, 0.0044644 secs] 2023-09-08T19:26:59.679+0800: 95397.681: [Unloading 2023-09-08T19:26:59.680+0800: 95397.683: [System Dictionary Unloading, 0.0004678 secs] 2023-09-08T19:26:59.681+0800: 95397.683: [Parallel Unloading, 0.0316819 secs] 2023-09-08T19:26:59.713+0800: 95397.715: [Deallocate Metadata, 0.0002243 secs], 0.0418402 secs], 0.0635304 secs]
 [Times: user=0.40 sys=0.00, real=0.06 secs]
2023-09-08T19:26:59.739+0800: 95397.741: [GC cleanup 12G->12G(18G), 0.0098570 secs]
 [Times: user=0.06 sys=0.00, real=0.01 secs]                                                                                        2211983,8     99%
2023-09-08T19:26:59.739+0800: 95397.741: [GC cleanup 12G->12G(18G), 0.0098570 secs]
 [Times: user=0.06 sys=0.00, real=0.01 secs]
2023-09-08T19:26:59.749+0800: 95397.751: [GC concurrent-cleanup-start]
2023-09-08T19:26:59.749+0800: 95397.751: [GC concurrent-cleanup-end, 0.0000482 secs]
```

以下是相关博客的记录：
## JVM 故障分析及性能优化系列
### [JVM 故障分析及性能优化系列之一：使用 jstack 定位线程堆栈信息](https://www.javatang.com/archives/2017/10/19/33151873.html)

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

### [JVM 故障分析及性能优化系列之二：jstack 生成的 Thread Dump 日志结构解析](https://www.javatang.com/archives/2017/10/19/51301886.html)

一个典型的 thread dump 文件主要由一下几个部分组成：
![Alt text](http://blog.jamesyoung94.top:8081/upload/2021-06-01-15-56-09.png)

#### 第一部分：Full thread dump identifier

这一部分是内容最开始的部分，展示了快照文件的生成时间和 JVM 的版本信息。

```log
2021-06-01 00:24:04
Full thread dump Java HotSpot(TM) 64-Bit Server VM (25.91-b14 mixed mode):
```

#### 第二部分：Java EE middleware, third party & custom application Threads

这是整个文件的核心部分，里面展示了 JavaEE 容器（如 tomcat、resin 等）、自己的程序中所使用的线程信息。这一部分详细的含义见 Java 内存泄漏分析系列之四：jstack 生成的 Thread Dump 日志线程状态分析。

#### 第三部分：HotSpot VM Thread

#### 第四部分：HotSpot GC Thread

#### 第五部分：JNI global references count

### [JVM 故障分析及性能优化系列之三：jstat 命令的使用及 VM Thread 分析](https://www.javatang.com/archives/2017/10/20/12131956.html)

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

### [JVM 故障分析及性能优化系列之四：jstack 生成的 Thread Dump 日志线程状态](https://www.javatang.com/archives/2017/10/25/36441958.html)

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

#### 系统线程状态 (Native Thread Status)





## 参考文章
> [JVM 故障分析及性能优化系列之一：使用 jstack 定位线程堆栈信息](https://www.javatang.com/archives/2017/10/19/33151873.html)
> [记一次诡异的频繁 Full GC](https://www.jianshu.com/p/f3fd1664f1ee)
> [Java Hotspot G1 GC 的一些关键技术](https://tech.meituan.com/2016/09/23/g1.html)
> [JVM系列-读懂 GC 日志](https://juejin.cn/post/6854573218968109070)
