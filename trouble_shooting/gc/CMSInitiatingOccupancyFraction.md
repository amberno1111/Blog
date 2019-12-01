# -XX:CMSInitiatingOccupancyFraction设置过高引起的Concurrent Mode Failure

线上出现的CMS Concurrent Mode Failure GC日志如下：
```java
2018-10-17T14:51:36.837+0800: 446863.254: [GC (Allocation Failure) 2018-10-17T14:51:36.837+0800: 446863.254: [ParNew (promotion failed): 613440K->597570K(613440K), 0.3319268 secs]2018-10-17T14:51:37.169+0800: 446863.586: [CMS2018-10-17T14:51:40.644+0800: 446867.062: [CMS-concurrent-sweep: 9.287/12.270 secs] [Times: user=31.94 sys=0.53, real=12.27 secs]
 (concurrent mode failure): 5077413K->1114319K(8755648K), 8.6483292 secs] 5676938K->1114319K(9369088K), [Metaspace: 128781K->128781K(1171456K)], 8.9825502 secs] [Times: user=9.16 sys=0.01, real=8.99 secs]
```

eden区不足，触发Minor GC（ParNew GC）,Minor gc后由于Survivor space不足，需要Tenured到old gen,但是此时old gen也不足，进而触发Major GC（cms gc）。解决这个问题的思路是提前触发CMS gc（-XX:CMSInitiatingOccupancyFraction=70 -XX:+UseCMSInitiatingOccupancyOnly）


