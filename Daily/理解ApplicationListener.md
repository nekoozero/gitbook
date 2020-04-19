# 理解ApplicationListener

> 时间：2020/4/19

监听容器中发布的事件，完成事件驱动模型的开发。是**观察者设计模式**的实现，通过 `ApplicationEvent` 和 `ApplicationListener` 接口，可以实现 `ApplicationContext` 事件处理。

简单的来说，在容器中注册一个 `ApplicationListener` 监听器，每当 `ApplicationContext` 发布 `ApplicationEvent` 时，监听器会根据事件的类型来处理一些逻辑。

## 监听器

创建一个监听器只需要去实现 `ApplicationListener` 接口，重写 `onApplicationEvent` 方法，并放到 `Spring` 容器即可。

下面是部分源码：

```java
package org.springframework.context;
import java.util.EventListener;
@FunctionalInterface
public interface ApplicationListener<E extends ApplicationEvent> extends EventListener {
    void onApplicationEvent(E var1);
}


//ApplicationEvent抽象类
package org.springframework.context;
import java.util.EventObject;
public abstract class ApplicationEvent extends EventObject {
    private static final long serialVersionUID = 7099057708183571937L;
    private final long timestamp = System.currentTimeMillis();

    public ApplicationEvent(Object source) {
        super(source);
    }

    public final long getTimestamp() {
        return this.timestamp;
    }
}

//EventObject
package java.util;
/**
 * <p>
 * The root class from which all event state objects shall be derived.
 * <p>
 * All Events are constructed with a reference to the object, the "source",
 * that is logically deemed to be the object upon which the Event in question
 * initially occurred upon.
 *
 * @since JDK1.1
 */
public class EventObject implements java.io.Serializable {

    private static final long serialVersionUID = 5516075349620653480L;

    /**
     * The object on which the Event initially occurred.
     */
    protected transient Object  source;

    /**
     * Constructs a prototypical Event.
     *
     * @param    source    The object on which the Event initially occurred.
     * @exception  IllegalArgumentException  if source is null.
     */
    public EventObject(Object source) {
        if (source == null)
            throw new IllegalArgumentException("null source");

        this.source = source;
    }

    /**
     * The object on which the Event initially occurred.
     *
     * @return   The object on which the Event initially occurred.
     */
    public Object getSource() {
        return source;
    }

    /**
     * Returns a String representation of this EventObject.
     *
     * @return  A a String representation of this EventObject.
     */
    public String toString() {
        return getClass().getName() + "[source=" + source + "]";
    }
}


//EventListener接口
package java.util;
/**
 * A tagging interface that all event listener interfaces must extend.
 * @since JDK1.1
 */
public interface EventListener {
}

```

## 内置事件



| 内置事件                    | 描述                                                         |
| --------------------------- | ------------------------------------------------------------ |
| **`ContextRefreshedEvent`** | `ApplicationContext` 被初始化或刷新时，该事件被发布。这也可以在 `ConfigurableApplicationContext` 接口中使用 `refresh()` 方法来发生。此处的初始化是指：所有 `Bean` 被成功装在，后处理 `Bean` 被检测并激活，所有 `Singleton Bean` 被实例化，`ApplicationContext` 容器已就绪可用。 |
| **`ContextStartedEvent`**   | 当使用  `ConfifurableApplicationContext`（`ApplicationContext` 子接口）接口中的 `start` 方法启动 `ApplicationContext` 时，该事件被发布。可以调查数据库，或者可以在接收到这个事件后重启任何停止的应用程序。 |
| **`ContextStoppedEvent`**   | 当使用 `ConfifurableApplicationContext` 接口中的 `stop` 停止 `ApplicationContext` 时，发布这个事件，可以在接收到这个事件后作必要的清理工作。 |
| **`ContextClosedEvent`**    | 当使用 `ConfigurableApplicationContext` 接口中的 `close()` 方法关闭 `ApplicationContext` 时，该事件被发布。一个已关闭的上下文到达生命周期末端；它不能被刷新或重启。 |
| **`RequestHandledEvent`**   | 这是一个 `web-specific` 事件，告诉所有 `bean HTTP` 请求已经被服务。只能应用于使用 `DispatcherServlet` 的 `Web` 应用。在使用 `Spring` 作为前端的 `MVC` 控制器时，当 `Spring` 处理用户请求结束后，系统会自动触发该事件。 |



想上一节使用的例子就是 **`ContextRefreshedEvent`** 事件，然后去执行相应的逻辑。



## 自定义事件

`EmailEvent` 事件:

```java
public class EmailEvent extends ApplicationEvent {

    private String address;
    private String text;

    //重写构造方法
    public EmailEvent(Object source, String address, String text) {
        super(source);
        this.address = address;
        this.text = text;
    }

    public EmailEvent(Object source) {
        super(source);
    }

    public String getAddress() {
        return address;
    }

    public void setAddress(String address) {
        this.address = address;
    }

    public String getText() {
        return text;
    }

    public void setText(String text) {
        this.text = text;
    }
}
```

继承 `ApplicationEvent` 抽象类。

定义 `EmailListener` 监听器：

```java
@Configuration、
//ApplicationListener是可以给出泛型的  只监听此类事件
public class EmailListener implements ApplicationListener<EmailEvent> {
    private Logger log = LoggerFactory.getLogger(EmailListener.class);

    @Override
    public void onApplicationEvent(EmailEvent emailEvent) {
        if(Objects.nonNull(emailEvent)) {
            log.info("邮件地址：{}", emailEvent.getAddress());
            log.info("邮件内容：{}", emailEvent.getText());
            log.info("source对象为:{}",emailEvent.getSource());
        }
    }
}
```

在启动类中发布该事件：

```java

@SpringBootApplication
public class SpringlistenerApplication implements CommandLineRunner, ApplicationContextAware {

    public static void main(String[] args) {
        SpringApplication.run(SpringlistenerApplication.class, args);
    }

    private ApplicationContext applicationContext;

    @Override
    public void run(String... args) throws Exception {
        EmailEvent event = new EmailEvent("hello", "qxnekoo@163.com", "ze ze ze");
        //主动触发该事件  需要 ApplicationContext 对象来发布
        applicationContext.publishEvent(event);
    }

    @Override
    //获取 Spring 的 applicationContext 容器
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }
}
```

运行结果：

![Spring.png](http://www.qxnekoo.cn:8888/images/2020/04/19/Spring.png)

还有一种监听的方式就是不使用泛型，全都监听

```java
@Configuration
public class CommonListener implements ApplicationListener {
    private Logger log = LoggerFactory.getLogger(CommonListener.class);
    @Override
    public void onApplicationEvent(ApplicationEvent applicationEvent) {
        log.info("{}",applicationEvent.getClass().getName());
        //通过不同的事件类型进行不同的逻辑处理
        if(applicationEvent instanceof ContextRefreshedEvent) {
            log.info("初始化Bean完成");
        } else if(applicationEvent instanceof EmailEvent) {
            log.info("邮箱事件触发");
        }
    }
}
```

运行结果

![Spring2.png](http://www.qxnekoo.cn:8888/images/2020/04/19/Spring2.png)

两种方式按照需求选择就可以了。

>额外补充
>
>```java
>//key位 beanName，value为bean
>Map<String, AppContextInitListener> result = applicationContext().getBeansOfType(AppContextInitListener.class);
>
>//返回 beanName 的String 数组
>String[] result = applicationContext().getBeanNamesForType(Interface.class);
>```





​	









