# rpm 和 yum

## rpm

### 介绍

一种用于互联网下载包的打包及安装工具，它包含在某些Linux分发版中。它生成 具有.RPM扩展名的文件。RPM是RedHat Package Manager（RedHat软件包管理工 具）的缩写，类似windows的setup.exe，这一文件格式名称虽然打上了RedHat的 标志，但理念是通用的。Linux的分发版本都有采用（suse,redhat, centos 等等），可以算是公认的行业标 准了。

### rpm管理

* 简单查询指令 查询已安装的rpm列表 (qa 是查询所有的意思 query all)

  ```rpm -qa|grep xx```

   一个rpm包名：firefox-45.0.1-1.el6.centos.x86_64.rpm 名称:firefox 版本号：45.0.1-1 适用操作系统: el6.centos.x86_64 表示centos6.x的64位系统 如果是i686、i386表示32位系统，noarch表示通用。。

* 其他查询指令
  * rpm -qa | more               分页查看
  * rpm -q 软件包名              查询软件包是否安装 
  * rpm -qi 软件包名             查询软件包信息 
  * rpm -ql 软件包名             查询软件包中的文件 （比如查询和MySQL相关的文件和目录）

### rpm卸载

* 语法

  rpm -e 包名

  **如果其它软件包依赖于您要卸载的软件包，卸载时则会产生错误信息。** 

  removing these packages would break dependencies:foo is needed by bar-1.0-1 

   如果我们就是要删除 foo这个rpm 包，可以增加参数 --nodeps ,就可以强制删除，但是一 般不推荐这样做，因为依赖于该软件包的程序可能无法运行 如：

  $ rpm -e --nodeps foo 

### rpm安装

* 语法

  rpm -ivh  RPM包全路径名称 

  参数说明：

  i = install 安装

  v = verbose 提示

  h = hash 进度条

## yum

### 介绍

Yum 是一个Shell前端软件包管理器。基于RPM包管理，能够从指定 的服务器自动下载RPM包并且安装，可以自动处理依赖性关系，**并且一次安装所有依赖的软件包。**

### 基本指令

* 查询yum服务器是否有需要安装的软件

  yum  list |grep XXX

* 安装指定的yum包

  yum install XXX

