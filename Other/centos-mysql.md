# 安装mysql

[原文链接](http://blog.java1234.com/blog/articles/308.html)

1. 进入mysql官网获取RPM包[下载地址](https://dev.mysql.com/downloads/repo/yum/)，右击 复制链接地址 https://dev.mysql.com/get/mysql57-community-release-el7-11.noarch.rpm

2. * wget：`yum -y install wget`

   - 下载rpm文件` wget https://dev.mysql.com/get/mysql57-community-release-el7-11.noarch.rpm`
   - 安装MySQL源`yum -y localinstall mysql57-community-release-el7-11.noarch.rpm `
   - 在线安装mysql`yum -y install mysql-community-server`
   
   

3. * mysql服务`systemctl start mysqld`
   * 开机启动`systemctl enable mysqld`，`systemctl daemon-reload`

   - 登陆，在本机/var/log/mysqld.log中给root一个随机密码，我们可以到那边查看
   - 登陆之后修改密码`ALTER USER 'root'@'localhost' IDENTIFIED BY 'Caofeng2012@';`



4. * 设置远程登陆（服务器的话不建议这么做）`GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'Caofeng2012@' WITH GRANT OPTION;`，允许数据库这么做。
   * ` firewall-cmd --zone=public --add-port=3306/tcp --permanent`，永久开启防火墙端口3306。
   * `firewall-cmd --reload`，刷新一下

5. * 设置默认编码，修改/etc/my.cnf,在[mysqld]下添加编码配置。`character_set_server=utf8` `init_connect='SET NAMES utf8'`
   * 重启mysql服务

   







