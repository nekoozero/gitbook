# ThreadLocal相关

>  时间：2020/8/12 -2020/8/13

之前使用了一次 ThreadLocal，想着自己能不能看看源码，看看到底底层是怎么做的。

印象中看过一些博客说是在 ThreadLocal 中会维持一个类似 Map 的东西，里面存储的就是线程 id 和对应线程 set 的值。

先大致看一下类结构：

可以看到确实存在一个 ThreadLocalMap 的静态类，该类中又有个内部静态类 Entry，先看一下 Entry的这个类：

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```

可以看到内部还是很简单的。（WeakReference 还是要留意的）

再看 ThreadLocalMap 类，几个关键的属性：

```java
/**
 * The table, resized as necessary.
 * table.length MUST always be a power of two.
 */
private Entry[] table;

/**
 * The number of entries in the table.
 */
private int size = 0;

/**
 * The next size value at which to resize.
 */
private int threshold; // Default to 0
```

隐约可以看到哈希表的那些苗头了，数组、阈值、大小等等。

那可以先做一个假设：

**这个 table （entry 数组）就是存放各个线程 set 值的，各个值最终会放到对应 entry 的 value 属性上面。**

看上去挺像一回事儿的。

## 具体看代码

首先先看 set 方法，看看到底做了什么？

```java
//ThreadLocal 类中
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
//ThreadLocal 类中
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;   //没想到吧!!
}
//ThreadLocal 类中
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}

```

有点小惊讶，map 竟然是从当前线程里面拿到的。

而 Thread 类中确实也有着对应的 threadLocals。

```java
//Thread 类中
/* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
ThreadLocal.ThreadLocalMap threadLocals = null;
```

可以看到，第一次 set 的时候，从当前线程去取这个 threadLocals 肯定为 null，那么接下来就会去走 createMap 方法了，该方法看名字就知道是创建一个 ThreadLocalMap 出来，参数为当前线程和set 的值。最后创建一个 ThreadLocalMap 实例并赋给当前线程的 threadLocals。此时 threadLocals 就不会为空了。看一下构造方法：

```java
//ThreadLocalMap 类
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    table = new Entry[INITIAL_CAPACITY];
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
    table[i] = new Entry(firstKey, firstValue);
    size = 1;
    setThreshold(INITIAL_CAPACITY);
}
```

他将当前的 ThreadLocal 对象和 set 的值放到传入到了 Entry 的构造方法去构造一个 Entry 出来。可以看到，我们 set 的值，确实是存到了 Entry 的 value 中，这是不会错的。

除此之外，还能看到 ThreadLocal 对象会参与到一个哈希计算中，目的其实就是找到该 Entry 在 table 的下标。因为最后 Entry 还是会放到 table 数组中来维护的。

那如果第二次 set，此时就不会去创建 ThreadLocalMap，而是直接用线程 Thread 里面 threadLocals.set 方法，并传入当前的 ThreadLocal 和 set 的值。

```java
//ThreadLocalMap 类
private void set(ThreadLocal<?> key, Object value) {

    // We don't use a fast path as with get() because it is at
    // least as common to use set() to create new entries as
    // it is to replace existing ones, in which case, a fast
    // path would fail more often than not.

    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);

    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) { //遍历 table 找到合适的位置i
        ThreadLocal<?> k = e.get();

        if (k == key) { //如果已存在就覆盖掉
            e.value = value;
            return;
        }

        if (k == null) { //没有的话就去新建并替换掉该 table 中该位置Entry
            replaceStaleEntry(key, value, i);
            return;
        }
    }
    //最后该位置重新给一个新的 entry
    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold) //清除一些 Slots 必要的话会去重新哈希
        rehash();
}
```

上面的内容还是挺多的，但是主要目的了解就行了，具体的哈希算法，包括存放位置 i 的计算等等还是比较复杂的。主要目的就是通过当前的ThreadLocal 对象去找到此次 set 的值的 Entry 应在位于 table 数组的位置。

接下来看一下 get 方法：

```java
//ThreadLocal 类
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}

//ThreadLocalMap 类
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
        return e;
    else
        return getEntryAfterMiss(key, i, e);
}
```

大致浏览一下其实也很清楚了，先看一下当前线程中 threadLocals 有没有值，有的话则根据当前的 ThreadLocal 对象去获取 Entry 并返回里面的 value。没有的话则返回 null。

上面总结的来看的话并回头看看那个假设：

**这个 ThreadLocalMap 其实是由 Thread 也就是当前线程来维护的，并不是 ThreadLocal 来维护的。ThreadLocal 最主要的用处是参与到了哈希位置的计算，也就是 set 的值的 Entry 放在 table 的哪个位置上。**

假设也只是对了一半：这个 table （entry 数组）不是存放**各个线程 set 值的**，存放的是**当前线程不同的 ThreadLocal 对象 set 的值**(下面有代码验证)。值确实是存放在了 Entry 对象的 value 中。

1. 为什么能做到线程间的数据隔离？

   因为数据都是存放在线程自己的 threadLocals (也就是ThreadLocalMap)中，get 也是从自己的 ThreadLocalMap 中 get。

2. 既然 set 的值是线程自己维护的，为什么要设计成哈希表的形式（table,entry 数组)？还要做复杂的哈希计算，重新哈希等等，直接用一个 Object 来存放不香吗？

   因为一个线程可能不止使用一个 ThreadLocal，当我有多个值想在线程的上下文使用的时候，就可以使用**多个 ThreadLocal**，即创建多个 ThreadLocal 对象来使用。此时就需要数组来存放这些数据了。**这些对象在代码中也就被视为 key**，用来计算哈希位置，存储和获取 value(set 的值)，这就是通常所说的表现行为类似 Map 结构，但实际上里面得数据结构和 HashMap 的 <k,v> 还是不大一样的。 

![threadLocal.jpg](http://www.qxnekoo.cn:8888/images/2020/08/13/threadLocal.jpg)

测试代码：

```java
   static ThreadLocal<String> threadLocal = new ThreadLocal<>();
   static ThreadLocal<Integer> threadLocal2 = new ThreadLocal<>(); 
   static class Demo2 extends Thread {
        @Override
        public void run() {
            threadLocal.set("threadLocal1对象");
            threadLocal2.set(99);
            Thread thread = Thread.currentThread();
            System.out.println(threadLocal.get());
        }
    }
    public static void main(String[] args) {
        Demo2 demo = new Demo2();
        Thread t1 = new Thread(demo);
        Thread main = Thread.currentThread();
        t1.start();
        System.out.println();
    }
```

在两个输出语句那边打上断点，然后就一目了然了：

![threadLocal_ex1.png](http://www.qxnekoo.cn:8888/images/2020/08/13/threadLocal_ex1.png)

![threadLocal_ex2.png](http://www.qxnekoo.cn:8888/images/2020/08/13/threadLocal_ex2.png)

![threadLocal_ex3.png](http://www.qxnekoo.cn:8888/images/2020/08/13/threadLocal_ex3.png)

上面基本就是大致流程和底层的一些原理了，其实内部还是有很多算法的……其实看源代码的话很简单就能看得出原理了，最值钱的还是那些个算法呀！

下面来看看这个 WeakReference! 这个 Entry 为什么要继承它称为一个弱引用？

```java
WeakReference<Apple> appleWeakReference = new WeakReference<>(apple);
```

 WeakReference 称为弱引用对象，而 apple 称为被弱引用对象，但被弱引用对象只是在这边是弱引用，可能其他地方会有强引用。

引用一共分为四种：强引用，软引用，弱引用，虚引用。引用强度依次递减。

在虚拟机部分也介绍过，这边就不一一赘述了，**只被弱引用关联的对象只能生存到下一次垃圾收集发生为止，不管当前内存是否足够，都会回收掉弱引用关联的对象。**

一个简单的示例：

```java
//继承WeakReference 称为一个弱引用对象 用来引用User 对象
public class Car extends WeakReference<User> {
    public Car(User referent) {
        super(referent);
    }
}

public class Car2 {
    User u;
    public Car2(User referent) {
        this.u = referent;
    }
}
//运行类
public static void main(String[] args) {
    User u = new User();
    User u2 = new User();
    Car car = new Car(u);
    Car2 car2 = new Car2(u2);
    u = null;
    u2 = null;
    System.gc();
    System.out.println(car.get());
    System.out.println(car2.u);
}
```

运行结果：

```
null
nekoo.User@1540e19d
```

可以看到在强引用 u 和 u2 被置为 null 后，一旦系统发生 gc 被弱引用的对象 u2 便被垃圾收集器回收了，而 u 还是可以获取到的。

如果这个对象是偶尔的使用，并且希望在使用时随时就能获取到，但又不想影响此对象的垃圾收集，那么应该用 WeakReference 来记住此对象。也就是说这个对象有自己的生命周期，但你不想介入这个对象的生命周期，这时就用弱引用。

1. 为什么要将 entry 继承WeakReference？为什么不使用强引用？

   先想一下如果使用强引用的话，entry 类该怎么写？

   ```java
   static class Entry {
       /** The value associated with this ThreadLocal. */
       Object value;
       
       ThreadLocal key;
       
       Entry(ThreadLocal<?> k, Object v) {
           key = k;
           value = v;
       }
   }
   ```

   entry.get() 也就变成了 entry.key 来使用了。这个不是关键，关键是此时 Entry 中有一个强引用（ThreadLocal key）引用着 ThreadLocal 对象，如果 ThreadLocal 我们不再使用了，但是线程的 threadLocals（ThreadLocalMap） 里的 entry 有个强引用，而这个 threadLocals 由于是线程的属性，它的生命周期和线程一样长，也就是说只要线程一直在，这个 ThreadLocal 对象就一直不会被垃圾收集。（值得一提的是，只要线程池不关闭，线程池中的核心线程是不会被结束的）。ThreadLocal 空间一直被占用着无法被释放，可能会导致内存泄漏。

   还有一方面是，一般 ThreadLocal 对象在声明的时候已经有强引用了，当我们想回收该对象，如果此时 entry 里还使用强引用的话，即使断开强引用，垃圾收集器也不会去清理整个对象的。

2. 为什么使用了弱引用，还存在内存泄漏的问题？

   entry 对象中，key 是通过弱引用引入的，但是 value 值本身是强引用引入。那就存在一种情况，那就是 key 由于弱引用的关系在 gc 时被回收了，但是 value 还是驻留在线程的 ThreadLocalMap 的 entry 中，导致有值无 key 的 无效entry，内存泄漏。

   虽然如此，但是源码中已经为我们做了很多防止工作：

   ```java
   private int expungeStaleEntry(int staleSlot) {
       Entry[] tab = table;
       int len = tab.length;
   
       // expunge entry at staleSlot
       tab[staleSlot].value = null;
       tab[staleSlot] = null;
       size--;
   
       // Rehash until we encounter null
       Entry e;
       int i;
       for (i = nextIndex(staleSlot, len);
            (e = tab[i]) != null;
            i = nextIndex(i, len)) {
           ThreadLocal<?> k = e.get();
           if (k == null) {
               e.value = null;
               tab[i] = null;
               size--;
           } else {
               int h = k.threadLocalHashCode & (len - 1);
               if (h != i) {
                   tab[i] = null;
   
                   // Unlike Knuth 6.4 Algorithm R, we must scan until
                   // null because multiple entries could have been stale.
                   while (tab[h] != null)
                       h = nextIndex(h, len);
                   tab[h] = e;
               }
           }
       }
       return i;
   }
   ```

   这个方法作用就是擦除某个下表的 Entry（置为 null，可以回收），同时检测整个 Entry[] 表中对 key 为 null 的 Entry 一并擦除，重新调整索引。

   该方法，在每次调用 ThreadLoca l的 get、set、remove 方法时都会执行，即 ThreadLocal 内部已经帮我们做了对 key 为 null 的 Entry 的清理工作。

3. 既然使用弱引用、强引用都会导致内存泄漏，为什么选择使用弱引用而不是强引用？

   弱引用相比强引用，我能想到的唯一好处就是如果用户忘写 remove，弱引用造成的代价会比强引用小的多，毕竟弱引用会自行清理一部分 key，同时 remove 和 get 方法里添加了清除无效 entry的逻辑，避免了引起严重的内存泄漏。

4. 声明 ThreadLocal 对象的时候已经给了一个强引用，那么什么时候垃圾收集器会清理这个 ThreadLocal 对象？

   我们手动可以断开这个强引用；obj作为句柄保持着对对象的引用，如果当前线程执行完毕，虚拟机栈被回收，栈中引用的obj句柄被回收，对象没有句柄保持引用，也会被回收。

## 最佳实践

目前我们使用多线程都是通过线程池管理的，对于核心线程数之内的线程都是长期驻留池内的。显式调用 remove，**一方面是防止内存泄漏，最为重要的是，不及时清除有可能导致严重的业务逻辑问题，产生线上故障（使用了上次未清除的值）。**出现异常了也要在 finally 中清除。

















