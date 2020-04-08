# JUC 常用并发工具类

## CountDownLatch

`CountDownLatch`允许一个或多个线程等待其他线程完成操作。

定义了一个就计数器和一个阻塞队列，当计数器的值递减为0之前，阻塞队列里面的线程处于挂起状态，当计数器递减为0时会唤醒阻塞队列所有线程，在`CountDownLatch`上等待的线程就可以恢复执行接下来的任务。

### 常用方法

```java
CountDownLatch(int count); //构造方法，创建一个值为count 的计数器。
await();//阻塞当前线程，将当前线程加入阻塞队列。
await(long timeout, TimeUnit unit);//在timeout的时间之内阻塞当前线程,时间一过则当前线程可以执行，
countDown();//对计数器进行递减1操作，当计数器递减至0时，当前线程会去唤醒阻塞队列里的所有线程。
```



> `CountDownLatch`是一次性的，计算器的值只能在构造方法中初始化一次，之后没有任何机制再次对其设置值，当`CountDownLatch`使用完毕后，它不能再次被使用。

这个的用法还是比较简单的。

## CyclicBarrier

`CyclicBarrier`是一个同步辅助类，它允许一组线程互相等待，直到所有线程都到达某个公共屏障点（也可以叫同步点），即互相等待的线程都完成调用`await`方法，所有被屏障拦截的线程才会继续运行`await`方法后面的程序。

> 1. `CyclicBarrier(int parties, Runnable barrierAction)` 创建一个`CyclicBarrier`实例，`parties`指定参与相互等待的线程数，`barrierAction`指定当所有线程到达屏障点之后，首先执行的操作。
> 2. `CyclicBarrier(int parties) `创建一个`CyclicBarrier`实例，parties指定参与相互等待的线程数。
> 3. `getParties() `返回参与相互等待的线程数。
> 4. `await()` 该方法被调用时表示当前线程已经到达屏障点，**当前线程阻塞进入休眠状态**，直到所有线程都到达屏障点，当前线程才会被唤醒。
> 5. `await(long timeout, TimeUnit unit)` 该方法被调用时表示当前线程已经到达屏障点，当前线程阻塞进入休眠状态，在`timeout`指定的超时时间内，等待其他参与线程到达屏障点；如果超出指定的等待时间，则抛出`TimeoutException`异常，如果该时间小于等于零，则此方法根本不会等待。
> 6. `isBroken()` 判断此屏障是否处于中断状态。如果因为构造或最后一次重置而导致中断或超时，从而使一个或多个参与者摆脱此屏障点，或者因为异常而导致某个屏障操作失败，则返回`true`；否则返回`false`。
> 7. `reset()` 将屏障重置为其初始状态。
> 8. `getNumberWaiting() `返回当前在屏障处等待的参与者数目，此方法主要用于调试和断言。

### 与 CountDownLatch 的区别

最主要的是`CountDownLatch`只能使用一次，而`CyclicBarrier`计数器可以循环使用，也可以使用`reset()`方法重置。

### 代码演示

```java
package concurrent;

import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;

public class Cycle {
    static CyclicBarrier c = new CyclicBarrier(2, new A());

    public static void main(String[] args) throws BrokenBarrierException, InterruptedException {
        new Thread(() -> {
            try {
                System.out.println(Thread.currentThread().getName() + ":" + 1);
                c.await();
            } catch (InterruptedException e) {
                e.printStackTrace();
            } catch (BrokenBarrierException e) {
                e.printStackTrace();
            }
        }).start();
        System.out.println(Thread.currentThread().getName() + ":" + 2);
        c.await();
        new Thread(() -> {
            try {
                System.out.println(Thread.currentThread().getName() + ":" + 11);
                c.await();
            } catch (InterruptedException e) {
                e.printStackTrace();
            } catch (BrokenBarrierException e) {
                e.printStackTrace();
            }
        }).start();
        System.out.println(Thread.currentThread().getName() + ":" + 22);
        c.await();
    }

    static class A implements Runnable {
        @Override
        public void run() {
            System.out.println(Thread.currentThread().getName() + ":" + 3);
        }
    }
}

```

输出结果：

> main:2
> Thread-0:1
> Thread-0:3
> main:22
> Thread-1:11
> Thread-1:3

可以看虽然是设定了2的值，但是是可以循环的，回调函数在一次循环结束之后调用。

**使用时一定要注意下一次循环开始不要在上一个循环的回调函数之前执行。**

## Semaphore

叫做信号量，有两个目的，一个是多个共享资源互斥使用，第二个是并发线程数的控制。

`Semaphore`用于限制可以访问某些资源（物理或逻辑的）的线程数目，他维护了一个许可证集合，有多少资源需要限制就维护多少许可证集合，假如这里有`N`个资源，那就对应于`N`个许可证，同一时刻也只能有N个线程访问。一个线程获取许可证就调用`acquire`方法，用完了释放资源就调用`release`方法。

`Semaphore`的构造方法`Semaphore(int permits)`接受一个整型的数字，表示可用的许可证数量。`Semaphore(10)`表示允许10个线程获取许可证，也就是最大并发数是10。

### 代码演示

```java
package concurrent;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Semaphore;

public class Semap {
    private static final int THREAD_COUNT = 30;

    private static ExecutorService es =Executors.newCachedThreadPool();
    private static Semaphore s = new Semaphore(10);

    public static void main(String[] args) {
        for(int i = 0;i<THREAD_COUNT;i++) {
            es.execute(()->{
                try {
                    s.acquire();
                    System.out.println("save data");
                    s.release();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            });
        }
        es.shutdown();
    }
}
```















