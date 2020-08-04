# Buffer和Channel

> 时间：2020/8/3

## Buffer

缓冲区本质上是一个可以读写数据的内存块，可以理解成是一个容器对象（含数组）。内置一些机制，跟踪和记录缓冲区的状态变化情况。

常用子类：

- **ByteBuffer**（用的较多）
- ShortBuffer
- CharBuffer
- IntBuffer
- LongBuffer
- DoubleBuffer
- FloatBuffer

***四个属性*：**

| 属性     | 描述                                                         |
| -------- | ------------------------------------------------------------ |
| Capacity | 容量，可以容纳的最大数据量；在缓冲区创建是被设定并且不能改变 |
| Limit    | 缓冲区当前重点，不能对缓冲区超过极限的位置进行读写操作，且极限是可以修改的 |
| Position | 位置，下一个要被读或写的元素的索引，每次读写缓冲区数据时都会改变该值，为下次读写做准备 |
| Mark     | 标记                                                         |

***常用方法***：

![Buffer_Method.png](http://www.qxnekoo.cn:8888/images/2020/08/03/Buffer_Method.png)

ByteBuffer 常用方法：

![ByteBuffer_Method.png](http://www.qxnekoo.cn:8888/images/2020/08/03/ByteBuffer_Method.png)

## Channel

1. NIO 的通道类似于流，但有些区别。
2. BIO 的 stream 是单向的，NIO 中的通道是双向的，可以读，也可以写。
3. Channel 在 NIO 中是一个接口。
4. 常用的 Channel 类：FileChannel（文件读写）、DatagramChannel（UDP的数据读写）、ServerSocketChannel 和 SocketChannel（TCP数据读写）

***FileChannel 类主要方法***

- public int read(ByteBuffer dst)，从通道读取数据并放到缓冲区中
- publuc int write(ByteBuffer src)，把缓冲区的数据写到通道中
- public long transferFrom(ReadableByteChannel src,long position,long count)，从目标通道中复制数据到当前通道
- public long transferTo(long position,long count WritableByteChannel target)，把数据从当前通道复制给目标通道

***代码示例***

```java
/**
 * @author qianxin
 * @date 2020/8/3
 */
public class NIOFileChannel03 {

    public static void main(String[] args) {
        try (final FileInputStream fileInputStream = new FileInputStream("d:\\file01.txt");
             final FileChannel inputStreamChannel = fileInputStream.getChannel();
             final FileOutputStream fileOutputStream = new FileOutputStream("1.txt");
             final FileChannel outputStreamChannel = fileOutputStream.getChannel()) {
            final ByteBuffer allocate = ByteBuffer.allocate(3);
            while (true) {
                //向Buffer中写
                int i = inputStreamChannel.read(allocate);
                System.out.println(i);
                if (i == -1) {
                    break;
                }
                //要切到读模式（从buffer中读）
                allocate.flip();
                outputStreamChannel.write(allocate);
                //当前使用完清空掉 很重要
                allocate.clear();
            }
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

## 注意点

可以将通道（Channel）理解成铁路，而缓冲区（Buffer）理解为火车上的货物。

1. ByteBuffer 支持类型化的 put 和 get ,如果 put 和 get 相应的数据类型不同，可能有 BufferUnderflowException 异常。

2. 可以将一个普通的 Buffer 转成只读 Buffer。

3. NIO 还提供了 MappedByteBuffer，可以让文件直接在内存（堆外的内存）中进行修改。

   > MappedByteBuffer mappedByteBuffer = channel.map(FileChannel.MapMode.READ_WRITE,0,5);
   >
   > 参数1：FileChannel.MapMode.READ_WRITE 使用的读写模式
   >
   > 参数2： 0 ：可以直接修改的起始位置
   >
   > 参数3： 5 ：是映射到内存的大小（不是索引位置）
   >
   > 实际类型 DirectByteBuffer

4. NIO 支持通过多个 Buffer（Buffer 数组）完成读写操作，即 Scattering 和 Gathering。













