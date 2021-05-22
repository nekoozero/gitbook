# nio 与 netty 的简单示例

> 时间：2020/09/02

简单的 TCP 例子，客户端向服务器发送消息，服务器接受并响应

## nio

服务端：

NioEchoServer

```java
/**
 * Echo 服务端
 *
 * @author qianxin
 * @date 2020/8/27
 */
public class NioEchoServer {
    /**
     * 服务管道，用来向客户端接受或发送数据
     */
    private ServerSocketChannel serverSocketChannel;
    /**
     * 多路复用器，用来向通道注册链接和监听通道事件
     */
    private Selector selector;

    /**
     * 服务端口
     */
    private Integer port;

    public NioEchoServer(Integer port) {
        this.port = port;
        initServer();
    }

    /**
     * 初始化参数管道、多路复用器、注册连接事件
     */
    private void initServer() {
        //开启服务管道
        try {
            serverSocketChannel = ServerSocketChannel.open();
            //开启多路复用选择器
            selector = Selector.open();
            //绑定端口
            serverSocketChannel.socket().bind(new InetSocketAddress(port));
            //设置IO模式为非阻塞
            serverSocketChannel.configureBlocking(false);
            //把管道注册到多路复用选择器上，监听客户端连接事件
            serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public void start() {
        new Thread(new NioSelectorHandler(selector)).start();
        System.out.println("服务启动port=" + port);
    }

    public static void main(String[] args) {
        NioEchoServer myNioServer = new NioEchoServer(9000);
        myNioServer.start();
    }
}
```

NioSelectorHandler

```java
/**
 * @author qianxin
 * @date 2020/8/27
 */
public class NioSelectorHandler implements Runnable {
    private Selector selector;
    private ByteBuffer readBuffer;
    private ByteBuffer writeBuffer;
    public NioSelectorHandler(Selector selector) {
        this.selector = selector;
        this.readBuffer = ByteBuffer.allocate(1024);
        this.writeBuffer = ByteBuffer.allocate(1024);
    }

    @Override
    public void run() {
        while (true) {
            try {
                //开始监听
                int keyNumber = selector.select();
                Set<SelectionKey> selectionKeys = selector.selectedKeys();
                Iterator<SelectionKey> selectionKeyIterator = selectionKeys.iterator();
                SelectionKey selectionKey;
                while (selectionKeyIterator.hasNext()) {
                    System.out.println("一个事件就绪");
                    selectionKey = selectionKeyIterator.next();
                    //处理就绪事件
                    process(selectionKey);
                    selectionKeyIterator.remove();
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    private void process(SelectionKey selectionKey) throws IOException {
        if (!selectionKey.isValid()) {
            return;
        }
        try {
            //处理客户端连接就绪事件
            if (selectionKey.isAcceptable()) {
                System.out.println("与客户端已经建立连接了！！！");
                //通过 selectionKey 获取服务端通道
                //这个serverSocketChannel 就是初始化服务器时的那个serverSocketChannel
                ServerSocketChannel channel = (ServerSocketChannel) selectionKey.channel();
                //服务端获取这个到channel的Socket的连接
                SocketChannel accept = channel.accept();
                accept.configureBlocking(false);
                //连接成功后，往 selector 注册对应通道的 OP_READ 读取事件
                accept.register(selector, SelectionKey.OP_READ);
            }
            //处理数据读取就绪事件
            if (selectionKey.isReadable()) {
                //socketChannel就是连接时accept的channel
                SocketChannel socketChannel = (SocketChannel) selectionKey.channel();
                //将通道中的数据写入到缓存；
                readBuffer.clear();
                int size = socketChannel.read(readBuffer);
                readBuffer.flip();
                if (size > 0) {
                    byte[] bytes = new byte[readBuffer.remaining()];
                    readBuffer.get(bytes);
                    String content = new String(bytes);
                    //
                    System.out.println("收到来自" + socketChannel.getRemoteAddress() + "的消息：" + content);
                    //响应消息客户端消息socketChannel
                    writeBuffer.clear();
                    writeBuffer.put(content.getBytes("UTF-8"));
                    writeBuffer.flip();
                    //回写消息
                    socketChannel.write(writeBuffer);
                    System.out.println("消息已回写");
                } else if (size < 0) {
                    System.out.println("与客户端连接断开");
                    socketChannel.close();
                    selectionKey.cancel();
                } else {
                    return;
                }
            }
        } catch (IOException e) {
            selectionKey.cancel();
            selectionKey.channel().close();
            System.out.println("有一个连接断开了");
        }
    }
}
```

客户端

NioEchoClient

```Java
/**
 * @author qianxin
 * @date 2020/8/27
 */
public class NioEchoClient {
    private SocketChannel socketChannel;

    private Integer port;

    private String host;

    private ByteBuffer readBuffer;

    private ByteBuffer writeBuffer;

    public NioEchoClient(String host, Integer port) {
        this.host = host;
        this.port = port;
        readBuffer = ByteBuffer.allocate(1024);
        writeBuffer = ByteBuffer.allocate(1024);
    }

    public void connect() {
        try {
            socketChannel = SocketChannel.open();
            socketChannel.connect(new InetSocketAddress(host, port));
            BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(System.in));
            while (true) {
                String line = bufferedReader.readLine();
                if (line != null && !line.equals("")) {
                    //向writeBuffer中写数据
                    writeBuffer.put(line.getBytes());
                    writeBuffer.flip();
                    //将 writeBuffer中的数据读到channel中
                    socketChannel.write(writeBuffer);
                    //清空缓存数据
                    writeBuffer.clear();
                    System.out.println("发送数据完毕！！！");
                }
                //从服务端去取数据
                readBuffer.clear();
                //从channel中把数据 写入到 readBuffer中
                int size = socketChannel.read(readBuffer);
                if (size > 0) {
                    readBuffer.flip();
                    byte[] result = new byte[readBuffer.remaining()];
                    //读取
                    readBuffer.get(result);
                    String content = new String(result);
                    System.out.println("服务端响应回来的数据：" + content);
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    public static void main(String[] args) {
        NioEchoClient nioEchoClient = new NioEchoClient("127.0.0.1", 9000);
        nioEchoClient.connect();
    }
}
```

## netty

服务端

NettyServer

```java
/**
 * @author qxnekoo
 * @date 2020/8/26
 */
public class NettyServer {
    public static void main(String[] args) throws InterruptedException {
        //创建 BossGroup 和 workerGroup
        //创建两个线程组 bossGroup 和 workerGroup
        //bossGroup 只是处理连接请求，真正的和客户端业务处理，会交给 workerGroup 完成
        //bossGroup 和 workerGroup 含有的子线程（NioEventLoop）的个数
        //默认实际 cpu 核数 * 2
        //按照我的理解 这 NioEventLoop 其实就是一个单线程的线程池
        //它的继承关系
        // SingleThreadEventLoop -> SingleThreadEventExecutor -> AbstractScheduledEventExecutor
        // ->AbstractEventExecutor -> AbstractExecutorService -> AbstractExecutorService
        // ->AbstractExecutorService 实现了 ExecutorService 接口
        NioEventLoopGroup bossGroup = new NioEventLoopGroup(1);
        NioEventLoopGroup workerGroup = new NioEventLoopGroup();

        try {
            ServerBootstrap serverBootstrap = new ServerBootstrap();
            //设置两个线程组
            serverBootstrap.group(bossGroup, workerGroup)
                    //使用NioServerSocketChannel 作为服务器通道实现
                    .channel(NioServerSocketChannel.class)
                    //设置线程队列得到连接个数
                    .option(ChannelOption.SO_BACKLOG, 128)
                    //设置保持活动连接状态
                    .childOption(ChannelOption.SO_KEEPALIVE, true)
                    //该handler对应bossGroup
                    //.handler(null)
                    //childHandler 对应 workerHandler
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) {
                            ch.pipeline().addLast(new CustomNettyServerHandler());
                        }
                    });
            //绑定一个端口并同步，生成一个 ChannelFuture 对象 启动服务器
            ChannelFuture channelFuture = serverBootstrap.bind(6668).sync();
            channelFuture.channel().closeFuture().sync();
        } finally {
            workerGroup.shutdownGracefully();
            bossGroup.shutdownGracefully();
        }
    }
}
```

自定义服务端处理handler:CustomNettyServerHandler

```java
/**
 * 自定义的一个handler 需要继承 netty 的一个类
 *
 * @author qxnekoo
 * @date 2020/8/26
 */
public class CustomNettyServerHandler extends ChannelInboundHandlerAdapter {
    private static Logger logger = LoggerFactory.getLogger(CustomNettyServerHandler.class);

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        //这边使用的线程是轮询使用的 也就是 workGroup 中的线程
        logger.info("服务器读取数据的线程:{}", Thread.currentThread().getName());
        //当出现非常耗时的操作的时候，我们可以将其异步执行，没有必要在这边等待
        //但需要注意的是，所谓的异步执行也就是将其放入到 nioEventLoop 的 taskQueue 中，也就是等待队列
        //因为 nioEventLoop 实际上是一个单线程的线程池，所以在执行完 下面的 A 代码之后才会去执行该任务
        ctx.channel().eventLoop().execute(() -> {
            try {
                Thread.sleep(10 * 1000);
                logger.info("休眠了10秒钟哦");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        //如果我在这边再加一个耗时 20 秒的任务，那么这个任务其实要等待上面那个耗时 10 秒的任务结束才会执行到，因为是一个单线程的线程池 
        
        
        //schedule会提交一个定时任务 用的也是同一个线程，不过它的队列是 scheduledTaskQueue
        //尽管如此，经过测试发现，同一个eventLoop 会优先执行非定时任务，但定时任务的定时会在后台进行，如果非定时任务结束了，定时也到了，会一股脑儿的去执行定时任务
        //也就是说 对于任务的执行，由于是单线程的 所以哪个先放入等待队列的肯定是优先执行的
        //只不过会先去非定时任务的队列taskQueue去取任务  取完再到定时任务的队列scheduledTaskQueue取任务 定时队列里的顺序和定的时间有关
        // ctx.channel().eventLoop().schedule(() -> {
        //    try {
        //       logger.info("到异步里面了！");
               
        //        logger.info("休眠了5秒钟哦");
        //    } catch (Exception e) {
        //        e.printStackTrace();
        //   }
       //  }, 15, TimeUnit.SECONDS);
        //通道
//        System.out.println(ctx.channel());
        //管道
//        System.out.println(ctx.pipeline());
        //将msg 转成一个 bytebuf
        //A 代码

        ByteBuf bytebuf = (ByteBuf) msg;
        System.out.println("这里是服务器，客户端发送的消息：" + bytebuf.toString(CharsetUtil.UTF_8));

    }

    /**
     * 数据读取完毕之后触发
     */
    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
        //服务端读取完毕之后再响应客户端
        ctx.writeAndFlush(Unpooled.copiedBuffer("hello,客户端", CharsetUtil.UTF_8));
    }

    /**
     * 处理异常  需要关闭通道
     *
     * @param ctx
     * @param cause
     * @throws Exception
     */
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        ctx.close();
    }
}

```

客户端

NettyClient

```java
/**
 * @author qxnekoo
 * @date 2020/8/26
 */
public class NettyClient {
    public static void main(String[] args) throws InterruptedException {
        EventLoopGroup group = new NioEventLoopGroup();
        //使用的是BootStrap而不是ServerBootStrap
        Bootstrap bootstrap = new Bootstrap();
        try {

            bootstrap.group(group)
                    .channel(NioSocketChannel.class)
                    .handler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            ch.pipeline().addLast(new CustomNettyClientHandler());
                        }
                    });
            System.out.println("客户端ok。。");
            //启动客户端去连接服务器
            //ChannelFuture 涉及到 netty'的异步模型
            ChannelFuture channelFuture = bootstrap.connect(
                    "127.0.0.1", 6668
            ).sync();
            //关闭通道是给上监听
            channelFuture.channel().closeFuture().sync();
        } finally {
            group.shutdownGracefully();
        }
    }
}
```

客户端处理handler:CustomNettyClientHandler

```java
/**
 * @author qxnekoo
 * @date 2020/8/26
 */
public class CustomNettyClientHandler extends ChannelInboundHandlerAdapter {
    private static Logger logger = LoggerFactory.getLogger(CustomNettyClientHandler.class);
    /**
     * 当通道就绪的时候会触发该方法
     *
     * @param ctx
     * @throws Exception
     */
    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        ctx.writeAndFlush(Unpooled.copiedBuffer("hello,server:喵", CharsetUtil.UTF_8));
    }

    /**
     * 当通道有读取事件时，会触发
     *
     * @param ctx
     * @param msg
     * @throws Exception
     */
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        ByteBuf buf = (ByteBuf) msg;
        logger.info("此时的线程为：{}",Thread.currentThread().getName());
        System.out.println("这里是客户端，服务器回复的消息" + buf.toString(CharsetUtil.UTF_8));
        System.out.println("服务器的地址：" + ctx.channel().remoteAddress());

    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        ctx.close();
    }
}
```

## 方案说明

1. Netty 抽象出两组线程池 group，每个 group 中的线程池都是单线程的，BossGroup 负责接收客户端的连接，WorkGroup 专门负责网络读写操作。
2. NioEventLoop 表示一个不断循环执行处理任务的线程，每个 NioEventLoop 都有一个 selector，用于监听绑定在其上的 socket 网络通道。
3. NioEventLoop 内部采用串行化设计，从消息的读取、解码、处理、编码、发送，始终由 IO 线程 NioEventLoop 负责。

- NioEventLoopGroup 下包含多个 NioEventLoop（单线程的线程池）
- 每个 NioEventLoop 的 Selector 上可以注册监听多个 NioChannel
- 每个 NioChannel 只会绑定在唯一的 NioEventLoop 上
- 每个 NioChannel 都绑定有一个自己的 ChannelPipeline



















