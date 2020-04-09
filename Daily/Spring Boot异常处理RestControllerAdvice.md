---
title: Spring Boot异常处理RestControllerAdvice
date: 2019-05-17 12:17:56
summary: 自定义异常和全局异常处理
categories: Spring Boot
---

## 自定义异常类

```java
package com.nekoo.beichenghuai.exception;

/**
 * 自定义异常类
 * spring 对于 RuntimeException 异常才会进行事务回滚。
 */
public class RRException extends RuntimeException {
    private String msg;
    private int code = 500;
    
    //构造函数
    public RRException(String msg){
        super(msg);
        this.msg = msg;

    }
    public RRException(String msg,int code) {
        super(msg);
        this.msg = msg;
        this.code = code;
    }
    
    public RRException (String msg,Throwable e) {
        super(msg,e);
        this.msg = msg;
    }
    
    public RRException(String msg ,int code,Throwable e) {
        super(msg,e);
        this.msg = msg;
        this.code = code;
    }
    
    //Getter&Setter
    public String getMsg() {
        return msg;
    }

    public void setMsg(String msg) {
        this.msg = msg;
    }

    public int getCode() {
        return code;
    }

    public void setCode(int code) {
        this.code = code;
    }
}

```

## 全局异常处理类

```java
package com.nekoo.beichenghuai.exception;

import com.nekoo.beichenghuai.common.R;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import org.springframework.web.servlet.NoHandlerFoundException;

/**
 * 异常处理器
 */
@RestControllerAdvice
public class RRExceptionHandler {
    /**
     * 处理自定义异常
     * @param e
     * @return
     */
    @ExceptionHandler(RRException.class)
    public R handleRRexception(RRException e) {
        R r = new R();
        r.put("code",e.getCode());
        r.put("msg",e.getMsg());
        return r;
    }
    /**
     * 处理no handler异常
     * @param e
     * @return
     */
    @ExceptionHandler(NoHandlerFoundException.class)
    public R handlerNoFoundException(Exception e) {
        return R.error(404,"接口不存在");
    }
    /**
     * 处理普通异常
     * @param e
     * @return
     */
    @ExceptionHandler(Exception.class)
    public R handleException(Exception e){
        return R.error(500,e.getMessage());
    }
}

```

当出现异常的时候会由这个异常处理器来处理，这里处理了自己的自定义异常，NoHandlerFound和一个普通异常。

测试代码：

```java
@RestController
public class UserController {
    @GetMapping("/error")
    public R error() throws Exception {
        //自定义异常
        throw new RRException("错误了啊",100);

    }
    @GetMapping("/test")
    public R test(){
        //普通异常
        int i = 1/0;
        return R.ok("ok");
    }
}
```

在postman中测试，

error接口：

```json
{
    "msg": "错误了啊",
    "code": 100,
    "message": "success"
}
```



test接口：

```json
{
    "code": 500,
    "message": "/ by zero"
}
```



未知接口，errorsss接口:

```json
{
    "code": 404,
    "message": "接口不存在"
}
```



NoHandlerFoundException这个异常还要在yml中配置：

```yaml
spring:
  mvc:
    throw-exception-if-no-handler-found: true
  resources:
    #禁用资源映射
    add-mappings: false
```

否则，异常处理器不会去处理这个异常。

通过测试发现，springboot的WebMvcAutoConfiguration会默认配置如下资源映射：

> /映射到/static（或/public、/resources、/META-INF/resources） /webjars/ 映射到classpath:/META-INF/resources/webjars/ 
>
> /**/favicon.ico映射favicon.ico文件.

所以，即使地址错误，还是会匹配到/**这个静态资源映射，不会发生NoHandlerFoundException,所以还可以改掉默认的静态资源映射访问路径

```yaml
spring:
  mvc:
    throw-exception-if-no-handler-found: true
    static-path-pattern: /statics/**
```

spring预提供了一个ResponseEntityExceptionHandler，能处理大部分spring自己定义的异常，我们只需要覆盖handleExceptionInternal这个汇总处理方法，将返回的ResponseEntity的body部分替换为我们自己定义的ApiErrorResponse即可。
当然，你也可以自由地配置对其它异常的处理。

详细：https://my.oschina.net/u/3049656/blog/1798583

## 统一接口返回数据

### 继承HashMap

直接上代码

```java
package com.nekoo.beichenghuai.common;

import java.util.HashMap;
import java.util.Map;

/**
 * 返回数据
 */
public class R extends HashMap<String,Object>{

    public R() {
        put("code",200);
        put("message","success");
    }

    /**
     * 返回指定的错误代码和错误信息
     * @param code
     * @param msg
     * @return
     */
    public static R error(int code,String msg) {
        R r = new R();
        r.put("code",code);
        r.put("message",msg) ;
        return r;
    }
    public static R error(){
        return error(500,"未知错误，请联系管理员");
    }



    /**
     * 操作成功直接返回默认信息success
     * @return
     */
    public static R ok() {
        return new R();
    }
    public static R ok(String msg) {
        R r = new R();
        r.put("message",msg);
        return r;
    }
    public static R ok(String msg,int code) {
        R r = new R();
        r.put("message",msg);
        r.put("code",code);
        return r;
    }
    public static R ok(Map<String,Object> map) {
        R r = new R();
        r.putAll(map);
        return r;
    }
    @Override
    public R put(String key,Object value) {
        super.put(key,value);
        return this;
    }
}

```

这个R类继承了HashMap，仔细看还是看得懂的，只有一个构造函数，给code和message赋值，默认都是成功的值；提供了两个错误静态方法，带参数的需要传入错误代码和错误的信息（会覆盖message），不带参数的默认就是未知错误；提供了三个成功静态方法，效果也是差不多，提供自己定义的成功代码和信息。

那么需要的数据怎么出入呢，可以注意到它重写了HashMap的put方法，调用了super的put方法之后，把自己返回出去了，因为HashMap的put方法是返回null的。这么做的目的就是可以方便传递数据：

```java
@GetMapping("/list")
public R user(){
    User u = new User();
    User u1 = new User();
    u1.setUsername("qxnekoo");
    u.setUsername("北城槐");
    return R.ok("操作成功").put("admin",u).put("member",u1);
}
```

在Mapping方法最后可以直接return，也方便了链式添加数据，返回数据如下：

```json
{
    "code": 200,
    "member": {
        "id": 0,
        "username": "qxnekoo",
        "password": null
    },
    "admin": {
        "id": 0,
        "username": "北城槐",
        "password": null
    },
    "message": "操作成功"
}
```

### 自定义

直接上代码，结果类：

```java
package com.company.project.core;

import com.alibaba.fastjson.JSON;

/**
 * 统一API响应结果封装
 */
public class Result<T> {
    private int code;
    private String message;
    private T data;

    public Result setCode(ResultCode resultCode) {
        this.code = resultCode.code();
        return this;
    }

    public int getCode() {
        return code;
    }

    public String getMessage() {
        return message;
    }

    public Result setMessage(String message) {
        this.message = message;
        return this;
    }

    public T getData() {
        return data;
    }

    public Result setData(T data) {
        this.data = data;
        return this;
    }

    @Override
    public String toString() {
        return JSON.toJSONString(this);
    }
}

```

工具类：

```java
package com.company.project.core;

/**
 * 响应结果生成工具
 */
public class ResultGenerator {
    private static final String DEFAULT_SUCCESS_MESSAGE = "SUCCESS";

    public static Result genSuccessResult() {
        return new Result()
                .setCode(ResultCode.SUCCESS)
                .setMessage(DEFAULT_SUCCESS_MESSAGE);
    }

    public static <T> Result<T> genSuccessResult(T data) {
        return new Result()
                .setCode(ResultCode.SUCCESS)
                .setMessage(DEFAULT_SUCCESS_MESSAGE)
                .setData(data);
    }

    public static Result genFailResult(String message) {
        return new Result()
                .setCode(ResultCode.FAIL)
                .setMessage(message);
    }
}

```

枚举：

```java
package com.company.project.core;

/**
 * 响应码枚举，参考HTTP状态码的语义
 */
public enum ResultCode {
    SUCCESS(200),//成功
    FAIL(400),//失败
    UNAUTHORIZED(401),//未认证（签名错误）
    NOT_FOUND(404),//接口不存在
    INTERNAL_SERVER_ERROR(500);//服务器内部错误

    private final int code;

    ResultCode(int code) {
        this.code = code;
    }
    public int code() {
        return code;
    }
}

```

其实都是大同小异，这个返回类的比较固定，只有code、message、data三个，工具类也提供了三个方法，虽然少，但可读性高，而且也确实是够用了。



个人比较一下这两个，第一种可以在Api返回的数据添加多个data，比如说查询订单这个需求，可能还要去查询顾客的个人信息和商家的信息，而订单、顾客、商家是三个model类，使用第二个的话需要封装到一个data里面可能会有点繁琐，而使用HashMap则可以一直put。

