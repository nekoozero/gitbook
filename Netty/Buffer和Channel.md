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

四个属性：

| 属性     | 描述                                                         |
| -------- | ------------------------------------------------------------ |
| Capacity | 容量，可以容纳的最大数据量；在缓冲区创建是被设定并且不能改变 |
| Limit    | 缓冲区当前重点，不能对缓冲区超过极限的位置进行读写操作，且极限是可以修改的 |
| Position | 位置，下一个要被读或写的元素的索引，每次读写缓冲区数据时都会改变该值，为下次读写做准备 |
| Mark     | 标记                                                         |

常用方法：

![Buffer_Method.png](http://www.qxnekoo.cn:8888/images/2020/08/03/Buffer_Method.png)

ByteBuffer 常用方法：

![ByteBuffer_Method.png](http://www.qxnekoo.cn:8888/images/2020/08/03/ByteBuffer_Method.png)

## Channel



