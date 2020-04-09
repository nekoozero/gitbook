---
title: java中的序列化
date: 2019-05-20 10:29:13
categories: 编程基础
summary: 简单记录一下序列化和transient
---

## 序列化

* 什么是序列化

为了保存在内存中的各个对象的状态，并且可以把保存的对象状态再读出来。保存对象状态的机制，就是序列化。

专业一点： 

序列化就是把Java对象转换为字节序列的过程。

反序列化就是把字节序列恢复为Java对象过程。

* 为什么序列化

1. 当想要把内存中的对象保存到一个文件中或者数据库中
2. 当用套接字在网络上传送对象
3. 通过RMI传输对象的时候

当我们保存地时候不仅仅是保存对象的实例变量的值，JVM还要保存一些销量信息，比如类的类型等以便恢复原来的对象。

* 序列化的方式

如果一个对象想要实现序列化，必须实现下面两个接口之一：

1. Serializable接口
2. Externalizable接口

就不分开详细讲了。

* 使用

实体类

```java
package com.crossoverjie.utils;

import java.io.Serializable;

public class User implements Serializable {
    private  String username;
    private  String password;

    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }
}
```

测试：

```java
package com.crossoverjie.Seriablize;

import com.crossoverjie.utils.User;

import java.io.*;

public class TransientTest {
    public static void main(String[] args) {
        User user  = new User();   /
        user.setUsername("qxnekoo");
        user.setPassword("2222");

        System.out.println("序列化之前：");
        System.out.println("username:" +user.getUsername());
        System.out.println("password:" +user.getPassword());

        try{
            ObjectOutputStream os = new ObjectOutputStream(new FileOutputStream("C:/user.txt"));
            os.writeObject(user);
            os.flush();
            os.close();
        }catch(FileNotFoundException e) {
            e.printStackTrace();
        }catch (IOException e) {
            e.printStackTrace();
        }

        try{
            ObjectInputStream is = new ObjectInputStream(new FileInputStream("C:/user.txt"));
            user = (User) is.readObject();
            is.close();
            System.out.println("序列化之后");
            System.out.println("username:"+user.getUsername());
            System.out.println("password:"+user.getPassword());
        }catch(FileNotFoundException e) {
            e.printStackTrace();
        }catch (IOException e) {
            e.printStackTrace();
        }catch(ClassNotFoundException e) {
            e.printStackTrace();
        }
    }
}
```

结果：

```
序列化之前：
username:qxnekoo
password:2222
序列化之后
username:qxnekoo
password:2222
```

* 注意事项

1. 当父类实现序列化，子类自动实现序列化，不需要显式实现Serializable接口；
2. 当一个对象的实例变量引用其他对象，序列化该对象时也把引用对象进行序列化；
3. 并非所有对象都可以序列化

* serialVersionUID（版本号）

一个对象数据，在反序列化过程中，如果序列化串中的serialVersionUID与当前对象值不同，则反序列化失败，否则成功。

如果serialVersionUID没有显式生成，*系统就会自动构成一个*。**类名、类及其属性修饰符、接口及接口顺序、属性、静态初始化、构造器**，任何一项的改变都会导致serialVersionUID变化。

为了避免这个问题，因为serialVersionUID的不同而导致反序列化失败，一般系统都会要求实现Serializable接口的类显式的声明一个serialVersionUID。

如果我们保持了serialVersionUID的一致，则在反序列化时，对于新增的字段会填入默认值null（int的默认值为0），对于减少的字段则直接忽略。



## transient

一个类的有些属性需要序列化，而其他属性不需要序列化，比如说用户的敏感信息，这些信息对应的变量就可以加上transient关键字。这个字段的声明周期仅存在与调用者的内存中而不会写到磁盘里持久化。

上面的代码中，给password加上关键字transient,得到的结果如下所示：

```
序列化之前：
username:qxnekoo
password:2222
序列化之后
username:qxnekoo
password:null
```

* 注意事项

1. 一旦变量被transient修饰，变量将不再是对象持久化的一部分，该变量内容在序列化后无法获得访问。
2. transient关键字只能修饰变量，而不能修饰方法和类。注意，本地变量是不能被transient关键字修饰的。变量如果是用户自定义类变量，则该类需要下实现Serializable接口。
3. 被transient关键字修饰的变量不再被序列化，一个静态变量不管是否被transient修饰，均不能被序列化。因为静态变量不归属于对象，属于类本身。

