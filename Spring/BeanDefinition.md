# BeanDefinition

源码中的注释，结合英文看：

```
bean定义描述了一个bean实例，该实例具有属性值、构造函数参数值和由具体实现提供的进一步信息。

这只是一个最小的接口:主要目的是允许{@link BeanFactoryPostProcessor}内省和修改属性值和其他bean元数据。
```

`BeanDefinition` 是 `Spring` 对 `Bean` 的抽象。容器中每一个 `bean` 都会有一个对应的 `BeanDefinition` 实例，该实例负责保存 `bean` 对象的所有必要信息，包括 `bean` 对象的 `class` 类型、是否抽象类、构造方法、参数、其他属性等。是用来描述 `Bean` 的。

![BeanDefinition.png](http://www.qxnekoo.cn:8888/images/2020/04/21/BeanDefinition.png)

- AttributeAccessor

  提供了访问属性的能力。就是定义了对对象属性的一些访问方法。

- BeanMetadataElement

  只有一个方法 `Object getSource()`，用来获取元数据元素的配置源对象。`AttributeAccessorSupport`是唯一抽象实现，内部基于`LinkedHashMap`实现了所有的接口，供其他子类继承使用  主要针对属性CRUD操作。这个方法在`@Configuration`中使用较多，因为它会被代理。

源码：

```java
package org.springframework.beans.factory.config;

import org.springframework.beans.BeanMetadataElement;
import org.springframework.beans.MutablePropertyValues;
import org.springframework.core.AttributeAccessor;
//BeanDefinition描述了一个bean实例，该实例具有属性值、构造函数参数值和由具体实现提供的进一步信息。
//这只是一个最小的接口:主要目的是允许BeanFactoryPostProcessor内省和修改属性值和其他bean元数据。
public interface BeanDefinition extends AttributeAccessor, BeanMetadataElement {
    String SCOPE_SINGLETON = "singleton";
    String SCOPE_PROTOTYPE = "prototype";
    int ROLE_APPLICATION = 0;
    int ROLE_SUPPORT = 1;
    int ROLE_INFRASTRUCTURE = 2;

    void setParentName(String var1);

    String getParentName();
    // 设置 Bean 的类名称，将来是要通过反射来生成实例的
    void setBeanClassName(String var1);
    // 获取 Bean 的类名称
    String getBeanClassName();

    void setScope(String var1);

    String getScope();

    void setLazyInit(boolean var1);

    boolean isLazyInit();

    void setDependsOn(String... var1);

    String[] getDependsOn();

    void setAutowireCandidate(boolean var1);

    boolean isAutowireCandidate();

    void setPrimary(boolean var1);

    boolean isPrimary();
    //有些实例不是用反射生成的，而是用工厂模式生成的
    void setFactoryBeanName(String var1);

    String getFactoryBeanName();

    void setFactoryMethodName(String var1);

    String getFactoryMethodName();
    //获取构造器参数
    ConstructorArgumentValues getConstructorArgumentValues();
    //Bean 中的属性值，后面给 bean 注入属性值的时候会说到
    MutablePropertyValues getPropertyValues();

    boolean isSingleton();

    boolean isPrototype();

    boolean isAbstract();

    int getRole();

    String getDescription();

    String getResourceDescription();

    BeanDefinition getOriginatingBeanDefinition();
}
```

`BeanDefinition`仅仅是一个最简单的接口，主要功能是允许`BeanFactoryPostProcessor` 例如`PropertyPlaceHolderConfigure` 能够检索并修改属性值和别的bean的元数据