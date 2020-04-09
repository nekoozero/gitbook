# 在docker中安装mysql

其实很简单命令：

```
docker run -p 3307:3306 --name qxMysql
-v /home/qxdocker/mysql/logs:/logs
-v /home/qxdocker/mysql/data:/var/lib/mysql
-v /home/qxdocker/mysql/conf:/etc/mysql/conf.d
-e MYSQL_ROOT_PASSWORD=root -d mysql:5.7
```

运行之后，外部可使用navicate进行连接（端口我这边给的是3307），注意防火墙是否开启。

```
docker exec -it qxMysql /bin/bash
```

再次进入mysql的容器。

