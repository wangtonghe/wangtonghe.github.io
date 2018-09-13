---
layout: post
title:  "java I/O体系总结(三) java NIO"
date:   2018-09-13 21:12:00 +0800
categories: java nio
header-img: img/posts/java/java-io.jpg
tags:
 - java
 - io
 - nio
 - 总结
---

## 概览

||IO|NIO|
---|---|---
|特点|面向流|面向缓冲|
|是否阻塞|阻塞IO|非阻塞IO
||无|选择器


java 新IO主要部分：Buffer(缓冲区)、Channel(通道)、Selectors(选择器)

Java NIO的非阻塞模式，如使一个线程从某通道发送请求读取数据，但是它仅能得到目前可用的数据，如果目前没有数据可用时，就什么都不会获取。而不是保持线程阻塞，所以直至数据变的可以读取之前，该线程可以继续做其他的事情。一个单独的线程可以管理多个输入和输出通道（channel）。

NIO中，所以操作都以缓冲区进行的。

Channel 表示通道，与流有一些类似，用于在字节缓冲区和位于通道另一侧的实体（文件或套接字）之间有效的传输数据。需注意的是，通道只接受ByteBuffer作为参数。

缓冲区是通道内部用来发送和接收数据的端点。

### Channel与流的区别：

1. 通道(Channel)既可以读取数据也可以写入数据，而流是单向的（如InputStream是输入流，OutputStream是输出流）
2. 通道(Channel)不能直接访问数据，只能通过缓冲（Buffer）去访问。
3. 通道只在字节缓冲区操作（因为操作系统都是以字节的形式实现底层I/O接口的）
4. 流，就像水流一样，单向，流过去了就不会回来；而通道如其名，双向，可来可去，可读可写。



通道

|channel类别|说明|
|---|---|
|FileChannel|文件通道| 
| DatagramChannel |UDP通道，用于通过UDP读取网络中的数据通道|
| SocketChannel |TCP通道，用于通过TCP读取网络数据|
| ServerSocketChannel |监听新进来的TCP连接，对每个链接都创建一个SocketChannel |


以上4种channel大致可分为文件通道和套接字通道。文件通道指的是FileChannel，套接字通道则有三个，分别是SocketChannel、ServerSocketChannel和DatagramChannel。

### 获取Channel的方法

1. 通过getChannel()方法获取。
   FileInputStream/FileOutputStream、Socket、DatagramSocket等类都有此方法。
2. 静态open方法；如FileChannel.open()
3. Files.newByteChannel


## FileChannel 

先说说文件通道吧，

看其继承图


![这里写图片描述](https://img-blog.csdn.net/20180913212203202?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3d0aGZlbmc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

FileChannel可通过FileInputStream或FileOutputStream或RandomAccessFile对象上调用getChannel()方法来获取。

得到的FileChannel拥有和file对象相同的访问权限。

FileChannel是线程安全的，多个进程可在同一实例上并发调用。

需注意的是，文件通道是阻塞的。FileChannel不能切换到非阻塞模式。而套接字通道都可以。

### 示例

```*java*

   /**
     * FileChannel 读取
     * @throws Exception
     */
    @Test
    public void testChannel() throws Exception {

        File file = new File("/Users/wangtonghe/local/tmp/hello.txt");

        FileInputStream fileInputStream = new FileInputStream(file);
        // 获取channel
        FileChannel fileChannel = fileInputStream.getChannel(); 
        // 分配Buffer
        ByteBuffer byteBuffer = ByteBuffer.allocate(40);
        // 将文件内容读取出来
        fileChannel.read(byteBuffer);
        // 将channel设为可读状态
        byteBuffer.flip();
        while (byteBuffer.hasRemaining()) {
            System.out.print((char) byteBuffer.get());
        }
    }

```

```java

   @Test
    public void testChannel2() throws Exception {
        File file = new File("/Users/wangtonghe/local/tmp/hello.txt");
        FileOutputStream fileOutputStream = new FileOutputStream(file);
        FileChannel fileChannel = fileOutputStream.getChannel();
        ByteBuffer byteBuffer = ByteBuffer.allocate(20);
        String str = "asdfghjk\n";
        byteBuffer.put(str.getBytes());
        byteBuffer.flip();
        fileChannel.write(byteBuffer);
        byteBuffer.clear();
        fileOutputStream.close();
        fileChannel.close();
    }

```

### Buffer

Buffer有以下4个主要属性，用以控制读写状态。

|属性	|作用|
|---|---|
|capacity|	容量,指缓冲区能够容纳的数据元素的最大数量，这一容量在缓冲区创建时被设定，并且永远不能被改变
|limit	|上界，缓冲区中现存元素的边界。即不可读或不可写的位置|
|position	|指示位置，缓冲区读取或写入的下一个位置。位置会自动由相应的get()和put()函数更新
|mark|	标记，指一个备忘位置，调用mark()来设定mark=position，调用reset()来设定postion=mark，标记未设定前是未定

#### flip方法

很重要的方法，将Buffer从可写状态变为可读状态。


#### 直接缓冲区

直接缓冲区，避免了缓冲区在I/O上的复制。直接缓冲区使用的内存是直接调用操作系统分配的，绕过了JVM的堆栈结构。

可通过调用ByteBuffer.allocateDirect()分配。



## Socket通道

有关Socket的Channel主要有3个：ServerSocketChannel、SocketChannel、DatagramChannel。ServerSocketChannel表示服务器端的Socket通道，而SocketChannel表示客户端的Socket通道。DatagramChannel表示数据报（UDP）的通道。

Socket通道均支持非阻塞式连接。在介绍非阻塞前，首先看看阻塞式的Socket是怎样的


### 阻塞式Socket

这里讨论的阻塞非阻塞针对服务器端。阻塞式处理一般采用多线程的方式。使用ServerSocket(服务器端Socket)和Socket(客户端Socket)。步骤如下：

1. 调用ServerSocket的accept()方法，等待客户端连接，如没有连接此方法会一直阻塞，若有，返回与客户端通信的socket
2. 根据1.中返回的socket获取IntputStream及OutputStream,以便与客户端进行通信。
3. 通信完毕，关闭连接。
4. 有新连接到来，重复2

要了解非阻塞式IO,就要先了解阻塞IO到底哪里阻塞住了？看代码

```java
       public static void main(String[] args) throws Exception {

        // 创建服务器端
        ServerSocket server = new ServerSocket(8000);
        while (true) {
            // 在这阻塞，直到下一个请求到来
            try (Socket socket = server.accept()) {
                // 没有分线程处理，即请求串行执行，当前请求必须处理完毕才能处理下一个，
                // 若某个请求处理很慢，将直接影响后续所有请求
                Reader reader = new InputStreamReader(socket.getInputStream());
                int c;
                while ((c = reader.read()) != -1) {
                    System.out.print((char) c);
                }
                reader.close();
            } catch (IOException ex) {
                ex.printStackTrace();
            }
        }
    }

```
是accept()方法阻塞才被认为是阻塞IO吗？可是，accept()方法是在等待客户端连接，没有人连接也就不用处理什么业务啊。显然关键不在这里。我们在学习IO流的时候就说过，流是同步的，当程序在读、写一段数据时，要等待该数据流可读或可写，也就是读、写过程有IO等待及操作的时间。也即

> IO读写时间=IO等待阻塞时间+操作时间

而IO等待是不需要CPU的，且相对耗时较长，而操作时间则很快，属于CPU时间级别。非阻塞的原理就在怎样避免IO阻塞时间，让CPU把时间都花在操作时间上。

> 需要说明的是，首先上面示例仅为演示，没考虑效率问题；另外，一般阻塞式IO在处理读写操作时会使用一个固定的线程池来处理，以免读写操作太过耗时而影响所有后续连接，且这是一个很经典的做法。在如今线程已有大幅优化的情况下，阻塞IO+多线程仍是一个选择。


### 非阻塞式Socket

NIO是java1.4推出的重要功能，主要目的是用于构建高并发非阻塞式的服务器应用。其涉及的概念挺多，如通道（Channel）,缓冲区（Buffer）以及选择器（Selector）。用法稍显复杂。先来简单介绍下这些概念。

Channel主要用到ServerSocketChannel及SocketChannel,分别表示服务器和客户端。
Selector为选择器，用于注册通道。选择器不太好理解，是这样：假设有好多客户端都来连这个服务器，则每个客户端都有一条Channel(通道)与服务器相连，这么多Channel,同一时刻总有的通道没数据（IO阻塞），有的准备好了（可读或可写）。选择器的作用就是把那些能读或能写的通道选出来，供程序读写。这样就能节省掉IO阻塞的时间了。

选择器除了能辨别通道是否可读或可写，还能判断是否有连接到来（accept()方法），这样把accept()阻塞的时间都省了。

不过，选择器的select()方法是阻塞的，用于表示至少一个事件准备好了（可读或可写或连接到来，具体取决于注册的事件）。


```java

public static void main(String[] args) throws Exception {

        // 打开一个ServerSocketChannel
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
        // 设置监听地址
        serverSocketChannel.bind(new InetSocketAddress(8000));
        //设置非阻塞模式
        serverSocketChannel.configureBlocking(false);
        //选择器
        Selector selector = Selector.open();
        // 服务器注册接收事件
        serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
        ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
        while (true) {
            // 阻塞,直到连接到来
            selector.select();
            // 就绪通道的集合
            Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();
            while (iterator.hasNext()) {
                SelectionKey curKey = iterator.next();
                iterator.remove();
                if (curKey.isAcceptable()) {
                    ServerSocketChannel ssc = (ServerSocketChannel) curKey.channel();
                    // 与某个客户端已连接上
                    SocketChannel clientChannel = ssc.accept();
                    clientChannel.configureBlocking(false);
                    // 将该Channel注册到注册器上
                    clientChannel.register(selector, SelectionKey.OP_READ | SelectionKey.OP_WRITE);
                } else if (curKey.isReadable()) {
                    // 某一通道可读
                    SocketChannel sc = (SocketChannel) curKey.channel();
                    byteBuffer.clear();
                    while (sc.read(byteBuffer) > 0) {
                        byteBuffer.flip();
                        String msg = Charset.forName("UTF-8").decode(byteBuffer).toString();
                        System.out.println("received from: " + msg);
                    }
                } else if (curKey.isWritable()) {
                    SocketChannel sc = (SocketChannel) curKey.channel();
                    String body = "<html><head>百度</head><body>hello baidu!</body></html>";
                    ByteBuffer buffer = ByteBuffer.wrap(body.getBytes());
                    sc.write(buffer);
                }
            }
        }


    }


```
NIO的非阻塞式代码比较固定，大致都是这个写法。

1. 创建服务端的SocketChannel（ServerSocketChannel）并初始化
2. 创建选择器（Selector）
3. 将ServerSocketChannel及其接收新连接事件注册到选择器上。
4. 调用选择器的select()方法阻塞，直到新连接到来（这时只注册了这一个事件）
5. 每个新连接到来后，获取该连接对应的SocketChannel（客户端channel），将其可读或可写（或随需求）事件注册到选择器上。
6. 此时当连接的可读或可写事件准备好后，触发对应逻辑。然后回到4循环。


另外补充下

>  ServerSocketChannel只有一个功能，就是接收客户端的连接请求。无法读取、写入。只能支持的操作就是接受一个新的入站请求。

