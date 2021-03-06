# Geek

## 面向对象

- 面向对象的四大特性：封装、抽象、继承、多态
- 面向对象编程与面向过程编程的区别和联系
- 面向对象分析、面向对象设计、面向对象编程接口和抽象类的区别以及各自的应用场景
- 基于接口而非实现编程的设计思想
- 多用组合少用继承的设计思想
- 面向过程的贫血模型和面向对象的充血模型

**面向对象编程**：是一种编程范式或编程风格，以类或对象作为组织代码的基本单元，并将封装、抽象、继承、多态四个特性，作为代码设计和实现的基石 。

**面向对象编程语言**：支持类或对象的语法机制，并有现成的语法机制，能方便地实现面向对象编程四大特性（封装、抽象、继承、多态）的编程语言。

> 面向对象编程一般使用面向对象编程语言来进行，但是，不用面向对象编程语言，我们照样可以进行面向对象编程。反过来讲，即便我们使用面向对象编程语言，写出来的代码也不一定是面向对象编程风格的，也有可能是面向过程编程风格的。

**面向对象分析和面向对象设计**：面向对象分析就是要搞清楚做什么，面向对象设计就是要搞清楚怎么做。**两个阶段最终的产出是类的设计**，包括程序被拆解为哪些类，每个类有哪些属性方法、类与类之间如何交互等等。

### 封装

封装也叫做信息隐藏或者数据访问保护。类通过暴露有限的访问接口，授权外部仅能通过类提供的方式来访问内部信息或数据。它需要变成语言提供权限访问控制语法来支持，例如 java 中的 private、protected、public 关键字。封装特性存在的意义，**一方面是保护数据不被随意修改，提高代码的可维护性；另一方面是仅暴露有限的必要接口，提高类的易用性。**

### 抽象

封装主要讲如何隐藏信息、保护数据，那抽象就是讲如何隐藏方法的具体实现，让使用者只需要关心方法提供了哪些功能，不需要知道这些功能是如何实现的。抽象可以通过接口类或者抽象类来实现，但也并不需要特殊的语法机制来支持。抽象存在的意义，**一方面是提高代码的可扩展性、维护性，修改实现不需要改变定义，减少代码的改动范围；另一方面，它也是处理复杂系统的有效手段，能有效地过滤掉不必要关注的信息。**

### 继承

继承是用来表示类之间的 is-a 关系，分为两种模式：单继承和多继承。单继承表示一个子类只继承一个父类，多继承表示一个子类可以继承多个父类。为了实现继承这个特性，编程语言需要提供特殊的语法机制来支持。继承主要是用来解决代码复用的问题。

### 多态

多态是指子类可以替换父类，在实际的代码运行过程中，调用子类的方法实现。多态这种特性也需要编程语言提供特殊的语法机制来实现，比如继承、接口类、duck-typing。多态可以提高代码的扩展性和复用性，是很多设计模式、设计原则、编程技巧的代码实现基础。

### 什么是面向过程编程？什么是面向过程编程语言？

理解这两个概念最好的方式是跟面向对象编程和面向对象编程语言进行对比。相较于面向对象编程以**类为组织代码**的基本单元，面向过程编程则是以**过程（或方法）作为组织代码**的基本单元。它最主要的特点就是数据和方法相分离。相较于面向对象编程语言，面向过程编程语言最大的特点就是不支持丰富的面向对象编程特性，比如继承、多态、封装。

### 面向对象编程相比面向过程编程有哪些优势？

- 对于大规模复杂程序的开发，程序的处理流程并非单一的一条主线，而是错综复杂的网状结构。面向对象编程比起面向过程编程，更能应对这种复杂类型的程序开发。

- 面向对象编程相比面向过程编程，具有更加丰富的特性（封装、抽象、继承、多态）。利用这些特性编写出来的代码，更加易扩展、易复用、易维护。

- 从编程语言跟机器打交道的方式的演进规律中，我们可以总结出：面向对象编程语言比起面向过程编程语言，更加人性化、更加高级、更加智能。

  > C语言因商业应用成熟要比面向对象的编程语言早。操作系统虽然是用面向过程的C语言实现的 但是其设计逻辑是面向对象的。
  > C语言没有类和对象的概念，但是用结构体（struct）同样实现了信息的封装，内核源码中也不乏继承和多态思想的体现。
  > 面向对象思想，不局限于具体语言。

## 设计原则

对于每一种设计原则，我们需要掌握它的设计初衷，能解决哪些编程问题，有哪些应用场景。只有这样，我们才能在项目中灵活恰当地应用这些原则。

常用的设计原则：

- SOLID 原则 - SRP 单一职责原则
- SOLID 原则 - OCP 开闭原则
- SOLID 原则 - LSP 里氏替换原则
- SOLID 原则 - ISP 接口隔离原则
- SOLID 原则 - DIP 依赖倒置原则
- DRY 原则、KISS 原则、YAGNI 原则、LOD 法则

## 设计模式

设计模式是针对软件开发中经常遇到的一些设计问题，总结出来的一套解决方案或者设计思路。大部分设计模式要解决的都是代码的可扩展性问题。

经典的设计模式有 23 种，可以分为三大类：创建型、结构型、行为型。

1. 创建型

   常用：单例模式、工厂模式（工厂方法和抽象工厂）、建造者模式

   不常用：原型模式

2. 结构性

   常用：代理模式、桥接模式、装饰者模式、适配器模式

   不常用：门面模式、组合模式、享元模式

3. 行为型

   常用：观察者模式、模板模式、策略模式、职责链模式、迭代器模式、状态模式

   不常用：访问者模式、备忘录模式、命令模式、解释器模式、中介模式

## 总结

- 面向对象编程因为其具有丰富的特性（封装、抽象、继承、多态），可以实现很多复杂的设计思路，是很多设计原则、设计模式等编码实现的基础。
- 设计原则是指导我们代码设计的一些经验总结，对于某些场景下，是否应该应用某种设计模式，具有指导意义。比如，“开闭原则”是很多设计模式（策略、模板等）的指导原则。
- 设计模式是针对软件开发中经常遇到的一些设计问题，总结出来的一套解决方案或者设计思路。应用设计模式的主要目的是提高代码的可扩展性。从抽象程度上来讲，设计原则比设计模式更抽象。设计模式更加具体、更加可执行。
- 编程规范主要解决的是代码的可读性问题。编码规范相对于设计原则、设计模式，更加具体、更加偏重代码细节、更加能落地。持续的小重构依赖的理论基础主要就是编程规范。
- 重构作为保持代码质量不下降的有效手段，利用的就是面向对象、设计原则、设计模式、编码规范这些理论。

![](https://static001.geekbang.org/resource/image/f3/d3/f3262ef8152517d3b11bfc3f2d2b12d3.png)

















