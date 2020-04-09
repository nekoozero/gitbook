---
title: token的简单使用
date: 2019-05-17 21:49:32
summary: 自己记录的一套流程
categories: Spring Boot
---

## 导包

```xml
<!-- JWT核心依赖 -->
<dependency>
    <groupId>com.auth0</groupId>
    <artifactId>java-jwt</artifactId>
    <version>3.3.0</version>
</dependency>
<!-- java开发JWT的依赖jar包。 -->
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt</artifactId>
    <version>0.9.0</version>
</dependency>
```

## token工具类

```java
package com.nekoo.beichenghuai.common;


import com.auth0.jwt.JWT;
import com.auth0.jwt.JWTVerifier;
import com.auth0.jwt.algorithms.Algorithm;
import com.auth0.jwt.interfaces.DecodedJWT;

import java.io.UnsupportedEncodingException;
import java.util.Date;
import java.util.HashMap;
import java.util.Map;

/**
 * 生成token（加密）
 */
public class TokenGenerator {
    //private static final long EXPIRE_TIME=15*60*1000;

    //设置过期时间为30秒，实际过程中可以设置更久一点
    private static final long EXPIRE_TIME=30*1000;
    /**
     * 秘钥 加密和解密
     */
   private static final String TOKEN_SECRET = "BeiChengHuai&Qxnekoo";

    /**
     * 生成token claim只放入一个username
     * @param username
     * @return
     */
    public static String sign(String username) throws UnsupportedEncodingException {
        Date date = new Date(System.currentTimeMillis()+EXPIRE_TIME);

        try {
            Algorithm algorithm = Algorithm.HMAC256(TOKEN_SECRET);
            Map<String ,Object> header = new HashMap<>(2);
            header.put("typ","JWT");
            header.put("alg","HS256");
            return JWT.create().withHeader(header).withClaim("username",username)
                                .withExpiresAt(date)
                                .sign(algorithm);
        } catch (UnsupportedEncodingException e) {
            throw e;
            //e.printStackTrace();
            //return null;
        }
    }
    /**
     * 通过已有的token进行解密
     * @param token
     * @return
     */
    public static boolean verify(String token) {

        Algorithm algorithm = null;
        try {
            algorithm = Algorithm.HMAC256(TOKEN_SECRET);
            JWTVerifier verifier = JWT.require(algorithm).build();
            //如果解密码失败，这里会出错，将其抛出
            DecodedJWT jwt = verifier.verify(token);
            return true;
        } catch (UnsupportedEncodingException e) {
            return false;
        }


    }
}
```

## 拦截器

```java
package com.nekoo.beichenghuai.interceptor;

import com.nekoo.beichenghuai.common.TokenGenerator;
import com.nekoo.beichenghuai.exception.RRException;
import org.springframework.stereotype.Component;
import org.springframework.web.servlet.HandlerInterceptor;
import org.springframework.web.servlet.ModelAndView;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

@Component
public class TokenIntercepter implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {

        //登录和退出不需要进行登录过期验证  直接放行
  if(request.getRequestURI().indexOf("login")!=-1||request.getRequestURI().indexOf("logout")!=-1){
            return true;
        }
        //通过消息头发送的Authorization来获取token
        String token =  request.getHeader("Authorization");
        try{
            //如果token为空或者解密失败 就抛出错误异常
            if(token!=null&&TokenGenerator.verify(token)){
                return true;
            }
            throw new RRException("登录已过期，请重新登录",418);
        }catch (Exception e){
            throw new RRException("登录已过期，请重新登录",418);
        }
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {

    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {

    }
}
```

配置拦截器：

```
package com.nekoo.beichenghuai.config;

import com.nekoo.beichenghuai.interceptor.TokenIntercepter;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurationSupport;

@Configuration
public class WebMvcConfigurer extends WebMvcConfigurationSupport {
    @Autowired
    private TokenIntercepter tokenIntercepter;
    //添加拦截器
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        //检测token的拦截器
        registry.addInterceptor(tokenIntercepter).addPathPatterns("/**");
    }
}
```

## 使用

简单的模拟登陆：

```java
@GetMapping("/login")
public R login(String username) throws UnsupportedEncodingException {
    String token = TokenGenerator.sign(username);
    return R.ok("获取成功").put("token",token);
}
@GetMapping("/list")
public R user(){
    User u = new User();
    u1.setUsername("qxnekoo");
    return R.ok("操作成功").put("admin",u);
}
```

postman直接请求一般接口/list，返回结果如下：

```json
{
    "msg": "登录已过期，请重新登录",
    "code": 418,
    "message": "success"
}
```

请求接口/login

```json
{
    "code": 200,
    "message": "获取成功",
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE1NTgxMDIyMTF9.GMc-4eQhyDWLCsdKZ0EvAltlSvmvxtghDumJglPM1eY"
}
```

在发起请求的时候，需要在header中添加Authorization属性，`Authorization:"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE1NTgxMDIyMTF9.GMc-4eQhyDWLCsdKZ0EvAltlSvmvxtghDumJglPM1eY"`,请求结果如下：

```json
{
    "code": 200,
    "admin": {
        "id": 0,
        "username": "北城槐",
        "password": null
    },
    "message": "操作成功"
}
```

因为设置的过期时间为30秒，30秒后再次请求该接口：

```json
{
    "msg": "登录已过期，请重新登录",
    "code": 418,
    "message": "success"
}
```

因为该流程是自己研究的，虽然也达到了目的，但应该不是标准做法，这边只是简单的记录，希望早日学到标准的流程。

而且有个小问题，用户登录两次，分别有了两个token，按理说生成了第二个token之后，第一个token应该失效，但是，拿着第一个token还是可以访问数据。

自己想了一个解决办法：前台请求接口的时候不要在header消息头中添加额外的认证信息（也就不存在用户那拿着第几个token的问题了），首先用户登录的时候将token存入到用户表中，当用户请求其他接口的时候，拦截器中从数据库中取出这个token再进行认证，未过期就放行，过期了就让用户重新登录，依次反复。这个存在的问题是，确实不要在头消息中添加认证信息，但每个接口都需要传入一个用户的id来获取token。

