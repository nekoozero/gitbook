# TLAB解读

> 2020/3/12

## 问题

虚拟机创建对象、分配内存是很频繁的一件事，那么，在多线程运行的情况下，怎么保证虚拟机在堆中给对象划分内存时候的线程安全？

其实虚拟机有两种解决方案：

1. 对虚拟机划分内存的这个动作进行**同步操作**，可能通过`CAS`和失败重试机制来保证线程安全。（不管什么同步方式，太过频繁都会影响分配内存的效率）
2. 使用`TLAB(Thread Local Allocation Buffer)`。

虚拟机并不是只采用一种方案，而是在具体的情况下使用某一个，或者两者结合使用。

## TLAB

> 每个线程在Java堆中预先分配一小块内存，然后再给对象分配内存的时候，直接在自己这块”私有”内存中分配，当这部分区域用完之后，再分配新的”私有”内存。

这块内存就是`TLAB`，是从堆中的`eden`划分出来的一块专用区域，在`TLAB`功能启用的情况下，线程初始化时，虚拟机会为每个线程分配一块`TLAB`空间，只给当前线程使用，线程如果需要给对象分配空间，就在自己的`TLAB`上分配，减少竞争的情况，提升效率。

**这一块内存可以看作是线程独享的，但是只是在“分配”这个动作上独享的，至于在读取、垃圾回收等动作上都是线程共享的，而且在使用上也没什么区别。**也就是说其他线程是可以访问到别的线程的`TLAB`，但是不会在上面分配自己的内存。

在`TLAB`分配之后，并不影响对象的移动和回收，也就是说，虽然对象刚开始可能通过`TLAB`分配内存，存放在`Eden`区，但是还是会被垃圾回收或者被移到`Survivor Space`、`Old Gen`等。

因为`TLAB`本身并不是很大，必然存在一些大对象无法在`TLAB`直接分配，这种情况下，对象还是可能在`eden`或者老年代进行分配，但这种分配就需要同步控制了。所以说：**小的对象比大的对象分配起来更加高效。**

### 带来的问题

`TLAB`并不是很大，有可能出现不够的情况，一般有两种方案：

1. 如果一个对象需要的空间大小超过`TLAB`中剩余的空间大小，则直接在堆内存中对该对象进行内存分配。
2. 如果一个对象需要的空间大小超过`TLAB`中剩余的空间大小，则废弃当前`TLAB`，重新申请`TLAB`空间再次进行内存分配。

以上两个方案各有利弊，如果采用方案1，那么就可能存在着一种极端情况，就是`TLAB`只剩下`1KB`，就会导致后续需要分配的大多数对象都需要在堆内存直接分配。

如果采用方案2，也有可能存在频繁废弃`TLAB`，频繁申请`TLAB`的情况，但是`TLAB`内存自己从堆中划分出来的过程确实可能存在冲突的，所以，`TLAB`的分配过程其实也是需要并发控制的。而频繁的`TLAB`分配就失去了使用`TLAB`的意义。

**为了解决这两个方案存在的问题，虚拟机定义了一个`refill_waste`的值，这个值可以翻译为“最大浪费空间”。**

当请求分配的内存大于`refill_waste`的时候，会选择在堆内存中分配。若小于`refill_waste`值，则会废弃当前`TLAB`，重新创建`TLAB`进行对象内存分配。

> `TLAB`总空间`100KB`，使用了`80KB`，剩余`20KB`，设置的`refill_waste`的值为`25KB`，那么如果新对象的内存大于`25KB`，则直接堆内存分配，如果小于`25KB`，则会废弃掉之前的那个`TLAB`，重新分配一个`TLAB`空间，给新对象分配内存。

### 相关参数

`TLAB`功能是可以选择开启或者关闭的，可以通过设置`-XX:+/-UseTLAB`参数来指定是否开启`TLAB`分配。

`TLAB`默认是`eden`区的1%，可以通过选项`-XX:TLABWasteTargetPercent`设置`TLAB`空间所占用`Eden`空间的百分比大小。

默认情况下，`TLAB`的空间会在运行时不断调整，使系统达到最佳的运行状态。如果需要禁用自动调整`TLAB`的大小，可以使用`-XX:-ResizeTLAB`来禁用，并且使用`-XX：TLABSize`来手工指定`TLAB`的大小。

`TLAB`的`refill_waste`也是可以调整的，默认值为64，即表示使用约为1/64空间大小作为`refill_waste`，使用参数：`-XX：TLABRefillWasteFraction`来调整。

如果想要观察`TLAB`的使用情况，可以使用参数`-XX+PringTLAB` 进行跟踪。

## 总结

`TLAB`的空间其实并不大，所以大对象还是可能需要在堆内存中直接分配。那么，对象的内存分配步骤就是先尝试`TLAB`分配，空间不足之后，再判断是否应该直接进入老年代，然后再确定是再`eden`分配还是在老年代分配。