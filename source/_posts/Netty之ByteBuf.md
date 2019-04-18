---
title: Netty之ByteBuf
date: 2019-04-18 11:25:27
tags: [Netty,Netty in action]
categories: 
- java
- netty
---

## Netty之ByteBuf

本文内容主要参考**Netty In Action**,偏笔记向.

网络编程中,字节缓冲区是一个比较基本的组件.Java NIO提供了`ByteBuffer`,但是使用过的都知道`ByteBuffer`对于读写数据操作还是有些麻烦的,切换读写状态需要`flip()`.**Netty**框架对字节缓冲区进行了封装,名称是`ByteBuf`,相较于`ByteBuffer`更灵活.

<!-- more -->

### 1.ByteBuf特点概览

- 用户可以自定义缓冲区类型对其扩展
- 通过内置的符合缓冲区类型实现了透明的零拷贝
- 容量可以按需增长(类似`StringBuilder`)
- 切换读写模式不用调用`flip()`方法
- 读写使用各自的索引
- 支持方法的链式调用
- 支持引用计数
- 支持池化

### 2.ByteBuf类介绍

#### 2.1工作模式

`ByteBuf`维护了两个指针,一个用于读取(`readerIndex`),一个用于写入(`writerIndex`).

使用**ByteBuf的API**中的`read*`方法读取数据时,`readerIndex`会根据读取字节数向后移动,但是`get*`方法不会移动`readerIndex`;使用`write*`数据时,`writerIndex`会根据字节数移动,但是`set*`方法不会移动`writerIndex`.(`read*`表示`read`开头的方法,其余意义相同)

读取数据时,如果`readerIndex`超过了`writerIndex`会触发`IndexOutOfBoundsException`.

可以指定`ByteBuf`容量最大值,`capacity(int)`或`ensureWritable(int)`,当超出容量时会抛出异常.

#### 2.2使用模式

##### 2.2.1堆缓冲区

将`ByteBuf`存入**JVM**的堆空间.能够在没有池化的情况下提供快速的分配和释放.

除此之外,ByteBuf的堆缓冲区还提供了一个后备数组(backing array).后备数组和ByteBuf中的数据是对应的,如果修改了`backing array`中的数据,`ByteBuf`中的数据是同步的.

```java
public static void main(String[] args) {
        ByteBuf heapBuf = Unpooled.buffer(1024);
        if(heapBuf.hasArray()){
            heapBuf.writeBytes("Hello,heapBuf".getBytes());
            System.out.println("数组第一个字节在缓冲区中的偏移量:"+heapBuf.arrayOffset());
            System.out.println("缓冲区中的readerIndex:"+heapBuf.readerIndex());
            System.out.println("writerIndex:"+heapBuf.writerIndex());
            System.out.println("缓冲区中的可读字节数:"+heapBuf.readableBytes());//等于writerIndex-readerIndex
            byte[] array = heapBuf.array();
            for(int i = 0;i < heapBuf.readableBytes();i++){
                System.out.print((char) array[i]);
                if(i==5){
                    array[i] = (int)'.';
                }
            }
            //不会修改readerIndex位置
            System.out.println("\n读取数据后的readerIndex:"+heapBuf.readerIndex());
            //读取缓冲区的数据,查看是否将逗号改成了句号
            while (heapBuf.isReadable()){
                System.out.print((char) heapBuf.readByte());
            }
        }
```

输出:

```verilog
数组第一个字节在缓冲区中的偏移量:0
缓冲区中的readerIndex:0
writerIndex:13
缓冲区中的可读字节数:13
Hello,heapBuf
读取数据后的readerIndex:0
Hello.heapBuf
```

> 如果`hasArray()`返回`false`,尝试访问backing array会报错

##### 2.2.2直接缓冲区

直接缓冲区存储于**JVM堆外**的内存空间.这样做有一个好处,当你想把JVM中的数据写给socket,需要将数据复制到直接缓冲区(JVM堆外内存)再交给socket.如果使用直接缓冲区,将减少复制这一过程.

但是直接缓冲区也是有不足的,与JVM堆的缓冲区相比,他们的分配和释放是比较昂贵的.而且还有一个缺点,面对遗留代码的时候,可能不确定ByteBuf使用的是直接缓冲区还是堆缓冲区,你可能需要进行一次额外的复制.如代码示例.

与自带后备数组的堆缓冲区来讲,这要多做一些工作.所以,如果确定容器中的数据会被作为数组来访问,你可能更愿意使用堆内存.

```java
		//实际上你不知道从哪获得的引用,这可能是一个直接缓冲区的ByteBuf
		//忽略Unpooled.buffer方法,当做不知道从哪获得的directBuf
		ByteBuf directBuf = Unpooled.buffer(1024); 
		//如果想要从数组中访问数据,需要将直接缓冲区中的数据手动复制到数组中
        if (!directBuf.hasArray()) {
            int length = directBuf.readableBytes();
            byte[] array = new byte[length];
            directBuf.getBytes(directBuf.readerIndex(), array);
            handleArray(array, 0, length);
        }
```

##### 2.2.3符合缓冲区(CompositeByteBuf)

聚合缓冲区是个非常好用的东西,是多个ByteBuf的聚合视图,可以添加或删除ByteBuf实例.

> CompositeByteBuf中的ByteBuf实例可能同事包含直接内存分配和非直接内存分配.如果其中只有一个实例,那么调用CompositeByteBuf中的`hasArray()`方法将返回该组件上的`hasArray()`方法的值,否则返回`false`

多个ByteBuf组成一个完整的消息是很常见的,比如`header`和`body`组成的HTTP协议传输的消息.消息中的`body`有时候可能能重用,我们不想每次都创建重复的`body`,我们可以通过CompositeByteBuf来复用`body`.

对比一下JDK中的`ByteBuffer`实现复合缓冲区和Netty中的`CompositeByteBuf`.

```java
//JDK版本实现复合缓冲区
public static void byteBufferComposite(ByteBuffer header, ByteBuffer body) {
        //使用一个数组来保存消息的各个部分
        ByteBuffer[] message =  new ByteBuffer[]{ header, body };

        // 创建一个新的ByteBuffer来复制合并header和body
        ByteBuffer message2 =
                ByteBuffer.allocate(header.remaining() + body.remaining());
        message2.put(header);
        message2.put(body);
        message2.flip();
    }

//Netty中的CompositeByteBuf
 public static void byteBufComposite() {
        CompositeByteBuf messageBuf = Unpooled.compositeBuffer();
        ByteBuf headerBuf = Unpooled.buffer(1024); // 可能是直接缓存也可能是堆缓存中的
        ByteBuf bodyBuf = Unpooled.buffer(1024);   // 可能是直接缓存也可能是堆缓存中的
        messageBuf.addComponents(headerBuf, bodyBuf);
        //...
        messageBuf.removeComponent(0); // remove the header
        for (ByteBuf buf : messageBuf) {
            System.out.println(buf.toString());
        }
    }
```



`CompositeByteBuf`不支持访问其后备数组,所以访问`CompositeByteBuf`中的数据类似于访问直接缓冲区

```java
CompositeByteBuf compBuf = Unpooled.compositeBuffer();
int length = compBuf.readableBytes();
byte[] array = new byte[length];
//将CompositeByteBuf中的数据复制到数组中
compBuf.getBytes(compBuf.readerIndex(), array);
//处理一下数组中的数据
handleArray(array, 0, array.length);
```

Netty使用`CompositeByteBuf`来优化socket的IO操作,避免了JDK缓冲区实现所导致的性能和内存使用率的缺陷.内存使用率的缺陷是指对可复用对象大量的复制,Netty对其在内部做了优化,虽然没有暴露出来,但是应该知道CompositeByteBuf的优势和JDK自带工具的弊端.

> JDK的NIO包中提供了Scatter/Gather I/O技术,字面意思是打散和聚合,可以理解为把单个ByteBuffer切分成多个或者把多个ByteBuffer合并成一个.

### 3.字节级操作

ByteBuf的索引从0开始,最后一个索引是`capacity()-1`.

遍历演示

```java
ByteBuf buffer = Unpooled.buffer(1024); 
for (int i = 0; i < buffer.capacity(); i++) {
    byte b = buffer.getByte(i);//这种方法不会移动readerIndex指针
    System.out.println((char) b);
}
```

#### 3.1readerIndex和writerIndex

JDK中的`ByteBuffer`只有一个索引,需要通过`flip()`来切换读写操作,Netty中的`ByteBuf`既有读索引,也有写索引,通过两个索引把ByteBuf划分了三部分.

![](https://note.youdao.com/yws/api/personal/file/F82D62BF4B9E4AD8BCF6F9F9B63D29E1?method=download&shareKey=952006455eadac7758f7ae04b1d59c7f)

可以调用`discardReadBytes() `方法可丢弃可丢弃字节并回收空间.

调用`discardReadBytes() `方法之后

![](https://note.youdao.com/yws/api/personal/file/E6A068B1C5CB4A949911943A989D9892?method=download&shareKey=952006455eadac7758f7ae04b1d59c7f)

使用`read*`或`skip*`方法都会增加`readerIndex`.

移动`readerIndex`读取可读数据的方式

```java
ByteBuf buffer = ...;
while (buffer.isReadable()) {
    System.out.println(buffer.readByte());
}
```

`write*`方法写入`ByteBuf`时会增加writerIndex,如果超过容量会抛出`IndexOutOfBoundException `.

`writeableBytes()`可以返回可写字节数.

```java
ByteBuf buffer = ...;
while (buffer.writableBytes() >= 4) {
	buffer.writeInt(random.nextInt());
}
```

#### 3.2索引管理

> JDK 的` InputStream` 定义了 `mark(int readlimit)`和`reset()`方法，这些方法分别被用来将流中的当前位置标记为指定的值，以及将流重置到该位置。
> 同样，可以通过调用 `markReaderIndex()`、` markWriterIndex()`、 `resetWriterIndex()`和` resetReaderIndex()`来标记和重置 `ByteBuf `的` readerIndex `和 `writerIndex`。这些和`InputStream `上的调用类似，只是没有` readlimit` 参数来指定标记什么时候失效。 

如果将索引设置到一个无效位置会抛出`IndexOutOfBoundsException`.

可以通过`clear()`归零索引,归零索引不会清除数据.

#### 3.3查找

ByteBuf中很多方法可以确定**值**的索引,如`indexOf()`.

复杂查找可以通过那些需要一个`ByteBufProcessor`作为参数的方法完成.这个接口应该可以使用`lambda`表达式(但是我现在使用的Netty4.1.12已经废弃了该接口,应该使用`ByteProcessor`).

```java
ByteBuf buffer = ...;
int index = buffer.forEachByte(ByteProcessor.FIND_CR);
```

#### 3.4派生缓冲区

派生缓冲区就是,基于原缓冲区一顿操作生成新缓冲区.比如复制,切分等等.

`duplicate()`；`slice()`； `slice(int, int) `;`Unpooled.unmodifiableBuffer(…) `;`order(ByteOrder)`； `readSlice(int) `.

> 每个这些方法都将返回一个新的 ByteBuf 实例，它具有自己的读索引、写索引和标记
> 索引。 其内部存储和 JDK 的 ByteBuffer 一样也是共享的。这使得派生缓冲区的创建成本
> 是很低廉的，但是这也意味着，如果你修改了它的内容，也同时修改了其对应的源实例，所
> 以要小心 

```java
//复制
public static void byteBufCopy() {
        Charset utf8 = Charset.forName("UTF-8");
        ByteBuf buf = Unpooled.copiedBuffer("Netty in Action rocks!", utf8);
        ByteBuf copy = buf.copy(0, 15);
        System.out.println(copy.toString(utf8));
        buf.setByte(0, (byte)'J');
        assert buf.getByte(0) != copy.getByte(0);
    }
//切片
 public static void byteBufSlice() {
        Charset utf8 = Charset.forName("UTF-8");
        ByteBuf buf = Unpooled.copiedBuffer("Netty in Action rocks!", utf8);
        ByteBuf sliced = buf.slice(0, 15);
        System.out.println(sliced.toString(utf8));
        buf.setByte(0, (byte)'J');
        assert buf.getByte(0) == sliced.getByte(0);
    }
```

还有一些读写操作的API,留在文末展示吧.

### 4.ByteBufHolder接口

> 我们经常发现， 除了实际的数据负载之外， 我们还需要存储各种属性值。 HTTP 响应便是一个很好的例子， 除了表示为字节的内容，还包括状态码、 cookie 等。
> 为了处理这种常见的用例， Netty 提供了 ByteBufHolder。 ByteBufHolder 也为 Netty 的高级特性提供了支持，如缓冲区池化，其中可以从池中借用 ByteBuf， 并且在需要时自动释放。ByteBufHolder 只有几种用于访问底层数据和引用计数的方法。 

![](https://note.youdao.com/yws/api/personal/file/B261AE020F7B48D386C6E6F20E33AD75?method=download&shareKey=952006455eadac7758f7ae04b1d59c7f)

### 5.ByteBuf的分配

我们可以通过`ByteBufAllocator`来分配一个`ByteBuf`实例.`ByteBufAllocator`接口实现了ByteBuf的池化.

可以通过 `Channel`（每个都可以有一个不同的 `ByteBufAllocator `实例）或者绑定到`ChannelHandler` 的 `ChannelHandlerContext `获取一个到` ByteBufAllocator `的引用。 

```java
//从Channel获取一个ByteBufAllocator的引用
Channel channel = ...;
ByteBufAllocator allocator = channel.alloc();
....
//从ChannelHandlerContext获取ByteBufAllocator 的引用
ChannelHandlerContext ctx = ...;
ByteBufAllocator allocator2 = ctx.alloc();
```

> Netty提供了两种ByteBufAllocator的实现： PooledByteBufAllocator和UnpooledByteBufAllocator。前者池化了ByteBuf的实例以提高性能并最大限度地减少内存碎片。 后者的实现不 池化ByteBuf实例， 并且在每次它被调用时都会返回一个新的实例。 

默认使用的是`PooledByteBufAllocator `,可以通过`ChannelConfig`修改.

**Unpooled缓冲区**

可能有时候拿不到`ByteBufAllocator`引用的话,可以使用Unpooled工具类来创建未持化`ByteBuf`实例.

**ByteBufUtil类**

> ByteBufUtil 提供了用于操作 ByteBuf 的静态的辅助方法。因为这个 API 是通用的， 并且和池化无关，所以这些方法已然在分配类的外部实现。
> 这些静态方法中最有价值的可能就是 hexdump()方法， 它以十六进制的表示形式打印ByteBuf 的内容。这在各种情况下都很有用，例如， 出于调试的目的记录 ByteBuf 的内容。十六进制的表示通常会提供一个比字节值的直接表示形式更加有用的日志条目，此外，十六进制的版本还可以很容易地转换回实际的字节表示。
> 另一个有用的方法是 boolean equals(ByteBuf, ByteBuf)， 它被用来判断两个 ByteBuf实例的相等性。如果你实现自己的 ByteBuf 子类，你可能会发现 ByteBufUtil 的其他有用方法。 

#### 6.引用计数

> 引用计数是一种通过在某个对象所持有的资源不再被其他对象引用时释放该对象所持有的资源来优化内存使用和性能的技术。 它们都实现了 interface ReferenceCounted。 引用计数背后的想法并不是特别的复杂；它主要涉及跟踪到某个特定对象的活动引用的数量。一个 ReferenceCounted 实现的实例将通常以活动的引用计数为 1 作为开始。只要引用计数大于 0， 就能保证对象不会被释放。当活动引用的数量减少到 0 时，该实例就会被释放。注意，虽然释放的确切语义可能是特定于实现的，但是至少已经释放的对象应该不可再用了。 

```java
//从Channel获取ByteBufAllocator
Channel channel = ...;
ByteBufAllocator allocator = channel.alloc();
....
//从ByteBufAllocator分配一个ByteBuf
ByteBuf buffer = allocator.directBuffer();
assert buffer.refCnt() == 1;//引用计数是否为1

```



### 7.API

#### ByteBuf

![](https://note.youdao.com/yws/api/personal/file/CF647FAEE0C642CABCFC1329454F9A3C?method=download&shareKey=952006455eadac7758f7ae04b1d59c7f)

![](https://note.youdao.com/yws/api/personal/file/FD2B630F7AFF4B8C85E86467EFE53765?method=download&shareKey=952006455eadac7758f7ae04b1d59c7f)

![](https://note.youdao.com/yws/api/personal/file/64A587980C8748B5AFA674BCF223D953?method=download&shareKey=952006455eadac7758f7ae04b1d59c7f)

![](https://note.youdao.com/yws/api/personal/file/44DF026ABA3545C5AC63D096043BF880?method=download&shareKey=952006455eadac7758f7ae04b1d59c7f)

![](https://note.youdao.com/yws/api/personal/file/7762FDB1C0854B519C65B6DD9C1B53E7?method=download&shareKey=952006455eadac7758f7ae04b1d59c7f)

#### ByteBufAllocator

![](https://note.youdao.com/yws/api/personal/file/758F1B1A5A6F4D238012C7F79A0B7BC6?method=download&shareKey=952006455eadac7758f7ae04b1d59c7f)



#### Unpooled

![](https://note.youdao.com/yws/api/personal/file/F05DD429F5E144C5B3175CF96E572539?method=download&shareKey=952006455eadac7758f7ae04b1d59c7f)

