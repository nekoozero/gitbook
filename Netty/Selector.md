# Selector

> 时间：2020/08/05

## 基本介绍

Java 的 NIO，可以用一个线程，处理多个客户端连接，就会使用到 Selector(选择器)。

Selector 能够检测到多个注册的通道上是否有事件发生（多个 Channel 以事件的方式可以注册到同一个 Selector），如果有事件发生，便获取事件然后针对每个事件进行相应的处理。这样就可以只用单线程区管理多个通道，即管理多个连接和请求。

只有在连接/通道真正有读写事件发生时，才会进行读写，就大大地减少了系统开销，并且不必为每个连接都创建一个线程，不用去维护多个线程。

避免了多线程之间的上下文切换导致的开销。

## 特点说明

1. Netty 的 IO 线程 NioEventLoop 聚合了 Selector（选择器，多路复用器），可以同时并发处理成百上千个客户端连接。
2. 当线程从某客户端 Socket 通道进行读写数据时，若没有数据可用时，该线程可以进行其他任务。
3. 线程通常将非阻塞 IO 的空闲时间用于其他通道上执行 IO 操作，所以单独的线程可以管理多个输入和输出通道。
4. 由于读写操作都是非阻塞的，这就可以充分提升 IO 线程的运行效率，避免由于频繁 IO 阻塞导致的线程挂起。
5. 一个 IO 线程可以并发处理 N 个客户端连接和读写操作，这从根本上解决了传统同步阻塞 IO 已连接一线程模型，架构的性能、弹性伸缩能力和可靠性都得到了极大的提升。

## 相关方法

public static Selector open(); 得到一个选择器对象

public int select(long timeout); 监控所有注册的通道，当其中所有 IO 操作可以进行时，将对应的 SelectionKey 加入到内部集合中并返回，参数用来设置超时时间

public Set\<SelectionKey> selectedKeys();从内部集合中得到所有的SelectionKey

## 注意事项

1. NIO 中的 ServerSocketChannel 功能类似 ServerSocket,SocketChannel 功能类似 Socket。

2. selector.select();//阻塞

   selector.select(1000);//阻塞1000毫秒，在1000毫秒后返回

   selector.wakeup();//唤醒 selector

   selector.selectNow;//不阻塞，立马返回

## 流程分析

![NIO.png](http://www.qxnekoo.cn:8888/images/2020/08/05/NIO.png)

1. 客户端连接时，会通过 ServerSocketChanel 得到 SocketChannel;
2. Selector 进行监听 select 方法，返回有事件发生的通道的个数。
3. 将 socketChannel 注册到 Selector 上，register(Selector sel,int ops)，一个 selector 上可以注册多个 SocketChannel.
4. 注册后返回一个 SelectionKey，会和该 Selector 关联，放到某个集合中。
5. 进一步得到各个 SelectionKey(有事件发生)。
6. 再通过 SelectionKey 反向获取 SocketChannel,方法 channel().
7. 可以通过得到的 channel，完成业务处理。

## 代码演示

服务端：

```java
/**
 * NIO Server端
 * @author qianxin
 * @date 2020/08/05
 */
public class NIOServer {
    public static void main(String[] args) {
        try (ServerSocketChannel serverSocketChannel = ServerSocketChannel.open()) {
            //得到一个 Selector 对象
            Selector selector = Selector.open();
            //serverSocketChannel绑定一个端口6666，在服务器端监听
            serverSocketChannel.socket().bind(new InetSocketAddress(6666));
            //设置为非阻塞
            serverSocketChannel.configureBlocking(false);
            //把 serverSocketChannel 注册到 selector 关心事件为 OP_ACCEPT
            serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
            System.out.println("注册后的 selectionKey 数量：" + selector.keys().size());
            //等待客户端连接
            while (true) {
                //等待1秒 如果这一秒内没有任何事件发生 则进入if 否则继续往下执行
                if (selector.select(1000) == 0) {
                    System.out.println("服务器等待了1秒，无连接！此时selectionKeys数量=" + selector.selectedKeys().size());
                    continue;
                }
                Set<SelectionKey> selectionKeys = selector.selectedKeys();
                System.out.println("有连接过来了，此时selectionKeys数量=" + selectionKeys.size());
                Iterator<SelectionKey> selectionKeyIterator = selectionKeys.iterator();
                while (selectionKeyIterator.hasNext()) {
                    //获取到 SelectionKey
                    SelectionKey selectionKey = selectionKeyIterator.next();
                    if (selectionKey.isAcceptable()) {
                        //生成一个 SocketChannel
                        SocketChannel accept = serverSocketChannel.accept();
                        //将 accept 设置为非阻塞
                        accept.configureBlocking(false);
                        //将 accept 注册到selector,关注事件为 OP_READ,同时给 accept 关联一个 Buffer
                        accept.register(selector, SelectionKey.OP_READ, ByteBuffer.allocate(1024));
                        System.out.println("客户端连接后，注册的selectionKey数量=" + selector.keys().size());
                    }
                    if (selectionKey.isReadable()) {
                        //通过 key 反向获取对应 channel
                        SocketChannel channel = (SocketChannel) selectionKey.channel();
                        //获取到该 channel 关联的 buffer
                        ByteBuffer buffer = (ByteBuffer) selectionKey.attachment();
                        channel.read(buffer);
                        System.out.println("from 客户端" + new String(buffer.array()));
                    }
                    //移除当前的 selectionKye,防止重复操作
                    selectionKeyIterator.remove();
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

客户端：

```java
/**
 * @author qianxin
 * @date 2020/08/05
 */
public class NIOClient {
    public static void main(String[] args) {
        try (SocketChannel socketChannel = SocketChannel.open()) {
            socketChannel.configureBlocking(false);
            InetSocketAddress inetSocketAddress = new InetSocketAddress("127.0.0.1", 6666);
            if (!socketChannel.connect(inetSocketAddress)) {
                while (!socketChannel.finishConnect()) {
                    System.out.println("因为连接需要事件，客户端不会阻塞，可以做其它工作..");
                }
            }
            //连接成功，就发送数据
            String str = "hello ，钱鑫";
            ByteBuffer byteBuffer = ByteBuffer.wrap(str.getBytes());
            socketChannel.write(byteBuffer);
            System.in.read();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

## 补充

1. SelectionKey，表示 Selector 和网络通道的注册关系：

   int OP_ACCEPT:有新的网络连接可以 accept，值为16。(1<<4)

   int OP_CONNECT:代表连接已经简历，值为8。(1<<3)

   int OP_READ:代表读操作，值为1。(1<<0)

   int OP_WRITE:代表写操作，值为4。(1<<2)

2. SelectionKey 相关方法

   ```java
   public abstract Selector selector(); //得到与之关联的 Selector 对象
   public abstract SelectableChannel channel(); //得到与之关联的通道
   public final Object attachment();//得到与之关联的共享数据 buffer
   public abstract SelectionKey interestOps(int ops);//设置或改变监听事件
   public final boolean isAcceptable();//是否可以 accept
   public final boolean isReadable();//是否可以读
   public final booealn isWritable();//是否可以写
   ```

3. ServerSocketChannel

   **在服务器端监听新的客户端 Socket 连接。**

   ```java
   public static ServerSocketChannel open(); //得到一个 ServerSocket通道
   public final ServerSocketChannel bind(SocketAddress local);//设置成服务器端端口号
   public final SelectableChannel configureBlocking(boolean block);//设置阻塞或非阻塞模式，取值 false 表示采用非阻塞模式
   public SocketChannel acept();//接受一个连接，返回代表这个连接的通道对象
   ```

4. SocketChannel

   网络 IO 通道，具体负责进行读写操作。NIO 把缓冲区的数据写入通道，或者把通道里的数据读到缓冲区。

   ```java
   public static SocketChannel open();//得到一个 SocketChannel 通道
   public final SelectableChannel configureBlocking(boolean block);//设置阻塞或非阻塞模式，取值 false 表示采用非阻塞模式
   public boolean connect(SocketAddress remote);//连接服务器
   public boolean finishConnect();//如果上面的方法连接失败，接下来就要通过该方法完成连接操作
   public int write(ByteBuffer src);//往通道里写数据
   public int read(ByteBuffer dst);//从通道里读数据
   public final SelectionKey register(Selector sel,int ops,Object att);//注册一个选择器并设置监听事件，最后一个参数可以设置共享数据
   public final void close();//关闭通道
   ```

   























