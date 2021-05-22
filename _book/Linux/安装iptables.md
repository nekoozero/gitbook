# iptables

安装相关服务

```undefined
yum install -y iptables
```

```undefined
yum install iptables-services
```

停止firewalld服务

```undefined
systemctl stop firewalld
```

禁用firewalld服务

```undefined
systemctl mask firewalld
```

查看iptables现有规则

```undefined
iptables -L -n
```

允许来自于lo接口的数据包(本地访问)

```undefined
iptables -A INPUT -i lo -j ACCEPT
```

开放22端口

```undefined
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
```

开放21端口(FTP)

```undefined
iptables -A INPUT -p tcp --dport 21 -j ACCEPT
```

允许ping

```go
iptables -A INPUT -p icmp --icmp-type 8 -j ACCEPT
```

设置开机自启动

```bash
systemctl enable iptables.service
```

重启

```systemctl restart iptables.service```



## 开放端口

通过`vim /etc/sysconfig/iptables` 进入编辑增添一条

```
-A INPUT -m state --state NEW -m tcp -p tcp --dport 8080 -j ACCEPT
```

之后重启服务

```systemctl restart iptables.service```