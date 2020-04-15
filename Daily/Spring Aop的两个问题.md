# Spring Aop的两个问题

> 时间: 2020/4/10

先说两个结论：

> 1. `Spring`事务默认只对`RunTimeException`或`error`进行捕获回滚。但是可以通过`rollbackFor`来指定。
> 2. `AOP`生效的情况必须是代理对象来调用目标方法。

第一条很简单，一般都知道，但是第二条发现网上很多资料或者博客都没有提到这个。

只有是`Spring`代理的对象来调用目标方法，事务才会生效，正如上面所说，**不只是事务，只要是`Spring`做的`AOP`**，都必须是要代理对象来调用才会生效。

下面典型的错误示例：

```java
@Transactional
@Override
public void A(){
    ...
    B();
    ...
}
@Transactional
@Override
public void B(){
    ...
}
```

`A`和`B`都是实现`Service`层接口中的方法。其实代码运行出来，大部分情况下效果能符合我们的预期，但是**这是错误的用法，因为直接`B()`，调用B方法的不是Spring的代理对象，而是当前类的对象，`AOP`不会生效**。

当有别的地方对`B()`方法做了`AOP`,比如说做了个日志记录的`Aspect`,那么在这边A方法调用`B`方法的时候，`Aspect`不会生效。

或者将`B`上的`@Transactional`，改为`@Transactional(propagation = Propagation.REQUIRES_NEW)`,从详细的日志也可以看到，执行`B`方法并没有给我们重新开启一个事务。

正确的写法：

在`Spring Boot`启动类上加上`@EnableAspectJAutoProxy(exposeProxy=true)`,允许获取代理对象。调用`B()`方法时:

`((T)AopContext.currentProxy()).B()`

有机会会把自己之前测得代码拉上来。



## Transaction rolled back because it has been marked as rollback-only

这个错也很常见，理解起来也不难，在事务默认的传播机制下，**子方法抛出异常，将事务标记为`rollback-only`,但是父方法调用子方法时`try catch`了，父方法执行完之后想提交事务发现事务已经是`rollback-only`了，所以抛出异常。**

这边我仔细拿出来测试了一下，想看一下具体的场景，测试代码很简单，`USER`和`BOOK`两个表,各往其插入一条数据，主要是两个方法：

```java
@Override
//@Transactional(rollbackFor = Exception.class)
@Transactional
public void userMethod7() throws Exception {
    userMapper.insertOne();  //USER表插一条数据
    try{
        ((UserService) AopContext.currentProxy()).userMethod8();
        // userMethod8();
    }catch(Exception e) {
    }
}

@Override
//@Transactional(rollbackFor = Exception.class)
@Transactional
//@Transactional(propagation = Propagation.REQUIRES_NEW, rollbackFor = Exception.class)
public void userMethod8() throws Exception {
    bookMapper.insertOne();   //BOOK表插一条数据
    //throw new Exception("抛出Exception");
    throw new RuntimeException("抛出RunTimeException");
}
```

两个方法都是实现`Service`层的方法，7 调用 8，因为看网上有人说和`Exception`和`RunTimeException`有关，这边我分别都测一下。

- 上面先测一下`RunTimeException`，运行结果

![rollback-only.png](http://www.qxnekoo.cn:8888/images/2020/04/10/rollback-only.png)

抛出错误。

表中也无数据，但并不是由于回滚了，而是由于这个错。



- 下面测试一下`Exception`,修改代码：

```java
@Override
@Transactional(rollbackFor = Exception.class)
//@Transactional
public void userMethod7() throws Exception {
    userMapper.insertOne();
    try{
        ((UserService) AopContext.currentProxy()).userMethod8();
        // userMethod8();     
    }catch(Exception e) {
    }
}

@Override
@Transactional(rollbackFor = Exception.class)
//@Transactional
//@Transactional(propagation = Propagation.REQUIRES_NEW, rollbackFor = Exception.class)
public void userMethod8() throws Exception {
    bookMapper.insertOne();
    throw new Exception("抛出Exception");
    //throw new RuntimeException("抛出RunTimeException");
}
```

运行结果：

![rollback-only.png](http://www.qxnekoo.cn:8888/images/2020/04/10/rollback-only.png)

也会报出这个错误。

还有两种情况：

- 如果在`rollbackFor = Exception.class`的情况下去抛出`RunTimeException`还是会报出这个错，说明`Spring`即使在`rollbackFor = Exception.class`下，还是能捕获到`RunTimeException`。

- 如果在默认情况下（`@Transactional`）下抛出`Exception`,并不会报错，程序正常执行完毕，而且发现表中都插入了数据。（其实我想写出这样代码的人应该都是想的是：**如果8方法中出错，只回滚8方法，7方法不要回滚，所以7方法中try catch住**）显然也不符合预期结果。

  如果真的上面这个需求，使用`Propagation.REQUIRES_NEW`或者`Propagation.NESTED`,开启一个新的事务或者使用嵌套事务。**使用时要注意，父方法还是要用`try catch`的，不然错误会在父方法抛出，导致父方法的事务被回滚。**

`Propagation.REQUIRES_NEW`不要和我以前一样，认为：**两个不同的事务，子方法出错会回滚，父方法不做任何处理也没有关系，也不会回滚，因为是两个事务。**

上面的观念是错误的，因为父方法不处理，会接着往外抛，导致父方法回滚。



上面的代码，其实我还测了不使用代理对象来调用的情况，就是`userMethod8();  `这样调用的方式。结果和上面第四种情况一样，数据全部插进去了。所以一定要注意这些错误的写法。



总结一下这个错：大概率是在默认的传播机制下，父方法调用子方法时，想要`try catch`住子方法的错误，达到一种子方法回滚，父方法不回滚的效果。 和`Exception`和`RunTimeException`没有关系，两者都会导致这个错。



-----如果文中有错误，欢迎进行指导。