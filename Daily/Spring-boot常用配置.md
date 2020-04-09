---
title: Spring boot常用配置
date: 2019-05-26 15:01:10
summary: 一些常用的配置 为了记录 防止用的时候到处找
categories: Spring Boot
---



## 跨域

在Spring Boot新建配置类WebMvcConfigurer，继承WebMvcConfigurationSupport 。代码配置：

```java
//解决跨域问题
@Override
public void addCorsMappings(CorsRegistry registry) {
    registry.addMapping("/**")
        .allowedOrigins("*")
        .allowedMethods("POST", "GET", "PUT", "OPTIONS", "DELETE")
        .maxAge(3600)
        .allowedHeaders("*")
        .allowCredentials(true);

}
```

前台发送请求的时候加上http。

## 驼峰转换

数据库中的字段有带下划线，比如说user_email，对应的model中的属性为userEmail，在mybatis中获取出来的时候会获取不到，需要做一下驼峰转换，在mybatis中的配置类配置SqlSessionFactory时

```java
@Bean
public SqlSessionFactory sqlSessionFactoryBean(DataSource dataSource) throws Exception {
    SqlSessionFactoryBean factory = new SqlSessionFactoryBean();
    org.apache.ibatis.session.Configuration configuration = new org.apache.ibatis.session.Configuration();
    //主要是这个配置 下划线和驼峰命名映射
    configuration.setMapUnderscoreToCamelCase(true);
    factory.setConfiguration(configuration);
    factory.setDataSource(dataSource);
    return factory.getObject();
}
```

## FastJson循环检测

阿里的FastJson默认情况下在获取的数据中会启用循环引用检测，即如果从数据库中获取的数据以List接受，如果List里的Object的属性有相同的，FastJson会将其以引用的形式使用ref，这时需要我们禁用掉该配置,在配置类中WebMvcConfigurer配置

```java
/使用阿里 FastJson 作为JSON MessageConverter
@Override
public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
    FastJsonHttpMessageConverter converter = new FastJsonHttpMessageConverter();
    FastJsonConfig config = new FastJsonConfig();
    //禁止循环引用检测
    config.setSerializerFeatures(SerializerFeature.DisableCircularReferenceDetect);
    // config.setSerializerFeatures(SerializerFeature.WriteMapNullValue);//保留空的字段
    //SerializerFeature.WriteNullStringAsEmpty,//String null -> ""
    //SerializerFeature.WriteNullNumberAsZero//Number null -> 0
    // 按需配置，更多参考FastJson文档哈

    converter.setFastJsonConfig(config);
    converter.setDefaultCharset(Charset.forName("UTF-8"));
    converter.setSupportedMediaTypes(Arrays.asList(MediaType.APPLICATION_JSON_UTF8));
    converters.add(converter);
}
```

参考资料：

https://www.jianshu.com/p/50fe2b473cae

https://www.cnblogs.com/luckyyi/p/7999175.html

## 上传文件

默认上传文件有限制大小，有些情况下需要改，依旧在配置类上：

```java
@Bean
public MultipartConfigElement multipartConfigElement() {
    MultipartConfigFactory factory = new MultipartConfigFactory();
    //单个文件最大
    factory.setMaxFileSize("102400KB"); //KB,MB 100m
    /// 设置总上传数据总大小
    factory.setMaxRequestSize("1024000KB");   //1000m
    return factory.createMultipartConfig();
}
```

这里在举一个简单上传文件的例子

```java
@PostMapping("/uploadfile")
public Result uploadfile(@RequestParam("file") MultipartFile file,String openid,String filename){

    Date now = new Date();
    SimpleDateFormat f = new SimpleDateFormat("yyyy-MM-ddHHmmss");
    String date = f.format(now);
    String path = "c://qxnekoo/nginx/html/"+openid+"/"+date+"/";
    // 这里使用Apache的FileUtils方法来进行保存
    try {
        FileUtils.copyInputStreamToFile(file.getInputStream(), new File(path, filename));
        return ResultGenerator.genSuccessResult();
    } catch (IOException e) {
        e.printStackTrace();
        return ResultGenerator.genFailResult("上传失败");
    }
}
```



## PageHelper的简单使用



```java
 @Bean
    public SqlSessionFactory sqlSessionFactoryBean(DataSource dataSource) throws Exception {
        SqlSessionFactoryBean factory = new SqlSessionFactoryBean();
        org.apache.ibatis.session.Configuration configuration = new org.apache.ibatis.session
                .Configuration();
        configuration.setMapUnderscoreToCamelCase(true);
        factory.setConfiguration(configuration);
        factory.setDataSource(dataSource);
        factory.setTypeAliasesPackage(MODEL_PACKAGE);


        //配置分页插件，详情请查阅官方文档
        PageHelper pageHelper = new PageHelper();
        Properties properties = new Properties();
        properties.setProperty("pageSizeZero", "true");//分页尺寸为0时查询所有纪录不再执行分页
        properties.setProperty("reasonable", "true");//页码<=0 查询第一页，页码>=总页数查询最后一页
        properties.setProperty("supportMethodsArguments", "true");//支持通过 Mapper 接口参数来传递分页参数
        pageHelper.setProperties(properties);

        //添加pageHelper插件
        factory.setPlugins(new Interceptor[]{pageHelper});

        //添加XML目录
        ResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();
        factory.setMapperLocations(resolver.getResources("classpath:mapper/*.xml"));
        return factory.getObject();
    }
```

在Controller中使用起来也很简单

```java
@PostMapping("/lists")
/**
     * 
     * @param page 查询第几页的数据
     * @param size 一页查询多少数据
     * @return
     */
public Result lists(@RequestParam(defaultValue = "0") Integer page, @RequestParam(defaultValue = "0") Integer size) {
    //加上这个就可以了
    PageHelper.startPage(page, size);
    List<ShopApply> list = shopApplyService.findAll();
    return ResultGenerator.genSuccessResult(list);
}
```



## Mybatis配置动态SQL

一般在Mybatis的xml文件中可以写sql语句，包括动态sql，mybatis都有相应的xml标签。

实际上也可以在Mapper中，也就是DAO中直接写sql，动态sql也是可以写的。

```java
@SelectProvider(type = ShopApplyCus.class, method = "getShopApplyCus")
List<ShopApply> getShopApplyCus(@Param("status") String status);

class ShopApplyCus {
    //这里的@Param注解必须加上
    public String getShopApplyCus(@Param("status") String  status) {
        String sql = "SELECT * FROM shop_apply";
        if(!status.equals("4")){
            sql += " where status = #{status}";
        }
        return sql;
    }
}

@Select("select * from shop where shop_name like  concat(concat('%',#{shopname}),'%') and opreation_status=1")
List<Shop> getLiked(@Param("shopname") String search);
```



## mybatis配置一对一、一对多

资料：

https://blog.csdn.net/chpllp/article/details/81939793

https://blog.csdn.net/m0_37779570/article/details/81514757

通常在xml文件中配置一对一、一对多关系，其实在查询过程中也是可以直接配置的。

一个需求：查询订单，同时也要查询出订单的顾客的一些信息，显然订单信息和顾客信息是放在两个表中的，在以订单为主体的时候，订单和顾客的关系是一对一的。希望可以在执行一次查询就可以查出结果，当然发起多次http请求来获取也是可以的。

这里是查询出所有的订单，并且查询出每个订单的顾客信息。这里使用FastJson的时候可能会出现循环检测问题，解决方案在上面。

```java
@Select("select * from print_order,wx_user where shop_id=#{shopid} and status=#{status} and wx_user.openid=print_order.wx_openid")
@Results(
    {
       @Result(column="wx_openid", property="wxuser", one=@One(select="com.company.project.dao.WxuserMapper.selectByopenid"))
    }
)
List<Order> getListByShop(@Param("shopid") String shopid,@Param("status") int status);
```

wx_openid是订单表中顾客表的外键，wxuser是订单实体类中的一个属性，类型为Wxuser，同时会指定一个查询去查询顾客信息，参数就是wx_openid，查询出的结果会被映射到wxuser中去。

@One表示一对一关系，一对多是@Many。

在@Table注解的实体类中，如果有些字段不在数据库中，要用@Transient来修饰。



要是还有的话以后会继续补充……

