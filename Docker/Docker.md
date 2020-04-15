# Dokcer 的基本组成

> 时间: 2019/10/20

* Docker Client 客户端
* Docker Daemon 守护进程
* Docker Image 镜像
* Docker Container 容器
* Docker Registry 仓库

## Docker 客户端/守护进程

C/S 架构

本地/远程

通过 Docker 客户端执行 docker 命令（docker pull 、docker run），将这些命令发给守护进程，守护进程执行完后返回结果给客户端。

## Docker Image镜像

容器的基石

层叠的只读文件系统

联合加载（union mount）

结构如下

> add Apache
>
> add emacs
>
> rootfs(Ubuntu)       基础镜像
>
> bootfs

## Docker Container 容器

通过镜像启动

启动和执行阶段

写时复制（copy on write）

> 可写层
>
> add Apache
>
> add emacs
>
> rootfs(Ubuntu)       基础镜像
>
> bootfs

容器是镜像创建的运行实例。



## 总结

我们把应用程序和配置依赖打包好形成一个可交付的运行环境，这个打包好的运行环境就是image文件。只有通过镜像文件才能生成Docker容器。image文件可以看作是容器的模板。Docker根据image文件生成容器的实例。**同一个image文件，可以生成多个同时运行的容器实例**。

* image 文件生成的容器实例，本身也是一个文件，成为镜像文件。

* 一个容器运行一种服务，当我们需要的时候，就可以通过docker客户端创建一个对应的运行实例，也就是我们的容器。
* 仓库就是存放了一堆image文件的东西。



# 常见命令

## 镜像命令

* docker images : 显示本地所有镜像
* docker images  -a : 列出本地所有镜像（含中间映像层）
* docker images -q : 只显示镜像ID
* docker images --digests : 显示摘要信息
* docker search ...  : 在docker hub上查找镜像
  * docker search --filter=stars=30 ... : 在docker hub上查找starts超过30的某个镜像

* docker pull ... : 拉去镜像
* docker rmi [-f] ... : 删除镜像[强制]

## 容器命令

一个镜像可以创建多个容器实例。

* docker run [options] 
  * --name "容器名字"：为容器指定一个名称
  * -d : 后台运行容器，并返回容器id，也即启动守护式容器
  * **-i : 以交互模式启动容器，通常与 -t 同时使用**
  * **-t : 为容器重新分配一个为输入终端，通常与 -i 同时使用**
  * -P : 随机端口映射
  * -p : 指定端口映射，有以下四种格式
    - ip:hostPort:containerPort
    - ip::containerPort
    - **hostPort:containerPort**
    - containerPort

* 退出方式：

  * exit 容器停止退出
  * ctrl+p+q 容器不停止退出

* 停止容器：docker stop 容器id或容器名   docker kill 容器id或容器名（强制性的）

* 显示容器： docker ps  ：显示正在运行的容器

  ​                    docker ps -a ：显示所有运行过的容器

  ​                    docker ps -l  ：显示上一个运行的容器

  ​                    docker ps -n 2 ：显示上两个运行的容器

* 删除容器： docker rm 容器id ： 删除指定的容器

* 重启容器： docker restart 容器id ：重启容器



docker run -d centos 以后台模式（守护进程）启动一个容器，然后使用docker ps会发现**容器已经退出去了**。

*docker容器后台运行，就必须有一个前台进程。*容器如果不是那些一直挂起的命令（比如top，tail），就是会自动退出的。

解决方案，将要运行的容器以前台进程的形式运行。



docker run -d centos /bin/sh -c "while true;do echo hello zzyy;sleep 2;done"

以后台形式运行docker，并且让centos每两秒输入hello zzyy的日志。

这样docker ps 就可以查看到该容器了，因为该容器一直在运行，没有被docker停止。



* 查看日志： docker logs [options] 容器id

  -t ：是加入时间戳

  -f ：跟随最新的日志打印

  --tail 数字 ：显示最后多少条

* 查看容器内的运行进程： docker top 容器id

* 查看容器的内部细节：docker inspect 容器id 

  会以嵌套的json串的形式描述一下这个容器。

* 重新进入容器：docker attach 容器id

   比如说进入docker run -it centos 之后，ctrl+p+q不停止退出该容器，可以通过docker attach 容器id重新进入该容器。

* 宿主机在容器中执行命令： docker exec -t 容器id 执行的命令

  docker exec -t 80bbab5673d7 ls

  在该容器执行ls的命令，并且自动退出容器（不会停止容器），**会把结果返回给宿主机**

  docker exec -it  80bbab5673d7 /bin/bash 也可以直接重新进入容器(交互的方式)，**要有/bin/bash的参数**

* 在容器内创建的文件，在下次进入该容器的时候还是存在的。

  所以将容器内的文件复制出来： docker cp 容器id:文件路径 宿主机的文件路径

# 镜像相关

## UnionFs

Union文件系统是一种分层、轻量级并且高性能的文件系统，他支持对文件系统的修改作为一次提交来一层层的叠加，同时可以将不同目录挂载到同一个虚拟文件系统下。Union文件系统是Docker镜像的基础。镜像可以通过分层来进行继承，基于基础镜像，可以制作各种具体的应用镜像。

## Docker镜像加载原理

bootfs(boot file system)主要包含bootloader和kernel，bootloader主要是引导加载kernel，Linux杠启动时回家再bootfs文件系统，**在Docker镜像最底层是bootfs**，这一层与我们典型的Linux/Unix系统是一样的，包含boot加载器和内核。当boot加载完成之后整个内核就都在内存当中了，此时内存的使用权已由bootfs转交给内核，此时系统也会卸载bootfs。

rootfs(root file system)，在bootfs之上，包含的就是典型的Linux系统中的/dev,/proc,/bin,/etc等标准目录和文件。rootfs就是各种不同的操作系统发行版，比如Ubuntu，Centos等等。

对于一个精简的OS，rootfs可以很小，只需要包括最基本的命令、工具和程序库就可以了，因为底层直接用Host的kernel，自己只需要提供rootfs就行了。由此可见对于不同的linux发行版，bootfs基本是一致的，rootfs会有差别，因此不用的发行版可以公用bootfs。

分层的联合文件系统特点是可以共享资源。

当我们下载一个tomcat镜像时，会去下载linux的镜像、jdk的镜像、tomcat的镜像等等，也就是说tomcat能供运行的所有环境，所以tomcat的镜像文件会比较大。



## 示例tomcat

docker run -it -p 8888:8080 tomcat

浏览器访问 ip:8888即可

也就是说docker的8888端口映射了tomcat的8080端口



docker run -it -P tomcat

再通过docker ps来查看运行容器的端口即可

### 提交docker镜像

docker commit -a="nekoo" -m="tomcat by me" 9fedfgs atnekoo/mytomcat

就可以在本地看到自己的创建的这个镜像了。



# Docker数据卷

* 将运用与运行的环境打包形成容器运行，运行可以伴随着容器，但是我们对数据的要求希望是**持久化的**。
* 容器之间希望有可能**共享数据**。

Docker容器产生的数据，如果不通过docker commit生成新的镜像，使得数据作为镜像的一部分保存下来，那么当容器被删除后，数据自然也就没有了。

为了能保存数据，在docker中我们使用卷（容器数据卷）。

特点：

1. 数据卷可以在容器之间共享或重用数据。
2. 卷中的更改可以直接生效。
3. 数据卷中的更改不会包含在镜像的更新中。
4. 数据卷的生命周期一致持续到没有容器使用它位为止。

## 命令添加

- docker run -it -v /宿主机绝对路径目录:/容器内目录 镜像名

这里是创建一个容器实例并且关联数据卷，只是这个容器关联了，而不是用这个镜像的容器都关联了。

docker inspect 容器id 

可以查看到相关数据卷的绑定关系



- docker run -it -v /宿主机绝对路径目录:/容器内目录:ro 镜像名

  ro的意思是只读（read only）,也就是说，只能在宿主机上写操作，容器内是可以看到的，但是容器内是无法对这个目录进行任何的写操作。

## Dockerfile添加

```dockerfile
#volume test
FROM centos
VOLUME ["/dataDockerVolum1","/dataDockerVolum2"]
CMD echo "finished,-------successful"
CMD /bin/bash
```

通过命令执行

docker build -f /mydocker/Dockerfile -t nekoo/centos .

-f 指定文件  

-t指定images名字 

后面需要带上一个 .  表示当前路径

输出为：

```
Sending build context to Docker daemon  2.048kB
Step 1/4 : FROM centos
 ---> 0f3e07c0138f
Step 2/4 : VOLUME ["/dataDockerVolum1","/dataDockerVolum2"]
 ---> Running in 9f657a0140f6
Removing intermediate container 9f657a0140f6
 ---> b7d0bea5e9a9
Step 3/4 : CMD echo "finished,-------successful"
 ---> Running in 66b6283bb131
Removing intermediate container 66b6283bb131
 ---> eec469d8f75c
Step 4/4 : CMD /bin/bash
 ---> Running in f1f9351659dd
Removing intermediate container f1f9351659dd
 ---> b09adb41786c
Successfully built b09adb41786c
Successfully tagged nekoo/centos:latest
```

docker images查看镜像：

nekoo/centos        latest              **b09adb41786c**        11 seconds ago      220MB

则成功。

由于没有指定宿主中的数据卷关联的文件夹，需要在启动容器之后通过 docker inspect来查看

```
"Mounts": [
            {
                "Type": "volume",
                "Name": "91989fa82f62c6cf6609703fd627d0639b03fdafeb7fa9af28fb7380c7a20831",
                "Source": "/var/lib/docker/volumes/91989fa82f62c6cf6609703fd627d0639b03fdafeb7fa9af28fb7380c7a20831/_data",
                "Destination": "/dataDockerVolum1",
                "Driver": "local",
                "Mode": "",
                "RW": true,
                "Propagation": ""
            },
            {
                "Type": "volume",
                "Name": "6db5d3f5c65d4206d938514c8525a34560e196ef45fbf41a006b998258ee702e",
                "Source": "/var/lib/docker/volumes/6db5d3f5c65d4206d938514c8525a34560e196ef45fbf41a006b998258ee702e/_data",
                "Destination": "/dataDockerVolum2",
                "Driver": "local",
                "Mode": "",
                "RW": true,
                "Propagation": ""
            }
        ]
```

Source指定的目录即为宿主机中的数据卷目录。

## 数据卷容器

docker run -it --name dc002 --volumes-from dc01 nekoo/centos

以nekoo/centos的镜像开启一个容器，容器的数据卷和dc01的容器数据卷一样。

再开一个dc003,这三个容器中的数据卷内容都是共享且互通的，能互相影响的。

就算*删除*dc001容器，dc002和dc003之间的数据卷共享关系还是会存在的。

总结：**容器之间配置信息的传递，数据卷的生命周期一致持续到没有容器使用它为止。**

# Dockerfile

概念：Dockerfile是用来构建Docker镜像的构建文件，是由一系列命令和参数构成的脚本。

## Dockerfile内容基础知识

1. 每条关键字指令必须为大写字母且后面要跟随至少一个参数
2. 指令按照从上到下，顺序执行
3. #表示注释
4. 每条指令都会常见一个新的镜像层，并对镜像进行提交

## Docker执行Dokcerfile的大致流程

1. docker从基础镜像运行一个容器
2. 执行一条指令并对容器做出修改
3. 执行类似docker commit的操作提交一个新的镜像层
4. dokcer再基于刚提交的镜像运行一个新容器
5. 执行dockerfile中的天下一条指令直到所有指令都执行完成

## 关键字

* FROM

  基础镜像，当前新镜像是基于哪个镜像的

* MAINTAINER

  镜像维护者的姓名和邮箱地址

* RUN

  镜像构建时需要运行的命令

* EXPOSE

  当前容器对外暴露出的端口

* WORKDIR

  指定在创建容器后，终端默认登陆进来的工作目录，一个落脚点

* ENV

  用来再构建镜像过程中设置环境变量

  ENV MY_PATH /usr/mytest

  这个环境变量可以再后续的任何RUN指令中使用，这就如同再命令前面指定了环境变量前缀一样，也可以在其他指令中直接使用这些环境变量

  WORKDIR MY_PATH
  
* ADD

  将宿主机目录下的文件拷贝进镜像且ADD命令会自动处理URL和解压tar压缩包

* COPY

  类似ADD，拷贝文件和目录到镜像中

  将从构建上下文目录中<源路径>的文件/目录复制到新的一层的镜像内的<目标路径>位置

  COPY src dest

  COPY ["src","dest"]

* VOLUME

  容器数据卷，用于数据保存和持久化工作

* **CMD**

  指定一个容器启动时要运行的命令

  Dcokerfile中可以有多个CMD命令，**但只有最后一个生效**，CMD会被docker run之后的参数替换

* ENTRYPOINT

  指定一个容器启动时要运行的命令

  ENTRYPOINT的目的和CMD一样，都是在指定容器启动程序及参数

* ONBUILD

  当构建一个被继承的Dockerfile时运行命令，父镜像在被子继承后父镜像的onbuild被触发

  

总结：

| BUILD         | BOTH    | RUN        |
| ------------- | ------- | ---------- |
| FROM          | WORKDIR | CMD        |
| MAINTAINER    | USER    | ENV        |
| COPY          |         | EXPOSE     |
| ADD           |         | VOLUME     |
| RUN           |         | ENTRYPOINT |
| ONBUILD       |         |            |
| .dockerignore |         |            |

## 案例

1. base镜像（scratch）：Docker Hub上99%的镜像都是通过base镜像中安装和配置需要的软件构造出来的。

2. 

```dockerfile
#CMD
FROM centos
RUN yum install -y curl
CMD ["curl","-s","https://ip.cn"]


##ENTRYPOINT
FROM centos
RUN yum install -y curl
ENTRYPOINT ["curl","-s","https://ip.cn"]
```

ENTRYPOINT的方式可以在启动容器的时候在后面加上参数（追加覆盖）：

 docker run ip2centos -i

而CMD后面的参数会被认为是命令，覆盖掉原来Dockerfile文件中的CMD命令 会报错

这些输出都只是在创建容器的时候触发，而不是启动容器的时候触发

## 自制tomcat9

参考：

```dockerfile
FROM         centos
MAINTAINER    zzyy<zzyybs@126.com>
#把宿主机当前上下文的c.txt拷贝到容器/usr/local/路径下
COPY c.txt /usr/local/cincontainer.txt
#把java与tomcat添加到容器中
ADD jdk-8u171-linux-x64.tar.gz /usr/local/
ADD apache-tomcat-9.0.8.tar.gz /usr/local/
#安装vim编辑器
RUN yum -y install vim
#设置工作访问时候的WORKDIR路径，登录落脚点
ENV MYPATH /usr/local
WORKDIR $MYPATH
#配置java与tomcat环境变量
ENV JAVA_HOME /usr/local/jdk1.8.0_171
ENV CLASSPATH $JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
ENV CATALINA_HOME /usr/local/apache-tomcat-9.0.8
ENV CATALINA_BASE /usr/local/apache-tomcat-9.0.8
ENV PATH $PATH:$JAVA_HOME/bin:$CATALINA_HOME/lib:$CATALINA_HOME/bin
#容器运行时监听的端口
EXPOSE  8080
#启动时运行tomcat
# ENTRYPOINT ["/usr/local/apache-tomcat-9.0.8/bin/startup.sh" ]
# CMD ["/usr/local/apache-tomcat-9.0.8/bin/catalina.sh","run"]
CMD /usr/local/apache-tomcat-9.0.8/bin/startup.sh && tail -F /usr/local/apache-tomcat-9.0.8/bin/logs/catalina.out
```

自己写：(很奇怪的地方就是一些目录（比如启动tomcat的目录脚本、java_home的环境变量的配置）没写错，但是容器里面就是无法识别，说找不到目录，我复制到这个Dockerfile文件中才解决了)

```dockerfile
FROM centos
MAINTAINER qxnekoo<qxnekoo@163.com>
COPY c.txt /usr/local/container.txt
ADD jdk-8u191-linux-x64.tar.gz /usr/local/
ADD apache-tomcat-9.0.26.tar.gz /usr/local/
RUN yum -y install vim
ENV MYPATH /usr/local
WORKDIR $MYPATH
ENV JAVA_HOME /usr/local/jdk1.8.0_191/
ENV CLASSPATH $JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
ENV CATALINA_HOME /usr/local/apache-tomcat-9.0.26
ENV CATALINA_BASE /usr/local/apache-tomcat-9.0.26
ENV PATH $PATH:$JAVA_HOME/bin:$CATALINA_HOME/lib:$CATALINA_HOME/bin
EXPOSE 8080
# ENTRYPOINT ["/usr/local/apache-tomcat-9.0.26/bin/startup.sh" ]
# CMD ["/usr/local/apache-tomcat-9.0.26/bin/catalina.sh","run"]
CMD /usr/local/apache-tomcat-9.0.26/bin/startup.sh && tail -F /usr/local/apache-tomcat-9.0.26/logs/catalina.out
```

## 运行MySQL命令

docker run -p 3306:3306 --name mysql -v /opt/docker-mysql/mysql/conf:/etc/mysql/conf.d -v /opt/docker-mysql/mysql/logs:/logs -v /opt/docker-mysql/mysql/data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=qxnekoo -d  mysql:5.7



## 运行zookeeper

```
docker run --name zk -p 2181:2181 -p 2888:2888 -p 3888:3888 --restart always -d zookeeper
```

　    2181　　Zookeeper客户端交互端口

　　2888　　Zookeeper集群端口

　　3888　　Zookeeper选举端口

最好指定容器卷：

/data

/datalog

/logs

## 问题

1. 启动的一些软件容器，比如说zookeeper，没有vi编辑器

需要输入命令 apt-get update 这个命令的作用是：同步 /etc/apt/sources.list 和 /etc/apt/sources.list.d 中列出的源的索引，这样才能获取到最新的软件包。

apt-get install vim命令即可





## docker-compose

* 搭建zookeeper集群

  ```yaml
  version: '2'
  services:
      zoo1:
          image: zookeeper
          restart: always
          container_name: zoo1
          ports:
              - "2181:2181"
          environment:
              ZOO_MY_ID: 1
              ZOO_SERVERS: server.1=zoo1:2888:3888 server.2=zoo2:2888:3888 server.3=zoo3:2888:3888
   
      zoo2:
          image: zookeeper
          restart: always
          container_name: zoo2
          ports:
              - "2182:2181"
          environment:
              ZOO_MY_ID: 2
              ZOO_SERVERS: server.1=zoo1:2888:3888 server.2=zoo2:2888:3888 server.3=zoo3:2888:3888
   
      zoo3:
          image: zookeeper
          restart: always
          container_name: zoo3
          ports:
              - "2183:2181"
          environment:
              ZOO_MY_ID: 3
              ZOO_SERVERS: server.1=zoo1:2888:3888 server.2=zoo2:2888:3888 server.3=zoo3:2888:3888
  ```

  