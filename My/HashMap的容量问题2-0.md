---
title: HashMap的容量问题2.0
date: 2019-05-21 14:34:53
categories: 编程小得
summary: table这个东西
---

## 引言

昨天说`容量不像阈值那样，在类中有明确的变量定义着`，今天可能就好像被啪啪打脸了，因为今天在看源码的时候发现了这东西，table，实际上在昨天的代码中也列举出来了，不过没有怎么注意这个东西，今天再次研究了它。因为发现在扩容的时候，这个东西无处不在。

## 代码

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
```

再看看Node：

```java
/**
     * Basic hash bin node, used for most entries.  (See below for
     * TreeNode subclass, and in LinkedHashMap for its Entry subclass.)
     基本哈希bin节点，用于大多数条目。
（参见下面的TreeNode子类，以及LinkedHashMap中的Entry子类。）
     */
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;

    Node(int hash, K key, V value, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }

    public final K getKey()        { return key; }
    public final V getValue()      { return value; }
    public final String toString() { return key + "=" + value; }

    public final int hashCode() {
        return Objects.hashCode(key) ^ Objects.hashCode(value);
    }

    public final V setValue(V newValue) {
        V oldValue = value;
        value = newValue;
        return oldValue;
    }

    public final boolean equals(Object o) {
        if (o == this)
            return true;
        if (o instanceof Map.Entry) {
            Map.Entry<?,?> e = (Map.Entry<?,?>)o;
            if (Objects.equals(key, e.getKey()) &&
                Objects.equals(value, e.getValue()))
                return true;
        }
        return false;
    }
}
```

## 再次开撸

和阈值一样，我也想显式的知道这个table最起码的长度是多少，于是再度用到了反射，把昨天的代码稍加修改：

```java
package man;

import java.lang.reflect.Field;
import java.util.HashMap;
import java.util.Map;
import java.util.Objects;

public class MyHashMap {
	public static void main(String args[]) throws Exception {
		Map map = new HashMap();
		System.out.println("未加数据的阈值：" + getThr(map));
		try {
			System.out.println("未加数据的table个数：" + getTable(map));//如果有错，try catch捕获 即table为null
		} catch (Exception e) {
			System.out.println(e.getMessage());
		}

		int k = 2;
		for (int i = 0; i < k; i++) {
			map.put(i, i);
		}

		System.out.println("实际个数：" + map.size());
		try {
			System.out.println("table个数：" + getTable(map));
		} catch (Exception e) {
			System.out.println(e.getMessage());
		}
		System.out.println("阈值：" + getThr(map));
	}

	static int getThr(Map map)
			throws NoSuchFieldException, SecurityException, IllegalArgumentException, IllegalAccessException {
		Class clazz = map.getClass();
		Field f = clazz.getDeclaredField("threshold");
		f.setAccessible(true);
		return (int) f.get(map);

	}
//比较难的是，table是个Node类型的数组，Node是HashMap中的内部类
	static int getTable(Map map) throws Exception {
		Class clazz = map.getClass();
		Field f = clazz.getDeclaredField("table");
		f.setAccessible(true);
		if (f.get(map) == null) {   //table可以为空，如果是空就抛出错误
			throw new Exception("table为null");
		} else {
			Object obj = f.get(map);    //以Object来接受
			if (obj.getClass().isArray()) {   //判断是否为数组
				Object[] arr = (Object[]) obj;   //进行强转
				return arr.length;        //返回table的长度
			} else {
				throw new Exception("获取失败");
			}
		}
	}
}
```

## new HashMap()

HashMap构造不传参，k=0，不添加数据：

```
未加数据的阈值：：0
table为null
实际个数：0
table为null
阈值：0
```

k=1,加入一条数据：

```
未加数据的阈值：：0
table为null
实际个数：1
table个数：16
阈值：12
```

k=2:

```
未加数据的阈值：：0
table为null
实际个数：2
table个数：16
阈值：12
```

k=13：

```
未加数据的阈值：：0
table为null
实际个数：13
table个数：32
阈值：24
```

## new HashMap(8)

k=0：

```
未加数据的阈值：：8
table为null
实际个数：0
table为null
阈值：8
```

k=1:

```
未加数据的阈值：：8
table为null
实际个数：1
table个数：8
阈值：6
```

k=2:

```
未加数据的阈值：：8
table为null
实际个数：2
table个数：8
阈值：6
```

k=7

```
未加数据的阈值：：8
table为null
实际个数：7
table个数：16
阈值：12
```

## 对比发现

可以发现，无论构造函数是否带参数，没有进行put时候的table为null，很好理解，因为构造函数里面没有写相关table的操作，table默认值自然为null。

然而，在第一次put一个值的时候，就发生了变化，所以打开put的源码，

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}


final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)  //1条件 当table=null或者table.length=0
        n = (tab = resize()).length;          //这里可能会发生扩容 第一次put的时候肯定会扩容
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        ......
    }
    ......
    if (++size > threshold)                       //2条件
        resize();                        //这里可能发生扩容
    afterNodeInsertion(evict);
    return null;
}
```

所以有了第一个结论：

> 无论是否带有参数，在第一次put的时候否会发生扩容。

这也是我昨天的一个误区，认为带了参数构造函数的第一次put不需要扩容，这是错误的。



扩容的时候，阈值和table都会发生变化，那么到底怎么变化：

```java
final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {    //A入口
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0) //B入口 initial capacity was placed in threshold 初始容量被置于阈值
            newCap = oldThr;
        else {   //C入口 zero initial threshold signifies using defaults 零初始阈值表示使用默认值
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;         //阈值发生变化
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;             //table发生变化

        //数据复制到新的table中
        ......
        return newTab;
}
```

## 总结：

在这里总结一下：

1. 不管构造的时候带不带参数，第一次put的时候都会发生一次扩容。
2. 第一次put的时候，如果带了参数，进入B入口，如果没带，使用默认参数，进入C入口，之后的put都会进入A入口。
3. 小插曲：发现new HashMap(0)时第一次put的时候结果不对，之后发现这时第一次put会扩容两次（同时满足1、2条件），也是厉害了。
4. 不大清楚A入口里面`oldCap >= DEFAULT_INITIAL_CAPACITY`的具体作用，不过在构造函数参数很小的时候，这个确实发挥了作用。



最后一点，就是这个容量问题，结合这两篇文章来看：**容量是可以用table的长度来描述（即确实间接是有一个变量来描述容量），但是这些必须是在第一次put数据进去之后，单纯的new HashMap([arg]),只能知道阈值，table是null，自然没有容量这么一说了。**

仅个人理解。



HashMap里的东西绝对不止这么一点，这里仅仅是扩容的容量问题，还有里面的hash算法、链表转为红黑树、线程不安全等等。以后有机会的话再来记录吧。