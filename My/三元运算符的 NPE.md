# 三元运算符的 NPE

> 时间 2020/4/27

今天上班（摸鱼）的时候，看到了这样一篇博文：

https://mp.weixin.qq.com/s/iQ6qdNv7WTLa3drxNa9png

关于三目运算符的 NPE问题，这个问题我还从来没接触过，就研究了一番。

主要是阿里巴巴Java开发手册发布了最新版，新增了一条规约：

![alibaba1.jpg](http://www.qxnekoo.cn:8888/images/2020/04/27/alibaba1.jpg)

我们把这个反例代码拿到 `IDEA` 中研究一下:

按照以前的想法，觉得得到的 `result` 结果为 `null`，但现实是残酷的：

![fd703a8560a7fcee6aec31f285ca36ab.png](http://www.qxnekoo.cn:8888/images/2020/04/27/fd703a8560a7fcee6aec31f285ca36ab.png)

讲真的，完全看代码，我还真不知道这个空指针是在哪个地方报的。

先不急，慢慢来，将 `flag` 改为 `true`，执行结果，代码成功运行结束。`result` 结果为 2，那么难道是三目运算符第三个参数 `c` 报的空指针嘛！

其实那个规约中已经讲的很清楚了，因为自动拆箱导致NPE，那是因为啥导致自动拆箱了呢？其实上面也说了三目运算符自动拆箱的场景：

1. **表达式一和表达式二只要有一个是原始类型**。
2. **表达式一和表达式二的值的类型不一致，会强制拆箱升级成表示范围更大的那个类型。**

把 `flag` 改为 `flase`，我们可以对 `class` 文件进行 `javap -c` 查看

![javap.png](http://www.qxnekoo.cn:8888/images/2020/04/27/javap.png)

我理解的 `intValue` 就应该是自动拆箱了，最后再来查看一下反编译后的代码：

```java
public class San {
  public static void main(String[] args) {
    Integer a = Integer.valueOf(1);
    Integer b = Integer.valueOf(2);
    Integer c = null;
    Boolean flag = Boolean.valueOf(false);
    Integer result = Integer.valueOf(flag.booleanValue() ? (a.intValue() * b.intValue()) : c.intValue());
    System.out.println(result);
  }
}
```

这样一看就很明显了，`c.intValue()`，空指针就是在这边报出来的。



将运行代码改一下：

```java
Integer i = 9;
Integer c = null;
Boolean flag = false;
Integer result = (flag ? i : c);
```

反编译出来：

```java
public class San {
  public static void main(String[] args) {
    Integer i = Integer.valueOf(9);
    Integer c = null;
    Boolean flag = Boolean.valueOf(false);
    Integer result = flag.booleanValue() ? i : c;
  }
}
```

可以看到，并没有出现自动拆箱。

<u>经过测试发现，`Integer` 类型发生运算时，会导致其自动拆箱，变为基础类型，那么三元表达式中的另外一个参数也会自动拆箱，导致 `null` 调用方法，产生 `NPE`。这应该是上面产生 `NPE` 的原因了。</u>

这也是上面博文中得到的结论：

> 由于使用了三目运算符，并且第二、第三位操作数分别是基本类型和对象。所以对对象进行拆箱操作，由于该对象为null，所以在拆箱过程中调用null.intValue()的时候就报了NPE。



疑问：博文中举得例子我在运行的时候并没有 `NPE` 产生。(win10 java version"1.8.0_121")

```java
Map<String, Boolean> map = new HashMap<>();
Boolean b = ((map != null) ? map.get("test") : false);
```

反编译之后

```java
public class San {
  public static void main(String[] args) {
    Map<String, Boolean> map = new HashMap<>();
    Boolean b = (map != null) ? map.get("test") : Boolean.valueOf(false);
  }
}
```

其实细心的我发现将第二句改为 `boolean b = ((map != null) ? map.get("test") : false);`就会抛出 `NPE` 了。可能是由于博主笔误了吧。

再次反编译:

```java
public class San {
  public static void main(String[] args) {
    Map<String, Boolean> map = new HashMap<>();
    boolean b = (map != null) ? ((Boolean)map.get("test")).booleanValue() : false;
  }
}
```

这个空指针也能理解了，也是由于拆箱导致的 `NPE`。

小插曲：boolean作为基础类型是必须给默认值的，否则会编译报错。



> 时间 2020/5/13 更新

想起之前的一个空指针，mybatis中查出的数据放到一个pojo中，但是表中是没有数据的，按理说查出来的list应该是没有数据的，也就是说长度为0，但是那次报了个空指针，然后找到报错的地方，竟然是一个属性的get方法报出来的，也就是：

```java
public long getXxx() {
    return xxx;        //这边报了个空指针出来  我差点吐了
}
```

之后发现这个get函数的返回类型是基本类型long，但是xxx的类型是Long，当时没仔细去想，把long改为Long，之后就不报空指针了。

> 现在看来，其实也就是自动拆箱导致的空指针问题！因为表中的没有数据，查出来自然是null，然后自动拆箱，抛出空指针，binggo！