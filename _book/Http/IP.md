# IP、TCP、DNS

## 负责传输的 IP 协议

IP(Internet Protocol)网际协议位于网络层。

IP 协议的作用是把各种数据包传送给对方。而要保证确实传到对方那里，则需要满足各类条件。其中最重要的两个条件就是 **IP** 地址和 **MAC** 地址（Media Access Control Address）。

IP 地址指明了节点被分配到的地址，MAC 地址是指网卡所属的固定地址。IP 地址可以和 MAC 地址进行配对。IP 地址可变换，但 MAC 地址基本不会更改。

使用 **ARP** 协议凭借 **MAC** 地址进行通信。

IP 间的通信依赖 MAC 地址。<u>在网络上，通信的双方在同一局域网内的情况是很少的，通常是经过多台计算机和网络设备中转才能连接到对方。而在中转时，会利用下一站中转设备的 MAC 地址来搜索下一个中转目标，这时会采用 ARP 协议（Address Resolution Protocol）</u>。**ARP 是一种用以和解析地址的协议，根据通信方的 IP 地址就可以反查出对应的 MAC 地址。**

没有人能够全面掌握互联网中的传输状况，在到达通信目标前的中转过程中，那些计算机和路由器等网络设备只能获悉很粗略的传输路线。

这种机制称为路由选择（routing），有点像快递公司的送货过程。

![IP.png](http://www.qxnekoo.cn:8888/images/2020/05/23/IP.png)

## 确保可靠性的 TCP

TCP 位于传输层，提供可靠的字节流服务。

> 字节流服务（byte stream service）：为了方便传输，将大块数据分割成以报文段(segment)为单位的数据包进行管理。而可靠的传输服务是指，能够把数据准确可靠地传给对方。
>
> TCP 协议为了更容易传送大数据才把数据分割，而且 TCP 协议能够确认数据与最终是否送达到对方。

TCP 采用了三次握手（three-way handshaking）策略。用 TCP 协议把数据包送出去后，TCP 不会对传送后的情况置之不理，它一定会向对方确认是否成功送达。握手过程中使用了 TCP 的标志（flag）——SYN(synchronize) 和 ACK（acknowledgement）。

**发送端首先发送一个带 SYN 标志的数据包给对方。接受端收到后，回传一个带有 SYN/ACK 标志的数据包以示传达确认信息。最后，发送端再回传一个带 ACK 标志的数据包，代表“握手”结束。**

**若在握手过程中某个阶段莫名中断，TCP 协议会再次以相同的顺序发送相同的数据包。**

除了三次握手，TCP 还有其他方式来保证通信的可靠。

![109205ed29cead827c25afec5871c543.png](http://www.qxnekoo.cn:8888/images/2020/05/23/109205ed29cead827c25afec5871c543.png)

## 域名解析的 DNS

DNS(Domain Name System)服务是和 HTTP 协议一样位于应用层的协议。它提供域名到 IP 地址之间的解析服务。

计算机既可以被赋予 IP 地址，也可以被赋予主机名和域名。比如 www.hackr.jp。 

DNS 协议提供通过域名查找 IP 地址，或逆向从 IP 地址反查域名的服务。

![DNS.png](http://www.qxnekoo.cn:8888/images/2020/05/23/DNS.png)

了解下 IP 协议、TCP 协议和 DNS 服务在使用 HTTP 协议的通信过程中各自发挥了哪些作用。

![39192277e2b72ffd4a00f49b4661ae15.png](http://www.qxnekoo.cn:8888/images/2020/05/24/39192277e2b72ffd4a00f49b4661ae15.png)

## URI 和 URL

URI(Uniform Resource Identifier)：统一资源标识符

URL(Uniform Resource Locator)：统一资源定位符

- Uniform

  规定统一的格式可方便处理多种不同类型的资源，而不用根据上下文环境来识别资源指定的访问方式。另外，加入新增的协议方案（如 http: 或 ftp:）也更容易。

- Resource

  资源的定义是“可标识的任何东西”。除了文档文件、图像或服务（例如当天的天气预报）等能够区别于其他类型的，全都可作为资源。另外，资源不仅可以是单一的，也可以是多数的集合体。

- Identifier

  表示可标识的对象。也称为标识符。

URI 就是由某个协议方案表示的资源的定位标识符。协议方案是指访问资源所使用的协议类型名称。采用 HTTP 协议时，协议方案就是 http。除此之外，还有 ftp、mailto、telnet、file 等。

URI 用字符串标识某一互联网资源，而 URL 表示资源的地点（互联网上所处的位置）。可见 URL 是 URI 的子集。



























