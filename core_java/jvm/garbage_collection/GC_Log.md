# 理解GC Log
## ParNew GC Log
## CMS GC Log
## G1 GC Log




基本上都是这种格式：回收前区域占用的大小->回收后区域占用的大小（区域设置的大小），占用的时间


## 一次晋升失败的Full GC

> 2019-11-08T11:26:47.189+0800: 676324.767: [GC (Allocation Failure) 2019-11-08T11:26:47.189+0800: 676324.767: [ParNew (promotion failed): 1723732K->1723732K(1887488K), 3.3293587 secs]2019-11-08T11:26:50.518+0800: 676328.096: [CMS: 1569239K->1730063K(2097152K), 4.3522962 secs] 2717978K->1730063K(3984640K), [Metaspace: 92944K->92944K(1132544K)], 7.6819115 secs] [Times: user=9.20 sys=0.19, real=7.68 secs] 

时间戳：
> 2019-11-08T11:26:47.189+0800: 676324.767: [GC (Allocation Failure) 2019-11-08T11:26:47.189+0800: 676324.767:  

GC类型以及垃圾收集器：
> [ParNew (promotion failed): 1723732K->1723732K(1887488K), 3.3293587 secs]
ParNew是垃圾收集器，promotion failed表示担保失败，担保失败发生在新生代Survivor空间不足，且老年代也空间不足的情况下。(为什么担保失败？)
1723732K->1723732K(1887488K)，ParNew回收前和回收后的空间都1723732K，新生代的总大小1887488K
这次新生代的垃圾回收花费3.3293587 secs

老年代区域以及堆的垃圾回收情况：
> 2019-11-08T11:26:50.518+0800: 676328.096: [CMS: 1569239K->1730063K(2097152K), 4.3522962 secs] 2717978K->1730063K(3984640K)
时间戳，CMS是垃圾收集器，1569239K->1730063K(2097152K)这是老年代回收前大小->回收后大小(老年代总大小)，花费时间 4.3522962 secs
 2717978K->1730063K(3984640K)这是回收前堆大小->回收后堆大小(堆总大小)

永久代的垃圾回收：
[Metaspace: 92944K->92944K(1132544K)], 7.6819115 secs] 



## 一次正常的CMS GC日志

初始标记，Stop The World，标记GC Roots直接可达的对象：
> 2019-11-11T13:01:44.070+0800: 263624.318: [GC (CMS Initial Mark) [1 CMS-initial-mark: 1469105K(2097152K)] 1560639K(3984640K), 0.0302904 secs] [Times: user=0.08 sys=0.00, real=0.03 secs] 



并发标记，
> 2019-11-11T13:01:44.100+0800: 263624.348: [CMS-concurrent-mark-start]
> 2019-11-11T13:01:44.423+0800: 263624.671: [CMS-concurrent-mark: 0.323/0.323 secs] [Times: user=0.40 sys=0.01, real=0.32 secs] 

并发预处理
> 2019-11-11T13:01:44.423+0800: 263624.671: [CMS-concurrent-preclean-start]
> 2019-11-11T13:01:44.432+0800: 263624.680: [CMS-concurrent-preclean: 0.008/0.008 secs] [Times: user=0.01 sys=0.00, real=0.01 secs] 


并发可中断预处理
> 2019-11-11T13:01:44.432+0800: 263624.680: [CMS-concurrent-abortable-preclean-start]
> CMS: abort preclean due to time 2019-11-11T13:01:49.667+0800: 263629.915: [CMS-concurrent-abortable-preclean: 5.231/5.235 secs] [Times: user=6.67 sys=0.33, real=5.24 secs] 

重新标记
> 2019-11-11T13:01:49.668+0800: 263629.916: [GC (CMS Final Remark) [YG occupancy: 395629 K (1887488 K)]》》，
> 2019-11-11T13:01:49.668+0800: 263629.916: [Rescan (parallel) , 0.1703436 secs]2019-11-11T13:01:49.838+0800: 263630.086: [weak refs processing, 0.0001367 secs]2019-11-11T13:01:49.838+0800: 263630.086: [class unloading, 0.0278322 secs]
> 2019-11-11T13:01:49.866+0800: 263630.114: [scrub symbol table, 0.0226982 secs]2019-11-11T13:01:49.889+0800: 263630.137: [scrub string table, 0.0013816 secs][1 CMS-remark: 1469105K(2097152K)] 1864735K(3984640K), 0.2226694 secs] [Times: user=0.39 sys=0.00, real=0.22 secs] 


并发清除
> 2019-11-11T13:01:49.892+0800: 263630.140: [CMS-concurrent-sweep-start]
> 2019-11-11T13:01:51.235+0800: 263631.484: [CMS-concurrent-sweep: 1.343/1.343 secs] [Times: user=1.76 sys=0.09, real=1.35 secs] 

重置阶段
> 2019-11-11T13:01:51.236+0800: 263631.484: [CMS-concurrent-reset-start]
> 2019-11-11T13:01:51.241+0800: 263631.489: [CMS-concurrent-reset: 0.005/0.005 secs] [Times: user=0.01 sys=0.00, real=0.00 secs] 



