# G1垃圾收集器

## G1垃圾收集器简介

G1收集器是一款在server端运行的垃圾收集器，专门针对拥有多核处理器和大内存的机器，它在实现高吞吐量的同时，也能够满足低延迟的目标。

相比于之前的一些串行/并行收集器，G1具有如下的一些特点：
- G1的设计目标是 **收集尽可能多的垃圾(Garbage First)** 所以它不会等到内存耗尽(ParNew、Serial等)或者内存即将耗尽(CMS)的时候才开始收集垃圾。G1在内部实现了启发式算法来找出具有高收集收益的Region进行垃圾回收。
- G1采用了内存分区的思路，把堆内存划分为一个个大小相等的区域(Region)，回收时就以Region作为单位回收。
- G1依然保留了分代的概念，但是G1并不需要其他垃圾收集器配合就能管理整个堆空间。而且G1里的分代概念只是逻辑上的，不像之前的一些收集器新生代老年代的内存空间都是物理隔离的。
- G1建立了可预测的停顿模型，它能够让使用者明确指定在一个长度为M毫秒的时间范围内，消耗在垃圾收集上的时间不超过N毫秒。当然这个预测模型不是100%准确的，但确实可以达到一个非常高的准确率。
- G1的垃圾收集比较独特，可以分为Young GC和Mixed GC。Young GC和其他垃圾收集器的Young GC都差不多。但是在Mixed GC里，新生代和老年代可能会一起回收。


## G1中的一些重要概念


### Region的概念及其相关的内存分配

之前的一些串行/并行收集器都把连续的内存空间划分为新生代、老年代和永久代(Java8去掉了永久代，引入了MetaSpace)，这种划分的特点是，这些分区各自的内存在物理上是连续的，分区之间的内存是物理隔离的，有明显的界限。

**G1却采用了Region的概念，它把堆划分为一块块大小相等的区域(称之为Region)，这些Region在逻辑上是连续的，每块Region都会有唯一的分代标记(Eden,Survivor,Old)。在逻辑上，所有的Eden Region构成了Eden区，所有的Survivor Region构成了Survivor区，所有的Old Region构成了Old区**。

可以通过`-XX:G1HeapRegionSize`指定Region的大小，取值范围是1M-32M，需要是2的指数。如果不设定，G1则会根据Heap的大小自动计算出Region的大小。

#### G1动态调整堆的大小

G1可以通过`-Xmx/-Xms`指定堆空间的大小，但是堆空间不是一次性分配完毕的，而是可以动态调整的。当发生年轻代收集和混合收集时，通过计算GC与应用的耗费时间比，自动调整堆空间的大小。如果GC频率太高，可以通过增加堆尺寸来减少GC频率，相应的GC占用的时间也随之降低。当空间不足时，G1会首先尝试增加堆空间，如果扩容失败才会发起Full GC。Full GC结束后也会重新调整堆空间。

#### 分代

G1使用了Region之后，分代也边的和其他的串行/并行不一样了，比如前面提到的新生代和老年代的内存变得不是物理隔离了。G1会给每块Region打一个唯一的分代的标记。

G1提供设置固定的新生代参数大小的参数(`-XX:NewRatio`、`-Xmn`)，但是一般不推荐使用，因为设定了固定新生代大小会使得暂停目标参数失去意义，因为G1会通过调整新生代的大小来实现用户设置的暂停目标。
一般来说，使用G1时推荐`-XX:G1NewSizePercent`(新生代初始大小在堆中的占比，默认5%)以及`-XX:G1MaxNewSizePercent`(新生代最大大小在堆中的占比，默认60%)来控制新生代的大小，G1会在这个范围内动态的调整新生代的大小(通过用户设置的`-XX:MaxGCPauseMillis`(默认200ms)、需要扩容的大小以及RSet里的数据计算得到新生代的大小)。

#### TLAB

G1默认开启了TLAB优化，每个线程都会被分配某个Region用于线程本地的内存分配，从而减少内存分配在并发的环境下同步数据的性能损耗，提升内存分配的效率。

应用线程所分配的对象大部分都会在Eden区里(巨型对象或者分配失败时除外)，因此TLAB所占有的Region肯定是被打了Eden标记的。进行垃圾收集时，每个GC线程也可以占用一个Region(称之为GCLAB)来转移对象，每次回收都会将对象复制到Survivor区或者Old区。然对于从Eden/Survivor晋升到Survivor/老年代的对象，同样会有GC独占的本地缓冲区进行操作，该部分称为晋升本地缓冲区(PLAB)。

#### G1对大对象的特殊处理

G1里有巨型对象(Humongous Object)的概念，只要这个对象超过一个Region的一半大小，就称之为巨型对象。
G1在内存分配的时候会对这种对象做一些特殊处理：
- 这种巨型对象不会在线程的TLAB里分配，而是会单独找一些Region进行分配。有可能一个Region无法容纳这个巨型对象，G1就会找几个连续的Region用于分配这个对象，这些Region称为Humongous Region。
- 这些巨型对象可以在Young GC的时候被回收

由于巨型对象无法享受线程本地分配缓冲带来的优化，而且在堆中确定一片连续的空间(找到或者创建多个连续的空闲Region)需要扫描整个堆，所以分配巨型对象的时候，性能损耗会比较大。

### G1里的GC

G1里的GC有两种：Young GC和Mixed GC。其中Young GC只收集新生代，Mixed GC会同时收集新生代+老年代。G1里没有Full GC，如果使用里G1的时候发生里Full GC(就是Mixed GC回收Region的速度无法跟上用户线程分配内存的速度，导致没有空的Region进行分配)，就会触发G1的兜底策略，转而使用Serial Old进行Full GC，这个停顿时间会非常的长。

G1里GC其实差不多分为两部分，这两部分可以相对独立的执行：
- Global Concurrent Marking，全局并发标记
- Evacuation，拷贝存活对象

#### Young GC

在Eden空间占满之后，就会触发一次STW式的Young GC，这时候会回收所有的Eden区和Survivor区的Region。其中Eden区的存活对象被拷贝到Survivor区(这个拷贝过程是多个GC线程并发执行的)，Survivor区存活的对象则会开始晋升的流程，可能会被移动到PLAB、新的Survivor Region或者老年代Region等，而原先的这些Region就会被整体回收。用于控制Young GC开销的手段是动态的改变新生代Region的数量。

Young GC的时候，还需要维护对象的年龄，判断对象晋升等等。JVM此时会收集晋升对象的大小总和、对象的年龄信息维护在年龄表中，然后根据年龄表、Survivor Regions的总大小，Survivor区被占用的容量比例(`-XX:TargetSurvivorRatio`(默认50%))、用户设定的年龄阈值(`-XX:MaxTenuringThreshold`(默认15))这些参数计算出一个新的年龄阈值，超过这个新的年龄阈值的对象都会被晋升到老年代。

#### Mixed GC

当老年代所有Region在堆中占用的空间大小比例超过一定的阈值(`-XX:InitiatingHeapOccupancyPercent`(默认45%))时，G1就会启动一次Mixed GC。这时候G1会选定所有新生代的Region，然后开始进行Global Concurrent Marking统计出各个老年代的收集收益，然后在Evacuation阶段根据用户的暂停时间目标选定一部分收集收益高的老年代Region。最后开始对所有的新生代Region和这部分被选定的老年代Region进行回收。


### Collection Set

上文中提到的，在Evacuation阶段之前，会选定一些Region进行后续的回收，这些选定的Region被存储在某个集合当中，这个集合就叫做Collection Set，简称CSet。CSet中可能存放着各个分代的Region，垃圾回收完成之后这里的所有Region都会被释放，其中存活的对象会在GC中被移动(复制)到空闲Region中。因此无论是新生代还是老年代，其收集算法都是一致的，就是复制-清除算法。


### Remembered Set

CMS采用了Card Table来保存老年代对新生代的引用关系，极大的提高了扫描的效率。G1里也参考了这个空间换时间的思路，不过采用了一个新的抽象概念，叫做Remembered Set，简称RSet。RSet是比Card Table更抽象的一个概念，而Card Table可以是RSet的一种实现方式。

每个Region都有一个对应的RSet，RSet里保存了其他Region对本Rset所属Region内对象的引用关系，属于points-into结构(谁引用了我的对象)。而Card Table则是points-out结构(我引用了谁的对象)的结构，每个Card覆盖一定范围的Heap(一般是512Bytes)。

G1里RSet的实现是这样的：首先每个Region被划分为很多个Card，每个Card是512Bytes，这个Card里可能存在一组对象，但是只要这个对象被引用了，整个Card就会被记录，称之为Dirty Card。在扫描的时候，会找出所有的Dirty Card，然后对里面的对象进行扫描。实际上RSet的实现就是一堆Card的HashSet的集合，因为每个GC线程都有一个HashSet。

然后我们来看一个问题：
> Question: RSet里保存里所有的对象引用关系么，比如old引用young，young引用old，old引用old，young引用young，这四种对象引用关系都会保存么？

> Answer: 实际上RSet里只保存里old引用young，old引用old的引用关系。

- 当一个Region确定要扫描的时候，那么不需要RSet也能得到引用关系，所以引用了本分区内的对象这种引用关系实际上是不需要记录的。
- 因为G1每次GC都会全盘扫描新生代，所以young引用young，或者young引用old根本就不需要记录，因为这些关系在扫描新生代Region的时候可以获得。


### SATB

SATB是用来保证并发GC正确性的算法，全称Snapshot-at-the-Beginning。GC的正确性指的是保证回收的都是垃圾，活着的对象不会被回收。如果整个标记过程都是STW的话，那么GC的正确性是肯定可以保证的。但是在CMS、G1这种垃圾收集器里，总有一些标记过程是并发的，这个过程中GC线程一边标记，用户线程一边修改引用，就很难保证标记的正确性。SATB就是用来保证并发标记时GC的正确性的。

SATB的思路很简单，在GC开始的时候，对内存里的对象图做一个逻辑快照。在GC Roots Tracing并发标记的过程发现的对象，只要在快照中是活的，那么在整个GC过程中就是被认定是活的，即使该对象的引用稍后被用户线程修改或者删除导致这个对象实际上是死的。同时，新分配的对象也被认为是活的，除此之外的其他对象都是死的。这样SATB就保证里真正存活的对象不会被回收，但是同时也会造成某些死的对象逃过这次回收，从而成为浮动垃圾(Float Grabage)。


## G1的工作流程

前面有提到，G1在垃圾收集的时候，可以分为两部分：
- Global Concurrent Marking，全局并发标记
- Evacuation，拷贝存活对象

### 全局并发标记

全局并发标记是基于SATB形式的并发标记，分为以下几个阶段。

#### Initial Marking(STW)

初始标记，过程中是STW的。扫描GC Roots，对GC Roots直接可达的对象进行标记，并将它们压入扫描栈(Marking Stack)中等待后续的扫描。G1使用外部的bitmap来记录mark信息，而不是使用对象头的mark word里的mark bit。G1借用Young GC的暂停(STW)顺便启动初始标记，这样就可以节省一次STW暂停的时间。

#### Root Region Scanning

Root Region的扫描从Survivor区的对象触发，标记被引用到的老年代中的对象，并把他们压入扫描栈中等待后续的扫描。这个过程是并发执行的。Root Region Scanning必须在Young GC开始前完成。


#### Concurrent Marking

并发标记的阶段，GC线程和用户线程一起工作。这个阶段GC线程会从扫描栈中取出之前压入的对象，然后的递归的处理引用，直到扫描栈清空。这个过程中还会扫描SATB writer barrier所记录的引用。这个过程可以被Young GC打断。

#### Remarking(STW)

最终标记阶段，也是STW的。在完成并发标记之后，每个Java线程还会有一些剩下的SATB write barrier记录的引用尚未处理。这个阶段就负责把剩下的引用处理完。同时也进行弱引用的处理。这个暂停与CMS的Remark有一个本质上的区别，那就是这个暂停只需要扫描SATB buffer，而CMS的Remark需要重新扫描mod-union table里的Dirty Card外加整个根集合，而此时整个Young gen（不管对象死活）都会被当作根集合的一部分，因而CMS Remark有可能会非常慢。

#### Cleanup(STW)

清点和充值标记状态，这个过程也是STW的。在marking bitmap里统计每个region被标记为活的对象有多少。这个阶段如果发现完全没有活对象的region就会将其整体回收到可分配region列表中。


### 拷贝存活对象

Evacuation阶段也是STW的，这个阶段把一部分Region里的活对象拷贝到空Region里，然后回收旧的Region。
这个阶段可以自由选择(实际上是一些参数的组合计算结果)任意多个Region放入CSet中。CSet选定完成之后，就开始多个GC线程并发回收。

### G1为何是低延迟的垃圾收集器

从上面可以看到，G1实际上只有很少的阶段是GC线程与用户线程并发执行的，而“拷贝对象”(Evacuation)这个很耗时的动作却不是并发而是完全暂停的，那么G1为什么还可以叫做低延迟的垃圾收集器呢？
因为G1不是每次都Evacuation所有的有活对象的Region的，而是只选择少量的收集收益高的Region进行Evacuation，这个过程中的开销在一定范围内是可控的，每次Evacuation所花费的时间可能和每次Young GC的时间差不多。


## G1是如何应对一些GC常见的场景的

在垃圾收集里有几个比较常见的场景，或者说问题：
- Young GC的时候是否需要扫描整个老年代？
- 收集老年代的时候是否需要扫描整个新生代？


### Young GC的时候是否需要扫描整个老年代

JVM里判断对象是否存活使用的是可达性分析算法，也就是从一些GC Roots节点出发可达的对象就是可以存活的对象。
当进行一次Young GC的时候，垃圾收集器的回收目标是回收新生代的对象，但是GC Roots里的一些对象可能是老年代的，这时候就会扫描到老年代的对象。而且扫描到老年代的对象也不能不继续往下扫描，因为可能老年代对象会引用新生代对象。所以为了保证GC的正确性，看起来只能把所有的老年代对象都扫描一边了。这很明显是非常非常耗时的操作！

在前面的章节中有提到，在CMS里采用了Card Table的数据结构来保存老年代对新生代的引用关系，实际上就是为了处理这个问题而实现的，Card Table极大的提高了扫描的效率，可以参考[CMS垃圾收集器中的Card Table章节](concurrent_mark_sweep_collector.md)。
G1参考了这个思路，使用了RSet。

我们来看下RSet为什么可以解决这个问题：在进行Young GC的时候，只需要选定新生代的Region的RSet作为根集，这些RSet里记录里这些Region里的新生代对象被老年代对象引用的记录，所以只需要去扫描这些特定的老年代对象就可以了，不需要扫描整个老年代，这样扫描效率就很高。

### 收集老年代的时候是否需要扫描整个新生代

从上面G1的GC章节我们可以看到，在G1里，每一次GC都是要对所有的新生代进行回收的，然后G1在Mixed GC的时候会选定所有新生代的Region，所以这时候基本上就是扫描整个新生代，对的，**G1就是每次GC都扫描所有的新生代对象**。


## G1的一些最佳实践

### 不要显式的设置新生代的大小

以下几个参数不要设置：
- `-Xmn`
- `-XX:NewRatio`
- `-XX:SurvivorRatio`

这几个参数都是会直接固定新生代大小的，设置了这些参数之后，`-XX:G1NewSizePercent`、`-XX:G1MaxNewSizePercent`、`-XX:MaxGCPauseMillis`这三个参数会失效，G1的停顿预测模型就完全失效了，这会导致G1 GC的性能大幅度降低。

### 响应时间大小的设置

G1支持`-XX:MaxGCPauseMillis`来设置用户期望的GC暂停时间，但是这个暂停时间不是越小越好的，而是要考虑设置90%以上时间都能达到目标的值。G1的Evacuation Pause在几十到一百甚至两百毫秒都很正常。如果把`-XX:MaxGCPauseMillis`设得太低，会导致G1的Mixed GC回收Region的速度跟不上用户线程使用Region的速度，就容易导致垃圾堆积，反而更容易引发Full GC而降低性能。

### Evacuation Failed

待补充


## 参考资料
- https://hllvm-group.iteye.com/group/topic/44381
- https://hllvm-group.iteye.com/group/topic/21468#post-272070
- https://ericfu.me/g1-garbage-collector/
- https://www.oracle.com/technetwork/tutorials/tutorials-1876574.html

