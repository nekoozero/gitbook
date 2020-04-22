# BeanPostProcessor

> 时间：2020/4/21

先将源码中的注释贴上来，毕竟是翻译的，看的时候最好还是对照英文的来看：

```
工厂挂钩允许自定义修改新的bean实例;例如，检查标记接口或使用代理包装bean。通常，通过标记接口或类似方法填充bean的后处理器将实现{@link# postprocessbeforeinitialize}，而使用代理包装bean的后处理器通常将实现{@link# postprocessafterinitial}。


注册
{@code ApplicationContext}可以在其bean定义中自动检测{@code BeanPostProcessor} bean，并将这些后处理程序应用于随后创建的任何bean。普通的{@code BeanFactory}允许对后处理程序进行编程式注册，将它们应用于通过bean factory创建的所有bean。
```

直译过来，就是对象后处理器。接口中有两个默认方法

```java
package org.springframework.beans.factory.config;

import org.springframework.beans.BeansException;
import org.springframework.lang.Nullable;

public interface BeanPostProcessor {

	@Nullable
	default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}

	@Nullable
	default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}

}

```

可以先写一个实例代码来看看效果，自定义 `BeanPostProcessor`

```java
@Component
public class CustomPostProcessor implements BeanPostProcessor {
    private Logger logger = LoggerFactory.getLogger(CustomPostProcessor.class);

    public CustomPostProcessor() {
        logger.warn("CustomPostProcessor的构造函数");
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        if (bean instanceof Bean4BBP) {
            logger.warn("process after init");
        }
        return bean;
    }

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        if (bean instanceof Bean4BBP) {
            logger.warn("process before init");
        }
        return bean;
    }
}
```

然后创建 `Bean4BBP`，并注册为 `Bean`

```
@Component
public class Bean4BBP {

    private Logger log = LoggerFactory.getLogger(Bean4BBP.class);

    public Bean4BBP () {
        log.warn("Bean4BBp的构造函数");
    }
}
```

结果如下：

![BeanPostProcessor1.png](http://www.qxnekoo.cn:8888/images/2020/04/20/BeanPostProcessor1.png)

好好理解一下：

> 我们在使用 Spring 的时候，拿到的 Bean 其实就是一个对象，那么 Spring 肯定会在之前就去执行类的构造函数，来产生一个对象，那这个对象能被 Spring 直接拿过来用嘛？
>
> Spring 会对其进行额外的操作，就是将其初始化（InitializingBean 接口）为归 Spring 管理的 Bean，BeanPostProcessor 并不是是用来做这个事情的，但是会在对象被初始化之前、之后执行相应的回调函数，差不多是一种帮助其初始化的作用吧。

在对象创建完成之后，会去遍历 `BeanPostProcessor` 对象，然后去为普通 `Bean` 执行里面的 `postProcessAfterInitialization()` 和 `postProcessBeforeInitialization()`。

下图是Bean的初始化步骤，可以看到 `BeanPostProcessor `

![Bean-Initialization-Steps.jpg](http://www.qxnekoo.cn:8888/images/2020/04/20/Bean-Initialization-Steps.jpg)

看几个比较 `Spring` 自己的 `BeanPostProcessor `，

## 实例

上一篇说到的 `applicationListener`，它也是一个 `Bean`

1. `ApplicationListenerDetector.java`

![BeanPostProcessor_applicationListener.png](http://www.qxnekoo.cn:8888/images/2020/04/20/BeanPostProcessor_applicationListener.png)

2. `AbstractAutoProxyCreator.java`

   这个就有说道了，主要是 `Spring AOP` 和这个有关联。`AspectJAwareAdvisorAutoProxyCreator.java` 和 `AnnotationAwareAspectJAutoProxyCreator`，都是继承这个类，看名字其实知道是自动代理的创建类，而我们知道 `Spring AOP` 最主要的就是通过代理对象来调用目标方法，代理对象就是在这边生成的。

   > 这边也解释了我之前的一个疑问:我注入一个 Service 层的 Spring 的 Bean，然后获取这个 Bean 的类名（也就是看看它属于哪个类），得到的是 Service 实现类的类名;而在 Service 实现类上加上一个 @Transactional 注解，获取到的是一个由 `Spring` 创建的代理类的类名，下面会演示。

   也就是说在`AbstractAutoProxyCreator.java` 中的这个 `BeanPostProcessor` 调用了 `postProcessAfterInitialization()` 方法，这个里面会调用一个 `wrapIfNecessary` 方法，我们打上断点，

   ![BeanPostProcessor_AOP1.png](http://www.qxnekoo.cn:8888/images/2020/04/20/BeanPostProcessor_AOP1.png)

   撰写一个测试类：`TestServiceImpl.java`，但是并不声明 `@transactional`,打印它的类名：

   ```java
   logger.info("注入对象的类名：{}",testService.getClass().getName());
   ```

     运行结果，并没有进入断点，而且 `Spring` 也并没有为其创建代理对象：

   ![BeanPostProcessor_AOP2.png](http://www.qxnekoo.cn:8888/images/2020/04/20/BeanPostProcessor_AOP2.png)

   加上`@transactional`，重新运行，进入到断点，看到 `Spring` 为我们创建的代理对象，你品，你细品……：

   ![BeanPostProcessor_AOP5.png](http://www.qxnekoo.cn:8888/images/2020/04/20/BeanPostProcessor_AOP5.png)

   之后放开断点看结果：

   ![BeanPostProcessor_AOP6.png](http://www.qxnekoo.cn:8888/images/2020/04/20/BeanPostProcessor_AOP6.png)

   这就呼应上了，该对象就是 `Spring` 为其创建的对象，完事儿。    

   因为 `@Transactional` 注解就是使用的 `Spring AOP`，其实你只要为这个类中的方法做个 `Spring AOP`，注入的时候产生的对象就是 `Spring` 的代理对象，这也就解释了我们平常做 `Spring AOP`的时候为什么一定要去用代理对象来调用目标方法，而不能单纯用 `this`。



栗子就举到这儿吧，以后学习到了再来补充，感觉 `BeanPostProcessor` 还是怪好玩儿的！！