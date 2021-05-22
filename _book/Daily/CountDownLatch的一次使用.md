---
title: CountDownLatch的一次使用
date: 2019-07-21 10:11:41
categories: 编程应用
---

# 背景

> 时间: 2019/7/21

闲着无聊，用spring boot先做一个简单的小应用，通过[一言api](https://hitokoto.cn/api)，请求数据返回到命令行，通过http-client请求。总的来说很简单，并且也可以发送多个请求来接受数据。

之后，纯粹只是想到了，就想着用着多线程来发送请求，可能不会增加程序的效率，但为了练练手，就硬着头皮上了。

## 一顿操作

先看一下spring管理的一些bean：

```java
package com.qxnekoo.yiyan.bean;
import com.google.common.util.concurrent.ThreadFactoryBuilder;
import org.apache.http.client.methods.HttpGet;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache.http.impl.client.HttpClientBuilder;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.stereotype.Component;
import java.util.concurrent.*;

@Component
@ConfigurationProperties(prefix = "yiyan")
public class Configuration {
   //url属性通过在application.properties中进行赋值 用ConfigurationProperties注解
   //并且还有要getter/setter方法
   private String url;
   private int corePoolSize=2;
   private int maxPoolSize=2;
    @Bean  //httpClient注册为bean
    public CloseableHttpClient httpClient() {
        CloseableHttpClient httpClient = HttpClientBuilder.create().build();
        return httpClient;
    }
    @Bean  //httpGet注册为bean
    public HttpGet httpGet() {
        HttpGet httpGet = new HttpGet(url);
        return httpGet;
    }
    @Bean //ExecutorService注册为bean
    public ExecutorService executorService() {
        ThreadFactory namedThreadFactory = new ThreadFactoryBuilder().setNameFormat("get-yiyan-%d").build();
        ExecutorService executor = new ThreadPoolExecutor(corePoolSize,maxPoolSize,0L, TimeUnit.MILLISECONDS,new LinkedBlockingQueue<>(),
                namedThreadFactory,new ThreadPoolExecutor.AbortPolicy());
        return executor;
    }
    
    public String getUrl() {
        return url;
    }
    public void setUrl(String url) {
        this.url = url;
    }
}
```

namedThreadFactory使用的是google的并发包里的类，主要功能是给我们的线程进行统一的命名，并且通过线程池来创建线程。

接下里是线程类：

```java
package com.qxnekoo.yiyan.Thread;
import com.qxnekoo.yiyan.model.Yiyan;
import com.qxnekoo.yiyan.service.RequestService;
import org.apache.http.client.methods.HttpGet;
import org.apache.http.impl.client.CloseableHttpClient;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;


public class YiYanTask implements  Runnable{
    RequestService requestService;
    CloseableHttpClient httpClient;
    HttpGet httpGet;
    @Override
    public void run() {
        Yiyan yiyan = requestService.getResponse(httpClient,httpGet);
        System.out.println(yiyan.getHitokoto()+"  --"+yiyan.getFrom());
        System.out.println();
    }
    public YiYanTask(RequestService requestService, CloseableHttpClient httpClient, HttpGet httpGet) {
        this.requestService = requestService;
        this.httpClient = httpClient;
        this.httpGet = httpGet;
    }
}
```

提供了一个构造方法，里面传入RequesetService以及我们上面说到的两个bean，真正的任务是通过requesetService来获取Yiyan对象并且输入内容。接下来看看RequesetService的实现类：

```java
package com.qxnekoo.yiyan.service.impl;

import com.alibaba.fastjson.JSON;
import com.qxnekoo.yiyan.YiyanApplication;
import com.qxnekoo.yiyan.model.Yiyan;
import com.qxnekoo.yiyan.service.RequestService;
import org.apache.http.HttpEntity;
import org.apache.http.ParseException;
import org.apache.http.client.ClientProtocolException;
import org.apache.http.client.methods.CloseableHttpResponse;
import org.apache.http.client.methods.HttpGet;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache.http.util.EntityUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;

import java.io.IOException;
@Service  //由Spring容器管理
public class RequestServiceImpl implements RequestService {
    Logger logger = LoggerFactory.getLogger(RequestServiceImpl.class);
    @Override
    //通过函数传入httpClient和httpGet对象
    public Yiyan  getResponse( CloseableHttpClient httpClient,HttpGet httpGet) {  
        CloseableHttpResponse response = null;
        try {
            // 由客户端执行(发送)Get请求
            response = httpClient.execute(httpGet);
            // 从响应模型中获取响应实体
            HttpEntity responseEntity = response.getEntity();
           // System.out.println();
            logger.info("响应状态为:{}"+ response.getStatusLine());
            if (responseEntity != null) {
                String res = EntityUtils.toString(responseEntity);
                //fastJson进行解析
                Yiyan yiyan = JSON.parseObject(res,Yiyan.class);
                return yiyan;
            }
        } catch (ClientProtocolException e) {
            logger.error(e.getMessage());
        } catch (ParseException e) {
            logger.error(e.getMessage());
        } catch (IOException e) {
            logger.error(e.getMessage());
        } finally {
            if (response != null) {
                try {               //如果在这边也将httpClient关闭会怎样
                    response.close();
                } catch (IOException e) {
                    logger.error(e.getMessage());
                }
            }
            YiyanApplication.LATCH.countDown(); //线程让CountDownLatch减1
        }
        return null;
    }
}
```

这边其实一开始不是这么写的，一开始，我是在这个Service中注入了httpClient和httpGet两个Bean，而不是作为getResponse函数的参数传入进来的，还把当前Service注入到了YiYanTask中，想通过他直接调用，结果发现这样做的话httpClient、httpGet、requestService都是空的（通过debug调试），也就是说完全没有注入进来，查询资料也就有了一个结论：

* **其他线程不能使用主线程的Bean。**

所以上面的requestService、httpClient、httpGet我都是通过YiYanTask的构造函数传进来的，也就是说先由主线程注入bean，之后再由各个线程来使用他们。

写完之后在运行，发现有一条或者两条请求成功了，但之后就报错，说什么connection pool closed，一开始我还以为是线程池出现问题了，但仔细研究过发现，其实是httpClient的问题。

分析：

httpClient作为客户端，使用完之后都是要关闭的，所以一开始我把httpClient和上面的response放到一起关闭了，但是我前面也说到，我在主线程中注入httpClient的bean来供多线程来使用，当有一个线程使用完了，就会在getResponse函数中关闭（之前我都是在这里面关闭的），关闭了之后其他的线程就无法使用了，所以才会说连接被关闭了。

所以就有了新的问题：在多线程使用单例的情况下，怎么保证在所有线程都使用完了之后再对这个单例进行的操作。这边就是一个同步操作。

于是想到了CountDownLatch，Spring Boot运行类中：

```java
package com.qxnekoo.yiyan;

import com.qxnekoo.yiyan.Thread.YiYanTask;
import com.qxnekoo.yiyan.service.RequestService;
import org.apache.http.client.methods.HttpGet;
import org.apache.http.impl.client.CloseableHttpClient;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Profile;
import javax.annotation.Resource;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;

@SpringBootApplication
public class YiyanApplication implements CommandLineRunner {
    //使用java并发包中的CountDownLatch工具
    public static final  CountDownLatch LATCH= new CountDownLatch(4);

    @Resource
    ExecutorService executorService;
    @Resource
    RequestService requestService;
    @Resource
    CloseableHttpClient httpClient;
    @Resource
    HttpGet httpGet;
    public static void main(String[] args) {
        SpringApplication.run(YiyanApplication.class, args);
    }
    @Override
    public void run(String... args) throws Exception {
        System.out.println();
        for (int i = 1; i <= 4; i++) {
            executorService.execute(new YiYanTask(requestService,httpClient,httpGet));
        }
       executorService.shutdown();
        LATCH.await();  //主线程到这边会停住，等待CountDownLatch为0继续执行
        if (httpClient != null) {
            httpClient.close();
        }
    }
}
```

这样主线程就会等待所有线程使用完httpClient之后，再去关闭httpClient。这样也就达到了我们的目的。

# 总结

之前也学过CountDownLatch，但一直都没有用到过，这次也算是为我解决了问题，记录一下。