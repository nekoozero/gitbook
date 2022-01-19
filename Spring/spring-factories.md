# spring.factories

类似于JAVA SPI的机制。

> 当服务的提供者，提供了服务接口的一种实现之后，在jar包的META-INF/services/目录里同时创建一个以服务接口命名的文件。该文件里就是实现该服务接口的具体实现类。而当外部程序装配这个模块的时候，就能通过该jar包META-INF/services/里的配置文件找到具体的实现类名，并装载实例化，完成模块的注入。 基于这样一个约定就能很好的找到服务接口的实现类，而不需要再代码里制定。jdk提供服务实现查找的一个工具类：java.util.ServiceLoader

而Spring boot则是在 META-INF/spring.factories文件中配置接口的实现类名称，然后在程序中读取配置文件并实例化。

## 实现原理

spring-core包里定义了SpringFactoriesLoader类，这个类实现了检索META-INF/spring.factories文件。

**loadFactories** 根据接口类获取其实现类的实例，这个方法返回的是对象列表。 

**loadFactoryNames** 根据接口获取其接口类的名称，这个方法返回的是类名的列表。 

**loadSpringFactories **根据Spring.factories文件获得接口与其实现类们（可能有多个实现类），返回一个Map<接口,List<实现类名>>

![image-4.png](../myimage/image-4.png)

常量FACTORIES_RESOURCE_LOCATION就是"META-INF/spring.factories"

result.computeIfAbsent就是将配置文件中的配置放入到Map<配置属性名，List<配置结果>>中

![image-5.png](../myimage/image-5.png)

在日常工作中，我们可能需要实现一些SDK或者Spring Boot Starter给被人使用时， 

 我们就可以使用Factories机制。Factories机制可以让SDK或者Starter的使用只需要很少或者不需要进行配置，只需要在服务中引入我们的jar包即可。

