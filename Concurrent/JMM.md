# JMM

`Java`内存模型的目的是定义程序中各种变量的访问规则，即关注在虚拟机中把变量值存储到内存和从内存中取出变量值这样的底层细节。

这里的变量包括了实例字段、静态字段和构成数组对象的元素，不包括局部变量与方法参数，因为就这是线程私有的，不会被共享，自然就不会存在竞争问题。

定义：线程之间的共享变量存储在主内存中，每个线程都有一个私有的本地内存，本地内存中存储了该线程以读/写共享变量的副本。本地内存是`JMM`的一个抽象概念，并不真实存在。线程对变量的所有操作都必须在工作内存中进行，而不能直接读写主内存中的数据。不同的线程之间也无法直接访问对方工作内存中的变量，线程间变量值的传递均需要通过主内存来完成。（`volatile`变量依然有工作内存的拷贝，但是由于他特殊的操作顺序行，所以看起来如同直接在主内存中读写一样）

![JMM.png](http://www.qxnekoo.cn:8888/images/2020/03/19/JMM.png)

## 内存间交互操作

主内存与工作内存之间具体的交互协议，即一个变量如何从主内存拷贝到工作内存、如何从工作内存同步回主内存之类的实现细节，java 内存模型中定义了一下 8 种操作来完成，虚拟机实现时必须保证下面提及的每一种操作都是原子的、不可再分的（对于 double 和 long 类型的变量来说，load、store、read 和 write 操作在某些平台上允许有例外）。

- lock（锁定）：作用于主内存的变量，他把一个变量表示为一条线程独占的状态。
- unlock（解锁）：作用于主内存的变量，他把一个处于锁定状态的变量释放出来，释放后的变量才可以被其他线程锁定。
- read（读取）：作用于主内存的变量，他把一个变量的值从主内存传输到线程的工作内存中，以便随后的 load 动作使用。
- load（载入）：作用于主内存的变量，他把 read 操作从主内存中得到的变量值放入工作内存的变量副本中。
- use（使用）：作用于工作内存的变量，他把工作内存中一个变量的值传递给执行引擎，每当虚拟机遇到一个需要使用到变量的值的字节码指令时将会执行这个操作。
- assign（赋值）：作用域工作内存的变量，他把一个从执行引擎接收到的值赋值给工作内存的变量，每当虚拟机遇到一个给变量赋值的字节码指令时执行这个操作。
- store（存储）：作用于工作内存的变量，他把工作内存中的一个变量的值传送到主内存中，一边随后的 write 操作使用。
- write（写入）：作用于主内存的变量，他把 store 操作从工作内存中得到的变量值放入主内存的变量中。

如果要把一个变量从主内存复制到工作内存，那就要顺序地执行 read 和 load 操作，如果要把变量从工作内存同步回主内存，就要顺序地执行 store 和 write 操作。注意，java 内存模型只要求上述两个操作是可插入指令的，如对主内存中的变量 a、b 进行访问时，一种可能出现的顺序是 read a、read b、load b、load a。除此之外，java 内存模型还规定了在执行上述8种基本操作必须满足很多规则。

由于这种定义相当严谨但又十分繁琐，实践起来很麻烦，所以有一个等效判断原则————先行发生原则，用来确定在一个访问在并发环境下是否安全。

## 先行发生原则

它是`Java`内存模型中定义的两项操作之间的偏序关系。是判断数据是否存在竞争，线程是否安全的手段。可以通过几条简单规则来解决并发环境下两个操作之间是否可能存在冲突的所有问题。

> 如果`Java`内存模型中所有的有序性都依靠`volatile`和`synchronzed`来完成，那么很多操作都会很啰嗦，但是我们在写代码的时候并没有察觉到这个，就是因为有这个先行发生原则的存在。

### 讲解

1. A先行发生于B操作。

在发生操作B之前，操作A产生的影响能够被B观察到，“影响”包括修改了内存中共享变量的值、发送了消息、调用了方法。