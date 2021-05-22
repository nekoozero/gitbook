## Jenkins 安装

> 时间： 2020/5/25

其实也没啥的，都是网上的教程，面向搜索引擎编程！

官方文档：https://www.jenkins.io/zh/doc/book/installing/

我是采用的 Docker 安装

命令：

```shell
 docker run -u root --name jenkins -d -p 8081:8080 -p 50000:50000 -v /var/jenkins_home:/var/jenkins_home -v /var/run/docker.sock:/var/run/docker.sock jenkinsci/blueocean
```

其中的参数官方文档中的解释也很清楚，可以直接去看看。

简单说说遇到的问题吧：

1. 在进行一些比较复杂的操作的时候（比如gitbook install），jenkins 容器直接自动退出了，还有就是有错误的时候（比如使用 github 作为代码管理，因 token 问题访问不到 api），也会自动退出。这个是最棘手的，没办法，只能重启。其实还有不少情况导致重启，使用 Publish Over SSH 插件，操作远程机器，某些命令也导致容器退出了等等。有可能是我的机器配置不行。

2. 因为我这次之使用到了 node，但是貌似没法使用 jenkins 安装的 node 环境，我也不大清楚咋搞，没办法只能到容器里面的虚拟机装了个 node。

3. 最后是github-webhook的使用，push 之后自动构建，有两个参考网站：

   https://www.jianshu.com/p/f90013658c38

   https://blog.csdn.net/qq_36850813/article/details/92782070

   但是按照上面的步骤，我发现 github 在发送请求的时候 403

   https://stackoverflow.com/questions/7427557/jenkins-and-github-webhook-http-403

   简单的来说就是 Payload URL 要加上用户名和密码给 jenkins 认证。例子如下（github-webhook/是必须的）：

   http://xiaohong:xiaohong@www.justsoso.cn:8081/github-webhook/



- 一开始使用的 gitee，但想搞自动化构建，然后Gitee的插件下载失败了，所以就是用了 github，不同的是 github 要配置token，不然会构建失败容器退出（都不能算失败，其实是报错了，构建的记录也找不到），gitee 是不要配置的。
- 想使用 Publish Over SSH 插件把生成的文件传到服务器上（其实是一台服务器上，开了两个 docker 容器），但始终不行，容器老是自动退出，有时候构建的时候退出，有时候构建完了发送文件的时候退出。最后直接把容器的数据卷有用的内容拷过去（在一台服务器上），Publish Over SSH 插件也就用了一个运行 shell的命令，最后重启 gitbook 的那个容器。也有可能是操作、配置有问题，不过试了好多遍，以后有机会再试试吧。