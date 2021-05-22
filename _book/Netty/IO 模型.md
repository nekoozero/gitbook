# BIO

> 时间：2020/8/2

java 支持三种网络编程模型 IO 模式：BIO、NIO、AIO。

1. BIO：同步并阻塞（传统阻塞型），服务器实现模式为一个连接一个线程，即客户端有连接请求时服务器端就需要启动一个线程进行处理，如果这个连接不做任何事就会造成不必要的线程开销。

   ![BIO.png](http://www.qxnekoo.cn:8888/images/2020/08/02/BIO.png)

2. NIO：同步非阻塞，服务器实现模式为一个线程处理多个请求（连接），即客户端发送的连接请求都会注册到多路复用器上，多路复用器轮询到连接有 IO 请求就进行处理。

   ![NIO.png](http://www.qxnekoo.cn:8888/images/2020/08/02/NIO.png)

3. AIO 方式使用于连接数目多且连接比较长（重操作）的架构，比如相册服务器，充分调用 OS 参与并发操作，JDK7 开始支持。

## BIO 基本介绍

1. Java BIO 就是传统的 java io 编程，类和接口在 java.io。
2. BIO（blocking IO）: 同步阻塞，服务器实现模式为一个连接一个线程，即客户端有连接请求时服务器端需要启动一个线程进行处理，如果这个连接不做任何事情会造成不必要的线程开销，可以通过线程池机制改善。
3. BIO 方式适用于连接数目比较小且固定的架构，这种方式对服务器资源要求比较高，并发局限于应用中，JDK1.4 以前的唯一选择，程序简单易理解。

## 流程梳理

1. 服务器端启动一个 ServerSocket。
2. 客户端启动 Socket 对服务器进行通信，默认情况下服务器端需要对每个客户，建立一个线程与之通讯。
3. 客户端发出请求后。先咨询服务器是否有线程响应，如果没有则会等待，或者被拒绝。
4. 如果有响应，客户端线程会等待请求结束后，再继续执行。

## 示例代码

简单的客户端：

```java
/**
 * 简单的客户端
 *
 * @author qianxin
 * @Date 2020/7/31
 */
public class ClientSocketTest {
    public static void main(String[] args) throws IOException {
        Socket socket = new Socket("127.0.0.1", 9999);
        BufferedWriter bufferedWriter = new BufferedWriter(new OutputStreamWriter(socket.getOutputStream()));
        String str = "你好，这是我的第一个Socket";
        bufferedWriter.write(str);
        //没有下列代码  报错 Connection reset
        //因为客户端没有给他一个标识，告诉是否消息发送完成，所以服务端还在一直等待接受客户端的数据，
        //结果客户端此时已经关闭了，就是在服务端报错：java.net.SocketException: Connection reset

        //刷新输入流
        bufferedWriter.flush();
        //关闭socket的输出流  socket.close() 也可以结束socket
        //socket.close() 将socket关闭连接
        //socket.shutdownOutput()是将输出流关闭
        socket.shutdownOutput();

        //这样的方式 客户端发送之后 关闭输入流程序就结束了
    }
}

```

简单的服务端：

```java
/**
 * 简单的服务端
 *
 * @author qianxin
 * @Date 2020/7/31
 */
public class ServerSocketTest {
    public static void main(String[] args) throws IOException {
        ServerSocket serverSocket = new ServerSocket(9999);
        //等待客户端的连接  会阻塞当前线程
        System.out.println("等待客户端");
        Socket accept = serverSocket.accept();
        System.out.println("有一个客户端连接！");
        //获取输入流
        BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(accept.getInputStream()));
        //read的时候也会阻塞当前线程 这个很重要
        String str = bufferedReader.readLine();
        System.out.println(str);
        //服务端在接收到信息后输出出来之后服务端也关闭了

    }
}
```

上面的代码在客户端输入一次就断开连接了，所以一般使用循环：

循环客户端：

```java
/**
 * 利用while循环来 一直保证输入和输出 （客户端）
 *
 * @author qianxin
 * @date 2020/7/31
 */
public class ClientSocketWhileTest {
    public static void main(String[] args) throws IOException {
        Socket socket = new Socket("127.0.0.1", 9999);
        //通过socket获取字符流
        BufferedWriter bufferedWriter = new BufferedWriter(new OutputStreamWriter(socket.getOutputStream()));
        //通过标准输入流获取字符流
        BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(System.in, "UTF-8"));
        while (true) {
            String s = bufferedReader.readLine();
            bufferedWriter.write(s);
            bufferedWriter.write("\n");
            bufferedWriter.flush();
            //不给socket.close()和socket.shutdownOutput() 也可以获取到数据  因为客户端那边给了标识
            //在循环发送数据时候，每发送一行，添加一个换行标识“\n”标识，在告诉服务端我数据已经发送完成了。
            //而服务端在读取客户数据时，通过while ((str = bufferedReader.readLine())!=null)去判断是否读到了流的结尾，
            //否则服务端将会一直阻塞在哪里，等待客户端的输入。

            if (s.equals("end")) {
                //如果这边关闭了连接 ，那么此时服务端的(str = bufferedReader.readLine())=null了
                //服务端的代码走出循环 服务端也关闭了
                socket.close();
                break;
            }
        }
    }
}
```

循环服务端：

```java
/**
 * 利用while循环来 一直保证输入和输出 （服务端）
 *
 * @author qianxin
 * @date 2020/7/31
 */
public class ServerSocketWhileTest {
    public static void main(String[] args) throws IOException {
        ServerSocket serverSocket = new ServerSocket(9999);
        System.out.println("等待连接");
        Socket accept = serverSocket.accept();
        System.out.println("已有一个连接，等待输入");
        BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(accept.getInputStream(), "UTF-8"));
        String str;
        while ((str = bufferedReader.readLine()) != null) {
            System.out.println(str);
        }
    }
}
```

由于 socket 通信是阻塞式的，假设我现在有 A 和 B 俩个客户端同时连接到服务端的上，当客户端 A 发送信息给服务端后，那么服务端将一直阻塞在 A 的客户端上，不同的通过 while 循环从 A 客户端读取信息，此时如果 B 给服务端发送信息时，将进入阻塞队列，直到 A 客户端发送完毕，并且退出后，B 才可以和服务端进行通信。简单地说，**我们现在实现的功能，虽然可以让客户端不间断的和服务端进行通信，与其说是一对一的功能，因为只有当客户端 A 关闭后，客户端 B 才可以真正和服务端进行通信，这显然不是我们想要的。** 

所以大多数情况下都采用多线程技术来实现。

多线程：

```java
/**
 * Bio 阻塞演示
 *
 * @author qianxin
 * @date 2020/7/31
 */
public class BioServer {
    static ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(1, 3, 0, TimeUnit.SECONDS, new LinkedBlockingQueue<>(1), Executors.defaultThreadFactory());
    //思路
    //1创建一个线程池
    //2如果有客户端连接，就创建一个线程，与之通讯
    public static void main(String[] args) throws IOException {
        //创建ServerSocket
        ServerSocket serverSocket = new ServerSocket(6666);
        System.out.println("主线程：服务器启动了");
        while (true) {
            System.out.println("主线程：等待连接……");
            final Socket accept = serverSocket.accept();
            System.out.println("主线程：连接到一个线程");
            threadPoolExecutor.execute(() -> {
                try {
                    handler(accept);
                } catch (IOException e) {
                    e.printStackTrace();
                }
            });
        }
    }
    static void handler(Socket socket) throws IOException {
        byte[] bytes = new byte[1024];
        try (InputStream inputStream = socket.getInputStream()) {
            while (true) {
                int read = inputStream.read(bytes);
                if (read != -1) {
                    System.out.println(new String(bytes, 0, read));
                    System.out.println("活跃的线程数:" + threadPoolExecutor.getActiveCount());
                } else {
                    break;
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            socket.close();
        }

    }
}
```

可使用 `telnet` 访问，一个 `telnet` 窗口可视为一个客户端。

> telnet 127.0.0.1 6666
>
> Ctrl+] 开始输入 enter 发送信息

从结果上来看，如果开了很多 `telnet` 窗口，即有很多客户端连接服务端，从服务端也可以找到有很多线程来连接这些 `telnet` 窗口（客户端），但是如果这个客户端一直不输入的话，可以看到，线程还是一直在的，也就是说服务端的线程都在阻塞着等待客户端的输入，如果客户端一直不输入的话，将会有很大的线程浪费。

## 问题分析

1. 每个请求都需要创建独立的线程，与对应的客户端进行数据 Read，业务处理，数据 Write。
2. 当并发数较大时，需要创建大量线程来处理连接，系统资源占用较大。
3. 连接建立后，如果当前服务端线程暂时没有数据可读，则线程就阻塞在 Read 操作上，造成线程资源浪费















