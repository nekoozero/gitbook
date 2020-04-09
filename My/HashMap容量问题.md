---
title: HashMap容量问题
date: 2019-05-20 20:11:30
categories: 编程小得
summary: new HashMap(100)的容量到底是多少？
---

## 引言

之前也研究过HashMap的原理，经常有个疑问new HashMap(100)的容量是多少？

之前的观点：因为要放100个元素，所以容量x容量因子(0.75)>100, 但128x0.75=96<100，所以真正的容量应该为256。

## 一点点源码

以下的源码都是jdk1.8。

今天再度点进去HashMap看看，发现了新的收获，想验证自己的这个观点是否正确，先看看一部分源码吧

```java
/**
 * The default initial capacity - MUST be a power of two.
 默认初始容量 - 必须是2的幂。
 */
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
/**
 * The maximum capacity, used if a higher value is implicitly specified
 * by either of the constructors with arguments.
 * MUST be a power of two <= 1<<30.
 最大容量，如果具有参数的任一构造函数隐式指定，则使用最大容量。
 必须是2的幂<= 1 << 30。
 */
static final int MAXIMUM_CAPACITY = 1 << 30;

/**
 * The load factor used when none specified in constructor.
 在构造函数中未指定时使用的加载因子。
 */
static final float DEFAULT_LOAD_FACTOR = 0.75f;

```

这里写了一部分，还有一些就是有关于链表和红黑树的，那个就比较厉害了，不是这次讨论的范围内，我也没能力讨论。

再看一下其他变量：

```java
/**
 * The table, initialized on first use, and resized as
 * necessary. When allocated, length is always a power of two.
 * (We also tolerate length zero in some operations to allow
 * bootstrapping mechanics that are currently not needed.)
 该表在首次使用时初始化，并根据需要调整大小。分配时，长度始终是2的幂。
 （我们还在一些操作中容忍长度为零，以允许当前不需要的自举机制。）
 */
transient Node<K,V>[] table;
/**
 * Holds cached entrySet(). Note that AbstractMap fields are used
 * for keySet() and values().
 保存缓存的entrySet（）。请注意，AbstractMap字段用于keySet（）和values（）。
 */
transient Set<Map.Entry<K,V>> entrySet;

/**
 * The number of key-value mappings contained in this map.
 此映射中包含的键 - 值映射的数量
 */
transient int size;

/**
 * The number of times this HashMap has been structurally modified
 * Structural modifications are those that change the number of mappings in
 * the HashMap or otherwise modify its internal structure (e.g.,
 * rehash).  This field is used to make iterators on Collection-views of
 * the HashMap fail-fast.  (See ConcurrentModificationException).
 此HashMap已被结构修改的次数。结构修改是那些改变HashMap中映射数量或以其他方式修改其内部结构（例如，重新散列）的修改。此字段用于在HashMap的Collection-views上快速生成迭代器。 （请参阅ConcurrentModificationException）。
 */
transient int modCount;

/**
 * The next size value at which to resize (capacity * load factor).
 要调整大小的下一个大小值（容量*加载因子）。
 *
 * @serial
 */
 // (The javadoc description is true upon serialization.
 // Additionally, if the table array has not been allocated, this
 // field holds the initial array capacity, or zero signifying
 // DEFAULT_INITIAL_CAPACITY.)
int threshold;

/**
 * The load factor for the hash table.
 *哈希表的加载因子。
 * @serial
 */
final float loadFactor;
```

主要看一下threshold这个变量，就是我们所说的阈值。我们都知道HashMap扩容的两个因素，容量和负载因子，

正如上面注释所说，`capacity * load factor`,阈值=容量x负载因子。

所以我想知道在new HashMap([arg])的时候这个threshold的值到底是多少，但是这个是变量的访问限定符是default，这个包外面是访问不到的，当然也没有get方法，这就很尴尬，思来想去，于是乎，突然想到了一个东西：反射。

## 具体理解

不多说了，开撸：

* HashMap()

```java
public class MyHashMap {
    public static void main(String args[]) throws NoSuchFieldException, SecurityException, IllegalArgumentException, IllegalAccessException {
    	Map  map = new HashMap(100);
    	int k =97;     //这边控制向HashMap中存放多少个值
    	for(int  i = 0;i<k;i++) {
    		map.put(i, i);
    	}
        System.out.println("实际个数："+map.size());
    	System.out.println("阈值："+getThr(map));
    }
    
    static int getThr(Map map) throws NoSuchFieldException, SecurityException, IllegalArgumentException, IllegalAccessException {
    	Class clazz = map.getClass();   //获得这个对象的Class对象
    	Field f = clazz.getDeclaredField("threshold");    //获得名为threshold的Field
    	f.setAccessible(true);   //设置为可访问的
    	return (int) f.get(map);  //获取值
    }
}
```

提一下,

1. getFields()：获得某个类的所有的公共（public）的字段，包括父类中的字段。 
2. getDeclaredFields()：获得某个类的所有声明的字段，即包括public、private和proteced，但是不包括父类的申明字段。

这两个方法都是获取的数组，当然也可以按名字单个获取。



先别急，从`Map  map = new HashMap();`开始测试，设置k=0，即不添加任何数据，先看结果：

```
实际个数：0
阈值：0
```

这就是说，有印象的同学都记得，如果不传入参数的话，HashMap的容量应该16,那么，阈值应该为12才对啊。

这时候就要打开源码看看这个构造函数了

```java
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}

```

没错，就是这么浅显易懂，赋值了一下负载因子，其他啥事没干，这也就解释了threshold为啥是0了，因为它是int类型的啊，没赋值初始值就是0。

好奇的我把k设为了1，就是想HashMap中添加了一个数据,神奇的事情发生：

```
实际个数：1
阈值：12
```

这下阈值出来了12，容量自然也就是16了。

总结一下，不给大小的HashMap，阈值为0，当有一个数据被加进来时，会发生扩容，容量会变成16，阈值会变成12了。这一切都发生在put()里面，看源码会知道他也是调用了putVal(),至于这个的源码嘛，现在只要知道它会去检查是否需要扩容，需要就会去扩容。当然，还有hash操作、检查链表是否要转成红黑树等等，等我以后有能力看懂再说吧。



把k设为13，即添加了13个数据，大于了阈值（等于的时候不会发生扩容），结果：

```
实际个数：13
阈值：24
```

容量自然为32，所以这一切都说的通了。

* HashMap(int)

其实HashMap构造函数一共有三个，无参的已经说过了，现在看看另外两个：

```java
public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);
    this.loadFactor = loadFactor;
    this.threshold = tableSizeFor(initialCapacity);   //这里设置了阈值
}

public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}
```

很明显了，当我们给定大小的时候，会去调用HashMap(int initialCapacity, float loadFactor) ，并且给定默认负载因子0.75。

再去兴致勃勃地看tableSizeFor函数：

```java
/**
* Returns a power of two size for the given target capacity.
返回给定目标容量的两个大小的幂。
*/
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}

```

我看到这个内心是崩溃的，虽然知道这些运算符的意思，但是就是一种“这都是什么代码啊？完全看不懂~~~”的感觉，唉，只能说“给大佬递茶”。我把这个代码单独拿出来测试了几下，大体的意思就是**返回大于或者等于给定参数的最小的那个2的次幂**，比如说cap = 0,返回1（2<sup>0</sup>）；cap=3,返回4，cap=50,返回64。至于为啥要这样写，人家肯定是有原因的，兼容更多东西吧。

所以当我们new HashMap(15)时，设置k=0：

```
实际个数：0
阈值：16
```

设置k=17，

```
实际个数：17
阈值：24
```

设置k=25

```
实际个数：25
阈值：48
```



## 总结

回到一开始的问题new HashMap(100)的时候，容量到底是多少？

现在说说我的理解，首先，容量不像阈值那样，在类中有明确的变量定义着，只有注释中的公式`capacity * load factor`,所以，在知道阈值的时候理论上上是可以算出容量的。所以抛开代码，按照数学逻辑new HashMap(100)的时候，阈值为128，容量为170.666无穷，真是个牵强的答案~~，而且容量必须是2的次幂，相信大家也是耳有所闻的。

但是，代码就是代码，再回到那个有参构造函数中，new HashMap(100)时，阈值为128，仅此而已，至于容量这个只是个概念上的东西，没有代码逻辑上相关的变量，更没有赋值语句了。

所以，要把重点放到扩容上，当我放了129个数据到这个HashMap中时，必然会发生扩容，扩容时的代码确实太难懂了，**我感觉理论上在第一次发生扩容的时候，才会有容量这个概念**，放到129个数据的时候阈值变成192（256x0.75）。

那是否可以这样理解？当new HashMap(int i)的时候，如果i正好为2的次幂，那阈值和容量都为i，如果不是的话，那么就会变成大于或者等于i的最小的2的次幂，发生扩容的时候，容量变为2倍，阈值变为容量的0.75倍，只有再次发生扩容的时候，阈值和容量同时变为2倍，0.75的关系也就会一直维持下去。



明天再研究看看吧！

