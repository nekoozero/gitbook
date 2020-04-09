# swagger2的简单使用

1. 引进pom

```xml
        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-swagger-ui</artifactId>
            <version>2.9.2</version>
        </dependency>

       <dependency>
            <groupId>io.springfox</groupId>
           <artifactId>springfox-swagger2</artifactId>
            <version>2.9.2</version>
      </dependency>
```

2. 撰写配置类

   ```java
   package cn.nekoo.swagger.configuration;
   
   import org.springframework.context.annotation.Bean;
   import org.springframework.context.annotation.Configuration;
   import springfox.documentation.builders.ApiInfoBuilder;
   import springfox.documentation.builders.PathSelectors;
   import springfox.documentation.builders.RequestHandlerSelectors;
   import springfox.documentation.service.ApiInfo;
   import springfox.documentation.spi.DocumentationType;
   import springfox.documentation.spring.web.plugins.Docket;
   import springfox.documentation.swagger2.annotations.EnableSwagger2;
   
   @Configuration
   @EnableSwagger2
   public class Swagger4Configuration {
       //哪些包下面的接口会产生api 除了被 @ApiIgnore 指定的请求
       public static final String SWAGGER_SCAN_BASE_PACKAGE = "cn.nekoo.swagger.controller";
       //版本号
       public static final String VERSION = "1.0.0";
   
       @Bean
      // select() 函数返回一个 ApiSelectorBuilder实例用来控制哪些接口暴露给 Swagger 来展现
       public Docket createApi() {
           return new Docket(DocumentationType.SWAGGER_2)
                   .apiInfo(apiInfo()).select()
     .apis(RequestHandlerSelectors.basePackage(SWAGGER_SCAN_BASE_PACKAGE))
                           .paths(PathSelectors.any()) // 可以根据url路径设置哪些请求加入文档，忽略哪些请求
                           .build();
       }
   
       /**
        * 创建该文档的基本信息
        * @return
        */
       private ApiInfo apiInfo() {
           return new ApiInfoBuilder()
                   .title("测试服务")  //文档主题
                   .description("测试服务接口文档")   //文档描述
                   .version(VERSION)//文档版本
                   .termsOfServiceUrl("http://www.baidu.com") //设置文档的License信息->1.3 License information
                   .build();
       }
   
   }
   
   ```

3. 使用注解

   * @Api: 修饰整个类，描述Controller的作用

   * @ApiOperation: 描述一个类的一个方法，或者说一个接口

   * @ApiParam: 单个参数描述

   * @ApiModel: 用对象来接受参数，使用在model类中

   * @ApiProperty: 用对象接受参数时，描述类的一个字段，作用于类的字段上

   * @ApiResponse: HTTP响应其中1个描述

   * @ApiResponses: HTTP响应整体描述

   * @ApiIgnore: 使用该注解忽略这个API

   * @ApiError : 发生错误返回的信息

   * @ApiImplicitParam: 描述一个请求参数，可以配置参数的中文含义，还可以给参数设置默认值

   * @ApiImplicitParams: 描述由多个 @ApiImplicitParam 注解的参数组成的请求**参数列表**(@ApiImplicitParam的数组）

     具体做法与示例：https://cloud.tencent.com/developer/article/1454887

4. 其他

   @ApiImplicitParam注解的dataType、paramType两个属性的区别？

   > dataType="int" 代表请求参数类型为int类型，当然也可以是Map、User、String等；
   >
   > paramType="body" 代表参数应该放在请求的什么地方：
   >
   >     header-->放在请求头。请求参数的获取：@RequestHeader(代码中接收注解)
   >     query-->用于get请求的参数拼接。请求参数的获取：@RequestParam(代码中接收注解)
   >     path（用于restful接口）-->请求参数的获取：@PathVariable(代码中接收注解)
   >     body-->放在请求体。请求参数的获取：@RequestBody(代码中接收注解)
   >     form  任何@RequestParam注解修饰的字段，都可以用FormData来传，可以说比query更加推荐，但是swagger不用form是因为swagger-ui.html还不能发送formData
   > 一般的get请求又用query，post请求用body
   >
   > 上传文件的时候才会用form类型