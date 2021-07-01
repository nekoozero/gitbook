---
title: ThreadLocal基础
date: 2019-05-30 15:11:13
categories: 编程基础
---

## Thread.join()

> 时间: 2019/5/30

如果一个线程A执行了thread.join()，其含义是：当线程A等待thread线程终止之后才从thread.join()返回。（即线程A会从等待处继续执行）。

```java
package qxnekoo.threadlocal;

import java.util.concurrent.TimeUnit;

public class Join {
	public static void main(String args[]) throws InterruptedException {
		Thread previous = Thread.currentThread();
		for(int i = 1;i<=10;i++) {
			Thread thread = new Thread(new Nekoo(previous),String.valueOf(i));
			previous = thread;
			thread.start();
		}
		TimeUnit.SECONDS.sleep(2);
	
		System.out.println("主线程执行完毕");
	}
	static class Nekoo implements Runnable {
		Thread previous;
		public Nekoo(Thread t) {
			previous  =t;
		}
		@Override
		public void run() {
			try {
				previous.join();
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		System.out.println(Thread.currentThread().getName()+" terminated");
		}
	}
}
```

等待两秒后，出现结果：

```
主线程执行完毕
1 terminated
2 terminated
3 terminated
4 terminated
5 terminated
6 terminated
7 terminated
8 terminated
9 terminated
10 terminated
```

创建十个线程，只有前驱线程执行完成后，当前线程才可以继续执行。（等待前驱线程结束，接受前驱线程结束通知）



## ThreadLocal

介绍：

> *ThreadLocal类用来提供线程内部的局部变量。这种变量在多线程环境下访问(通过get或set方法访问)时能保证各个线程里的变量相对独立于其他线程内的变量。ThreadLocal实例通常来说都是*private static*类型的，用于关联线程和线程的上下文。*

ThreadLocal的作用是提供线程内的局部变量，这种变量在线程的生命周期内起作用，减少同一个线程内多个函数或者组件之间一些公共变量的传递的复杂度。

初衷：提供线程内部的局部变量，在本线程内随时随地可取，隔离其他线程。

### **initialValue函数**

```java
protected T initialValue() {
    return null;
}
```

该函数在调用get函数的时候会第一次调用，但是如果一开始就调用了set函数，则该函数不会被调用。通常该函数只会被调用一次，除非手动调用了remove函数之后又调用get函数，这种情况下，get函数中还是会调用initialValue函数。该函数是protected类型的，很显然是建议在子类重载该函数的，所以通常该函数都会以匿名内部类的形式被重载，以指定初始值

### 代码演示

```java
package qxnekoo.threadlocal;

public class TestThreadLocal {
    private static final ThreadLocal<Integer> value= new ThreadLocal<Integer>() {
    	@Override
    	protected Integer initialValue() {
    		return 1;
    	}
    };
    public static void main(String arrgs[]) {
    	for(int i = 0;i<5;i++) {
    		new Thread(new MyThread(i)).start();
    	}
    }
    static class MyThread implements Runnable{
        private int index;
        public MyThread(int index) {
        	this.index =index;
        }
		@Override
		public void run() {
			System.out.println("线程"+index+"的初始value："+value.get());
			for(int i=0;i<10;i++) {
				value.set(value.get()+i);
			}
			System.out.println("线程"+index+"的累加value"+value.get());
		}
    }
}
```

结果：

```
线程0的初始value：1
线程0的累加value46
线程2的初始value：1
线程1的初始value：1
线程1的累加value46
线程4的初始value：1
线程2的累加value46
线程4的累加value46
线程3的初始value：1
线程3的累加value46
```

可以看到，各个线程的value值是相互独立的，本线程的累加操作不会影响到其他线程的值，真正达到了线程内部隔离的效果。

在这里只进行简单概念性的介绍，深入的内容之后有机会会来记录。

