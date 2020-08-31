---
title: "Java进程容器内存监控指标"
date: 2020-08-21T19:52:30+08:00
draft: false
---

# 以container_memory_usage_bytes作为memory指标
过去的一段时间内，xxx服务在prometheus上面用来监控内存的指标是以容器内存使用量（率）来衡量的，如下：
```
container_memory_usage_bytes{pod=~"xxx.*-[^-]*-[^-]*$", image!="", container!="POD"}*100/container_spec_memory_limit_bytes{pod=~"xxx.*-[^-]*-[^-]*$", image!="", container!="POD"} > 80.0
```
xxx服务在kubernetes中的资源配置情况如下，

```
resources:
  limits:
    cpu: "2"
    memory: "4Gi"
  requests:
    cpu: "100m"
    memory: "512Mi"
```
在这样的配置下，上面监控rule的语义就是，当容器内使用内存量达到容器最大内存（4G）的80%时发出告警。



再来看下container_memory_usage_bytes如何定义，在cadvisor上面一个issue中提到：

container_memory_usage_bytes == container_memory_rss + container_memory_cache + container_memory_swap + kernel memory(issue中提到这里的kernel memory还未被暴露为metrics).


看到rss, swap, cache这些指标很自然的就想到top，在top中也有一个指标叫做used，其定义为 ：“USED - simply the sum of RES and SWAP”, https://man7.org/linux/man-pages/man1/top.1.html

# 以container_memory_usage_bytes作为java容器内存指标的问题


## 频繁告警
某刻xxx服务中某个pod中top指标如下
```
top - 05:35:43 up 7 days, 21:32,  0 users,  load average: 11.99, 11.35, 8.50
Tasks:  10 total,   1 running,   9 sleeping,   0 stopped,   0 zombie
%Cpu(s):  6.7 us,  6.5 sy,  0.0 ni, 85.9 id,  0.3 wa,  0.0 hi,  0.7 si,  0.0 st
KiB Mem : 65960736 total,  7194880 free, 42443524 used, 16322332 buff/cache
KiB Swap:        0 total,        0 free,        0 used. 19621872 avail Mem 

   PID %MEM    RES    VIRT    SHR   SWAP   CODE COMMAND                                               
     9  6.0   3.8g 9833416  16400      0      4 java                                                  
107812  0.0   2140   59728   1504      0     96 top                                                   
 69248  0.0   2152   59716   1504      0     96 top                                                   
 57442  0.0   2152   59712   1504      0     96 top                                                   
 57423  0.0   2992   16160   1624      0    888 bash                                                  
 68879  0.0   2928   16160   1564      0    888 bash                                                  
107795  0.0   2936   16160   1568      0    888 bash                                                  
 68899  0.0   3000   16156   1628      0    888 bash                                                  
     1  0.0   2548   16020   1328      0    888 bash                                                  
107946  0.0    476    7876    280      0     24 sleep
```
可以看到我们的9号java进程，RES占用有3.8GB，而前面说过，USED=RES+SWAP，这里还没有发生SWAP，所以USED等于RES占用了3.8GB。在我们的告警规则中，配置了当容器内存达到最大值4GB的80%时发生告警，即容器内USED > 3.2GB时会告警，容器内最重要java进程自己的USED就远大于这个阈值，所以会频繁告警。

那么，RES为3.8GB有什么问题呢？

## JVM指标
既然是java进程常驻内存很高，那么首先怀疑的就是jvm内存占用问题。

看一下堆内存，jmap -heap 9，发现很健康
```
Heap Configuration:
   MinHeapFreeRatio         = 40
   MaxHeapFreeRatio         = 70
   MaxHeapSize              = 3221225472 (3072.0MB)
   NewSize                  = 1073741824 (1024.0MB)
   MaxNewSize               = 1073741824 (1024.0MB)
   OldSize                  = 2147483648 (2048.0MB)
   NewRatio                 = 2
   SurvivorRatio            = 8
   MetaspaceSize            = 21807104 (20.796875MB)
   CompressedClassSpaceSize = 1073741824 (1024.0MB)
   MaxMetaspaceSize         = 17592186044415 MB
   G1HeapRegionSize         = 0 (0.0MB)


Heap Usage:
PS Young Generation
Eden Space:
   capacity = 859832320 (820.0MB)
   used     = 483407288 (461.0131149291992MB)
   free     = 376425032 (358.9868850708008MB)
   56.22111157673161% used
From Space:
   capacity = 106954752 (102.0MB)
   used     = 50262824 (47.934364318847656MB)
   free     = 56691928 (54.065635681152344MB)
   46.99447482239966% used
To Space:
   capacity = 106954752 (102.0MB)
   used     = 0 (0.0MB)
   free     = 106954752 (102.0MB)
   0.0% used
PS Old Generation
   capacity = 2147483648 (2048.0MB)
   used     = 143595616 (136.94345092773438MB)
   free     = 2003888032 (1911.0565490722656MB)
   6.68669193983078% used
```

不信邪手动gc一下，发现RES并没有什么变化。
```
jcmd 9 GC.run
```

到这里可以明白堆内内存没有什么问题，下一个怀疑对象是堆外内存。启动参数上加上-XX:NativeMemoryTracking=summary，然后jcmd 9 VM.native_memory summary scale=MB看一下发现依然很健康（虽然加上参数容器会重启，但下面数据也是在频繁告警的情况下获得的）


```
Native Memory Tracking:

Total: reserved=5261MB, committed=4124MB
-                 Java Heap (reserved=3072MB, committed=3072MB)
                            (mmap: reserved=3072MB, committed=3072MB) 
 
-                     Class (reserved=1180MB, committed=174MB)
                            (classes #25539)
                            (malloc=12MB #47472) 
                            (mmap: reserved=1168MB, committed=162MB) 
 
-                    Thread (reserved=535MB, committed=535MB)
                            (thread #531)
                            (stack: reserved=532MB, committed=532MB)
                            (malloc=2MB #2660) 
                            (arena=1MB #1057)
 
-                      Code (reserved=274MB, committed=144MB)
                            (malloc=30MB #29634) 
                            (mmap: reserved=244MB, committed=113MB) 
 
-                        GC (reserved=122MB, committed=122MB)
                            (malloc=10MB #615) 
                            (mmap: reserved=112MB, committed=112MB) 
 
-                  Compiler (reserved=1MB, committed=1MB)
                            (malloc=1MB #3104) 
 
-                  Internal (reserved=40MB, committed=40MB)
                            (malloc=40MB #38278) 
 
-                    Symbol (reserved=30MB, committed=30MB)
                            (malloc=26MB #267011) 
                            (arena=4MB #1)
 
-    Native Memory Tracking (reserved=6MB, committed=6MB)
                            (tracking overhead=6MB)

```

与前文的堆内内存相比较，容易发现这里所指的committed即capacity，那么offsetHeap的capacity=174+535+144+122+1+40+30+6 MB = 1052 MB，注意这里仅为committed(或者可以理解成capacity)，实际使用肯定是要小于这个值的，那么再考虑到前面heap的实际使用情况461+47+136 MB = 644 MB， 以上两者(heap+offsetHeap)之和肯定是要远远小于3.8g的。

这里可以根据到这里可以发现jvm很健康，内存占用是比较低的。那么，top命令中的RES究竟是什么呢？



## 重新考量top
写到这里啰嗦了一大堆，前面的逻辑简单概括一下：

使用了cadvisor里面的container_memory_usage_bytes作为指标参数，而这个指标和top强相关，都是RES+SWAP+一些其他东西（占用很小）；
在我们应用中没有发生swap，所以造成告警的原因主要是java进程的RES很大；
java进程内部堆内内存和堆外内存很健康，没有看到大量占用内存的迹象；
不得不说top是一个很强大的工具，但是他也有很多的诟病，尤其是对于java进程来说，下面摘录一些国外开发者的抱怨：

<em>"It really annoys me that there's no good tool for measuring memory usage on Linux. There are tools, like 'top', but they often cause more harm than good - most people don't even know what the fields really mean and only few people can interpret them correctly. Mind you, even I'm not sure I can, and in fact I sometimes doubt such person even exists. The problem is, even intepreting the numbers may not give the answer. Measuring memory usage on Linux is voodoo magic."</em>  
-- from https://blogs.kde.org/node/1445

<em>"This has been a long-standing complaint with Java, but it's largely meaningless...With a Java program, it's far more important to pay attention to what's happening in the heap. The total amount of space consumed is important, and there are some steps that you can take to reduce that. More important is the amount of time that you spend in garbage collection, and which parts of the heap are getting collected."</em>  
-- from https://stackoverflow.com/questions/561245/virtual-memory-usage-from-java-under-linux-too-much-memory-used

最后的观点就是，对于java进程来说，我们的关注点不应该是RES，而应该将目光转向heap。查阅很多资料后，能找到的关于这个现象下RES高的最贴切的解释是下面的一段话：

 

<b>When is Resident Set Size Important?</b>

<em>Resident Set size is that portion of the virtual memory space that is actually in RAM. If your RSS grows to be a significant portion of your total physical memory, it might be time to start worrying. If your RSS grows to take up all your physical memory, and your system starts swapping, it's well past time to start worrying.

But RSS is also misleading, especially on a lightly loaded machine. The operating system doesn't expend a lot of effort to reclaiming the pages used by a process. There's little benefit to be gained by doing so, and the potential for an expensive page fault if the process touches the page in the future. As a result, the RSS statistic may include lots of pages that aren't in active use.</em>



根据上面这段话，可以看出只有在发生了swap的时候，RES（上文中用RSS表示，二者意义相同）很高才会变得重要，这个时候要考虑是不是有memory leaking这种事情或者其他的情况。在一些轻负载并且尚未发生swap的场景下，高RES占用并不能说明什么问题，可能只是操作系统在reclaiming pages和效率之间的一个权衡，RES中统计的很多内存可能并不是真正在使用。

# 结论
 放弃容器级别的metrics转向jvm级别