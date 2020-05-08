# volatile

> 时间: 2020/3/29

## 保证变量的可见性

`volatile`修饰的变量对所有线程都具有可见性，这边的可见性主要是说，新值对于其他线程来说是立即得知的。

*`volatile` 修饰之后并不是让线程直接从主内存中获取数据，依然需要将变量拷贝到工作内存中*。当一个变量被 `volatile` 修饰时，任何线程对它的**写操作都会立即刷新到主内存中，并且会强制让缓存了该变量的线程中的数据清空**，必须从主内存重新读取最新数据。

当我们需要在两个线程间依据主内存通信时，通信的那个变量就必须的用 `volatile` 来修饰。

1. 对变量的写操作不依赖于当前值
2. 该变量没有包含在具有其他变量的不变式中

实际上，这些条件表明，可以被写入 `volatile` 变量的这些有效值独立于任何程序的状态，包括变量的当前状态。需要保证操作是原子性操作（`volatile`不具有原子性），才能保证使用volatile关键字的程序在并发时能够正确执行。

## 保证顺序性（禁止重排序）

禁止指令重排序，例子

```java
private static Map<String,String> value ;
private static volatile boolean flag = fasle ;

//以下方法发生在线程 A 中 初始化 Map
public void initMap(){
	//耗时操作
	value = getMapValue() ;//1
	flag = true ;//2
}


//发生在线程 B中 等到 Map 初始化成功进行其他操作
public void doSomeThing(){
	while(!flag){
		sleep() ;
	}
	//dosomething
	doSomeThing(value);
}
```

这里就能看出问题了，当 `flag` 没有被 `volatile` 修饰时，`JVM` 对 1 和 2 进行重排，导致 `value` 都还没有被初始化就有可能被线程 B 使用了。

还有就是双重检测机制了，这里就不细说了，主要是因为会对`new`对象的汇编指令进行重排序。

```java
public class Singleton {

    private static volatile Singleton singleton;

    private Singleton() {
    }

    public static Singleton getInstance() {
        if (singleton == null) {
            synchronized (Singleton.class) {
                if (singleton == null) {
                    //防止指令重排
                    singleton = new Singleton();
                }
            }
        }
        return singleton;
    }
}
```

有`volatile`修饰的变量，赋值后多执行了一个"`lock add1 0x0,(%esp)`"操作，这个操作的作用相当于一个**内存屏障**，重排序时，不能把后面的指令重排序到内存屏障之前的位置。如果只有一个处理器访问内存时，并不需要内存屏障；但如果有两个或更多处理器访问同一块内存，且其中一个在观测另一个，就需要内存屏障来保证一致性了。

## 随记

```java
public class VolatileExample {

    private static boolean flag = false;
    private static int i = 0;
    public static void main(String[] args) {
        new Thread(() -> {
            try {
                TimeUnit.MILLISECONDS.sleep(100);
                flag = true;
                System.out.println("flag 被修改成 true");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();
        while (!flag) {
            i++;
        }
        System.out.println("程序结束,i=" + i);
    }
}
```

这个程序的在`java8`服务端的运行情况，即程序不会停止，一直在循环。虽然知道单纯`flag`变量无法保证在线程之间的可见性，但为什么程序会一直执行下去？后台线程修改`flag`变量虽然暂时是对主线程不可见的，但难道后台线程的工作内存一直不会将值刷新到主内存吗？主线程也一直在从自己的工作内存中取值，而不是从主内存拿值？

**多线性并发时，如果 A 线程修改了共享变量，此时 B 线程感知不到此共享变量的变化，叫做活性失败。**

因为编译器对代码进行了优化：

```java
while (!flag) {
            i++;
}
//变成了
if(!flag) {
    while(true)
        i++;
}
```

这种优化是可以接受的，叫做提升（`hoisting`）,编译器会帮我们做一些大部分经典的优化动作：无用代码消除、循环展开、**循环表达式外提**、消除公共子表达式、常量传播、基本块重排序等。

**这个提升是` JIT `帮我们做的**，循环表达式外提。

还是一样的代码，禁用了 `JIT `的优化（添加虚拟机参数：`-Djava.compiler=NONE`）。程序正常运行结束了。

综上，如何解释为什么程序不会正常结束了：

> 由于变量`flag`没有被`volatile`修饰，而且子线程休眠的`100ms`中，`while`循环的`flag`一直为`false`,循环到一定的次数后，触发了`jvm`的即时编译功能，进行了循环表达式外提（Loop Expression Hoisting），导致形成死循环。而如果加了`volatile`去修饰`flag`变量，保证了`flag`的可见性，则不会进行提升。