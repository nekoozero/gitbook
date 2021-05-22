# 安装Docker

> 时间: 2020/2/20

官方网站(我的宿主机系统为centos)
https://docs.docker.com/install/linux/docker-ce/centos/
## 设置仓库
1. 安装必要的包
   
   `$ sudo yum install -y yum-utils \
    device-mapper-persistent-data \
    lvm2`

2. 设置一个稳定的仓库
   
   `$sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo`

## 安装Docker引擎
1. 安装最新的Docker Engine - Community，或者看第二步选择其他版本

   `sudo yum install docker-ce docker-ce-cli containerd.io`
   
2. 也可以选择具体的版本，先展示版本
   
   `yum list docker-ce --showduplicates | sort -r`

   下载

   `sudo yum install docker-ce-<VERSION_STRING> docker-ce-cli-<VERSION_STRING> containerd.io`

   注意，此时docker已经安装完成，但是并没有启动，而且 docker 的用户组也创建出来了，但是没有创建用户。

3. 启动docker
   
   `sudo systemctl start docker`
   
4. 运行一个简单的demo
   
   `sudo docker run hello-world`

5. 配置镜像加速器，编辑/etc/docker/daemon.json文件

   ```json
   {
       "registry-mirrors": [
           "https://registry.docker-cn.com"
       ]
   }
   ```

   也可以使用aliyun的镜像加速器。

   [简单教程](https://www.cnblogs.com/salmonLeeson/p/11610139.html)

   配置完成之后执行 systemctl daemon-reload 和 systemctl restart docker

## 安装docker-compose

国内的源下载会快很多

```curl -L https://get.daocloud.io/docker/compose/releases/download/1.25.0/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose```



```sudo chmod +x /usr/local/bin/docker-compose```

## 数据卷相关

1. 查看所有数据卷

   ```docker volume ls```

2. 查看某个数据卷的详细内容

   ```docker volume inspect 数据卷名 ```

   