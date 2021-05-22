# nacos集群安装

> 时间：2020/4/15

官网：https://nacos.io/zh-cn/index.html

`github` 地址：https://github.com/alibaba/nacos

最好还是养成看官网的习惯！

本次使用的版本是1.2.0，主要是由于 `github` 下载太慢了,`gitee`虽然可以下载下来源码，但在 `windows` 下编译打包后的 `tar.gz` 在 `linux` 环境下启动不了，编译的时候也确实出现了 `error`，不知道在 `linux` 下编译打包会不会好点，有机会会尝试的。

官网：

![nacos.jpg](http://www.qxnekoo.cn:8888/images/2020/04/16/nacos.jpg)

其实翻译出来大概是这样的

![nacos.png](http://www.qxnekoo.cn:8888/images/2020/04/16/nacos.png)

自己安装的时候 `nginx` 和 `mysql` 并没有安装集群。

## nacos集群安装

首先要先将 `nacos` 默认的数据库切换至 `MySQL`。修改 `conf` 目录下的 `application.properties`，在末尾加上连接 `MySQL` 的信息:

```properties
spring.datasource.platform=mysql

db.num=1
db.url.0=jdbc:mysql://192.168.46.128:3307/nacos_config?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true
db.user=root
db.password=root
```

安装了三个 `nacos` 都要切换这个配置。关于数据库的创建，在 `nacos`目录下的 `conf` 下有个 `nacos-mysql.sql` 文件，按照内容创建数据库，在执行里面的 `sql` 初始化表就可以了,结果如下：

![nacosd18beadd5eccbd9f.png](http://www.qxnekoo.cn:8888/images/2020/04/16/nacosd18beadd5eccbd9f.png)

接着配置集群信息，也很简单，在 `conf` 目录下创建 `cluster.conf` 文件，内容如下：

```
192.168.46.129:8848
192.168.46.128:8848
192.168.46.130:8848
```

写清楚各个节点的 `ip` 和端口号即可。

这样 `nacos` 就配置完成了，可以直接在 ` bin` 目录下运行 `sh startup.sh` 命令，`linux` 默认是开启集群模式的。

![nacos80efa86a6ed2c3c1.png](http://www.qxnekoo.cn:8888/images/2020/04/16/nacos80efa86a6ed2c3c1.png)

表示以集群的模式在启动了。

![nacos6845f86f5a58cc8a.png](http://www.qxnekoo.cn:8888/images/2020/04/16/nacos6845f86f5a58cc8a.png)

如上图所示就启动成功了，比较离谱的是大家也看到了，这个节点竟然启动了快4分钟了，中间就是一直在 `Nacos is starting...`，还好当时我没终止启动。

## nginx 配置

在这边由于采用的 `docker` 安装 `Nginx`，所以也拿出来记录一下,还有一些自己接触比较少的配置。

首先是安装

`docker run -v /root/nacos/nginxconf:/etc/nginx/conf.d --name mynginx -p 80:80 -d nginx`

使用的默认 `nginx` 的镜像，它的配置文件在 `/etc/nginx/conf.d` 中的一个 `default.conf` 文件。

插一句

`docker run -v /opt/gitbook:/usr/share/nginx/html --name gitbook -p 8080:80 -d nginx`

默认的显示 `index` 页面是在 `/usr/share/nginx/html` 目录下。

修改 `nginx` 配置文件：

```lua
#这一部分
upstream cluster {
  server 192.168.46.128:8848;
  server 192.168.46.129:8848;
  server 192.168.46.130:8848;
}
server {
    listen       80;
    server_name  localhost;

    #charset koi8-r;
    #access_log  /var/log/nginx/host.access.log  main;

    location / {
      #  root   /usr/share/nginx/html;
       # index  index.html index.htm;
        #还有这个配置
        proxy_pass  http://cluster;
    }

    #error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
       # proxy_pass  http://cluster;
    }

    # proxy the PHP scripts to Apache listening on 127.0.0.1:80
    #
    #location ~ \.php$ {
    #    proxy_pass   http://127.0.0.1;
    #}

    # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
    #
    #location ~ \.php$ {
    #    root           html;
    #    fastcgi_pass   127.0.0.1:9000;
    #    fastcgi_index  index.php;
    #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
    #    include        fastcgi_params;
    #}

    # deny access to .htaccess files, if Apache's document root
    # concurs with nginx's one
    #
    #location ~ /\.ht {
    #    deny  all;
    #}
}
```

默认情况下，`nginx` 按加权轮转的方式将请求分发到各服务器。

之后启动`nginx`，访问 `192.168.46.128:80/nacos`

![b36db005b544dfefe18c2d0bedf72932.png](http://www.qxnekoo.cn:8888/images/2020/04/16/b36db005b544dfefe18c2d0bedf72932.png)

图中的数据是自己加的，默认是没有数据的。表示集群搭建成功。