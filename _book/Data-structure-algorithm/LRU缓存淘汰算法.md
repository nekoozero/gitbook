# LRU缓存淘汰算法

LRU(Least Recently Used)缓存淘汰算法是一种常用策略。

## 基本实现

维护一个有序单链表，越靠近链表尾部的节点是越早之前访问的。当有一个新的数据被访问时，我们从链表头开始顺序遍历链表。

1. 如果此数据之前被缓存在链表中了，我们遍历得到这个数据对应的节点，并将其从原来的位置删除，然后再插入到链表的头部
2. 如果此数据没有在缓存链表中：
   - 如果此时缓存未满，则将此节点直接插入到链表的头部；
   - 如果此时缓存已满，则链表尾节点删除，将新的数据节点插入链表的头部。

不管缓存有没有满，我们都需要遍历一遍链表，所以这种基于链表的实现思路，缓存访问的时间复杂度为 O(n)。

## 力扣146LRU缓存机制

接受一个 capacity 参数作为缓存的最大容量，实现两个 API，一个 put(key,val) 方法存入键值对，另一个是 get(key) 方法获取 key 对应的 val，如果 key 不存在则返回 -1。（get、put 方法的时间复杂度为 O(1)）

### 算法设计

1. cache 中的元素必须要有时序，以区分最近使用和久未使用的数据，容量满了后要删除最久未使用的元素。
2. 要在 cache 中快速找到某个 key 是否存在并且返回对应的 val。
3. 每次访问 cache 中的某个 key，需要将这个元素变为最近使用的，cache 要支持在**任意位置快速插入和删除元素**。

哈希表查找快，但数据无固定顺序；链表有顺序之分，插入、删除快，但是查找慢，所以一个新的数据结构：**哈希链表 LinkedHashMap**。

>为什么要双向链表？
>
>**双向链表删除节点的时间复杂度为 O(1)。**
>
>哈希表中已经存在了 key 了，为什么链表中还要存 key，只存 val 不行吗？
>
>删除节点时还要删除哈希表中对应的内容，key 只能从链表的 Node 中拿。

### 代码实现

节点类：

```java
package leetcode.common.classic.lru;

/**
 * @author qianxin
 * @date 2021/05/20
 */
public class Node {
    public int key, val;
    public Node next, prev;

    public Node(int k, int v) {
        this.key = k;
        this.val = v;
    }
}
```

双链表：

```java
package leetcode.common.classic.lru;

/**
 * @author qianxin
 * @date 2021/05/20
 * 通过 Node 类型构建一个双链表，实现 LRU 算法的 API
 *
 * 实现的双链表 API 从尾部插入，也就是说靠尾部的数据是最近使用的，靠头部的数据是最久未使用的
 */
public class DoubleList {
    /**
     * 头尾虚节点
     */
    private Node head, tail;

    /**
     * 链表长度
     */
    private int size;

    public DoubleList() {
        //初始化双向链表的数据
        head = new Node(0, 0);
        tail = new Node(0, 0);
        head.next = tail;
        tail.prev = head;
        size = 0;
    }

    /**
     * 在链表尾部添加节点x，时间复杂度为O（1）
     *
     * @param x 添加的节点
     */
    public void addLast(Node x) {
        x.prev = tail.prev;
        x.next = tail;
        tail.prev.next = x;
        tail.prev = x;
        size++;
    }

    /**
     * 删除链表中的 x 节点（x一定存在）
     * 由于是双链表且给的是目标节点，时间复杂度为O(1)
     *
     * @param x 删除的节点
     */
    public void remove(Node x) {
        x.prev.next = x.next;
        x.next.prev = x.prev;
        size--;
    }

    /**
     * 双链表删除链表中的第一个节点，并返回该节点，时间复杂度为O(1)
     *
     * @return 被删除的节点
     */
    public Node removeFirst() {
        if (head.next == tail) {
            return null;
        }
        Node first = head.next;
        remove(first);
        return first;
    }

    /**
     * 返回链表长度，时间复杂度为O(1)
     *
     * @return 链表长度
     */
    public int size() {
        return size;
    }
}
```

LRU类：

```java
package leetcode.common.classic.lru;

import java.util.HashMap;

/**
 * @author qianxin
 * @date 2021/05/20
 * <p>
 * 双向链表和哈希表结合在一起
 */
public class LruCache {
    private HashMap<Integer, Node> nodeHashMap;
    private DoubleList cache;
    /**
     * 最大容量
     */
    private int cap;

    /**
     * 删除某个 key 的时候却忘记在 map 中删除对应的 key，有效的方法是：在这两种数据结构之伤提供一层抽象的 API
     * 避免 get 和 put 直接操作 cache 和 map 的细节
     *
     * @param capacity
     */
    public LruCache(int capacity) {
        this.cap = capacity;
        cache = new DoubleList();
        nodeHashMap = new HashMap<>();
    }

    /**
     * 将某个节点变为头节点（最初访问的）
     *
     * @param key 访问的 key
     */
    private void makeRecently(int key) {
        Node x = nodeHashMap.get(key);
        //先删除该节点
        cache.remove(x);
        //将该节点重新加入到队尾
        cache.addLast(x);
    }

    /**
     * 添加一个新的节点（最近使用元素）
     *
     * @param key 访问键
     * @param val 值
     */
    private void addRecently(int key, int val) {
        Node newNode = new Node(key, val);
        cache.addLast(newNode);
        nodeHashMap.put(key, newNode);
    }

    /**
     * 删除key
     *
     * @param key 要删除的 key
     */
    private void deleteKey(int key) {
        Node x = nodeHashMap.get(key);
        //移除节点
        cache.remove(x);
        //在map中删除
        nodeHashMap.remove(key);
    }

    /**
     * 移除最久未使用的元素
     */
    private void removeLeaseRecently() {
        //第一个元素就是最久未使用的
        Node node = cache.removeFirst();
        //在map中移除
        nodeHashMap.remove(node.key);
    }


    public int get(int key) {
        if (!nodeHashMap.containsKey(key)) {
            return -1;
        }
        makeRecently(key);
        return nodeHashMap.get(key).val;
    }

    public void put(int key, int val) {
        //如果已经包含了该元素 那么就删除在重新加入
        if (nodeHashMap.containsKey(key)) {
            deleteKey(key);
        }

        if (cap == cache.size()) {
            //如果容量满了 需要移除头节点最久未使用的
            removeLeaseRecently();
        }
        addRecently(key, val);
    }
}
```

### java 内置哈希链表

```java
package leetcode.common.classic.lru;

import java.util.LinkedHashMap;

/**
 * @author qianxin
 * @date 2021/05/21
 */
public class HashLinkedListLru {
    int cap;
    LinkedHashMap<Integer, Integer> cache = new LinkedHashMap<>();

    public HashLinkedListLru(int capacity) {
        this.cap = capacity;
    }

    private void makeRecently(int key) {
        int val = cache.get(key);
        //先移除元素
        cache.remove(key);
        //再添加到尾部
        cache.put(key, val);
    }

    public int get(int key) {
        if (!cache.containsKey(key)) {
            return -1;
        }
        makeRecently(key);
        return cache.get(key);
    }

    public void put(int key, int val) {
        if (cache.containsKey(key)) {
            cache.put(key, val);
            makeRecently(key);
            return;
        }
        if (cache.size() >= this.cap) {
            //链表头部的元素是最久未使用的
            int oldSetKey = cache.keySet().iterator().next();
            cache.remove(oldSetKey);
        }
        cache.put(key, val);
    }
}
```





















