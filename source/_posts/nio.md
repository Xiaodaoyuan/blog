---
title: Java NIO详解
date: 2018-10-24 18:26:20
categories: Java
tags: nio
---

#### 1.NIO概念  
　　NIO与传统IO相比最大的特点就是非阻塞。IO是阻塞的，当一个线程调用read或write方法时，线程必须等待文件读取完或者写完才能去执行其他任务；而NIO是非阻塞的，它允许线程请求从通道中读取数据，如果当前没有可用数据，线程可以继续执行其他操作，而不是一直阻塞。  
<!--more-->
---

#### 2.NIO主要组件
NIO包含三个主要的类：
- Buffer
- Channel
- Selector  

NIO有许多类组成，但是buffer，channel和selector构成了核心的API。所以接下来会主要介绍这三个类。通常，NIO中的所有IO都是以channel开始。Channel其实和IO中的stream差不多。通过channel数据能被写入buffer中，同样数据能从buffer中写入channel。如下图：  
![image](/images/channels-buffers.png)

Buffer和Channel的主要子类如下所示：
- ByteBuffer ->MappedByteBuffer(实现内存映射文件)
- CharBuffer
- ShortBuffer
- IntBuffer
- FloatBuffer
- DoubleBuffer
- LongBuffer
---
- FileChannel
- DatagramChannel
- SocketChannel
- ServerSocketChannel

Selector允许一个线程操作多个Channel。如果你的应用程序打开了很多连接（Channel），但是每个连接上的通信量很低，那么这时候Selector就很方便。比如聊天服务器。  
下图说明了selector和channel的关系：  
![image](/images/overview-selectors.png)

---
#### 3.Buffer
- **3.1 Capacity、Position、Limit**  
buffer主要有三个属性：capacity、position和limit。  
   * capacity：buffer的容量，我们在分配buffer时会指定一个容量。当buffer达到容量之后需要清除数据才能继续向buffer中写入数据。
   * position：buffer中数据写入的位置。初始为0，往buffer写入或读取一字节数据，position就会加1，当清除buffer 中数据或者由写模式切换为读模式时，position会重新置0，postion最大为capacity-1。
   * limit：在写模式中，limit就等于buffer的capacity。在读模式中，limit是指可以从buffer中读取的数据量的限制。
   ![image](/images/buffers-modes.png)
- **3.2 Allocating a Buffer**  
想要得到一个buffer首先必须分配它，每个Buffer类中都提供了一个静态方法allocate()来分配buffer。
```
ByteBuffer buf = ByteBuffer.allocate(2048);
CharBuffer buf = CharBuffer.allocate(1024);
```

- **3.3 Writing Data to a Buffer**  
往buffer中写入数据有两种方式：
  * 一是调用buffer的put方法写入数据，如buf.put(127)； 
  * 二是将数据通过channel写入buffer，如int num = channel.read(buf);  
 flip()方法是将buffer从写模式切换到读模式，这个会重新设置position和limit的值。
- **3.4 Reading Data from a Buffer**  
从buffer中读取数据也有两种方式：
  * 一是调用buffer的get方式读取数据，如buf.get()；
  * 二是读取buffer数据到channel中，如int bytesWritten = inChannel.write(buf);
- **3.5 rewind()、clear()、compact()**  
  * rewind()可以将position值重新置为0，这样就可以重新读取buffer中的数据。
  * clear()方法将设置poistion为0，设置limit为capacity，clear虽然不会清除buffer中的数据，但是后续写入的数据会覆盖之前的数据，相当于清空了buffer数据。
  * compact()和clear不同，它会将之前的数据移向左边，将position设置为最后一个未读数据，从这个基础上再开始写入。
- **3.6 mark()、reset()**  
调用mark()方法可以标记当前buffer中的position，执行一些操作之后调用reset()可以可以将position回到刚才标记的位置。
```
buffer.mark();

//call buffer.get() a couple of times, e.g. during parsing.

buffer.reset();  //set position back to mark.  
```
---

#### 4.Channel
所有的NIO操作都是始于channel，数据的读取和写入都会经过channel。
- **4.1  FileChannel**(文件通道，用于文件的读和写)   
不能设置为非阻塞模式，只能是阻塞模式
  - **Open a FileChannel**  
  使用一个FileChannel之前必须先打开它，但是不可能直接打开FileChannel，必须通过InputStream，OutputStream或者RandomAccessFile来获取一个FileChannel。  
 `RandomAccessFile aFile     = new RandomAccessFile("data/nio-data.txt", "rw");`  
`FileChannel      inChannel = aFile.getChannel();`
  - **Read Data from a FileChannel**  
  `ByteBuffer buf = ByteBuffer.allocate(48);`   
`int bytesRead = inChannel.read(buf);`  
上面bytesRead如果返回-1，说明读取完了

  - **Write Data to a FileChannel**  
  `String newData = "New String to write to file..." + System.currentTimeMillis();`  
`ByteBuffer buf = ByteBuffer.allocate(48);`  
`buf.clear();`  
`buf.put(newData.getBytes());`  
`buf.flip();`  
`while(buf.hasRemaining()) { `   
`    channel.write(buf); `   
`}`
  - **Close a FileChannel**  
  `channel.close()`
  - **FileChannel Position**  
  当你想在一个特定位置读取或写入FileChannel，你可以通过position()方法获取FileChannel的当前位置，然后你可以通过position(long newPosition)方法设置新的postiion。    
`long pos=channel.position();`  
`channel.position(pos + 123);`
  - **FileChannel Size**  
  FileChannel对象的size()方法返回通道连接的文件的文件大小。
  - **FileChannel Truncate**  
   调用FileChannel.truncate()方法来截断文件。当您截断一个文件时，您将它以给定的长度截断。  
   `channel.truncate(1024);`
  - **FileChannel Force**  
  force()方法将所有未写入的数据从通道刷新到磁盘。force()方法以一个布尔值作为参数，告诉文件元数据(权限等)是否也应该刷新。

- **4.2  SocketChannel**(用于通过TCP读写数据)  
SocketChannel是连接到TCP网络套接字的通道。它相当于Java NIO的Java网络套接字。  
打开一个SocketChannel的代码如下：  
`SocketChannel socketChannel = SocketChannel.open();`  
`socketChannel.connect(new InetSocketAddress("http://xxx.com", 80));`  
SocketChannel的读写和FileChannel没什么区别。

- **4.3  ServerSocketChannel**(TCP对应的服务端，用来某个端口进来的请求)  
ServerSocketChannel可以用来监听进来的tcp连接，就像java标准网络中的ServerSocket。
```
//实例化
ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
//监听端口
serverSocketChannel.socket().bind(new InetSocketAddress(9999));

while(true){
    //监听进来的tcp连接，创建SocketChannel
    SocketChannel socketChannel = serverSocketChannel.accept();
    //do something with socketChannel...
}

```
- **4.4  DatagramChannel**(用于通过UDP从网络上读写数据)   
DatagramChannel是一个能够发送和接收UDP包的通道，由于UDP是无连接的，所以在默认情况下，您不能像从其他通道那样读写DatagramChannel。你只可以发送和接收数据包。  
监听端口
```
DatagramChannel channel = DatagramChannel.open();
channel.socket().bind(new InetSocketAddress(9999));
```
接收数据
```
ByteBuffer buf = ByteBuffer.allocate(48);
buf.clear();
channel.receive(buf);
```
发送数据

```
String newData = "New String to write to file..."+ System.currentTimeMillis();
ByteBuffer buf = ByteBuffer.allocate(48);
buf.clear();
buf.put(newData.getBytes());
buf.flip();
int bytesSent = channel.send(buf, new InetSocketAddress("xxx.com", 80));
```

- **4.5 Channel Transfers**  
在NIO中，如果其中一个channel是FileChannel时，则可以将数据直接从一个通道传到另一个通道。  
FileChannel中的transferFrom()方法能够将数据从一个源通道传送到FileChannel中。
```
RandomAccessFile fromFile = new RandomAccessFile("fromFile.txt", "rw");
FileChannel      fromChannel = fromFile.getChannel();
RandomAccessFile toFile = new RandomAccessFile("toFile.txt", "rw");
FileChannel      toChannel = toFile.getChannel();
long position = 0;
long count    = fromChannel.size();
toChannel.transferFrom(fromChannel, position, count);
```
FileChannel中的transferTo()方法可以将数据从FileChannel中传送到其他通道。
```
RandomAccessFile fromFile = new RandomAccessFile("fromFile.txt", "rw");
FileChannel      fromChannel = fromFile.getChannel();
RandomAccessFile toFile = new RandomAccessFile("toFile.txt", "rw");
FileChannel      toChannel = toFile.getChannel();
long position = 0;
long count    = fromChannel.size();
fromChannel.transferTo(position, count, toChannel);
```

---
#### 5.Selector
Selector可以检查一个或多个channel，并确定哪些channel准备就绪。通过这种方式，一个线程可以管理多个channel，从而管理多个网络连接。  
![image](/images/overview-selectors.png)

##### 5.1 创建Selector和注册Channel到Selector

```
Selector selector = Selector.open();
//设置为非阻塞模式
channel.configureBlocking(false);
SelectionKey key = channel.register(selector, SelectionKey.OP_READ);
```
上面第二行用于设置非阻塞模式，由于Selector必须工作在非阻塞模式中，所以FileChannel不能使用Selector。  
register方法的第二个参数是一个int型参数（使用二进制标记位，叫interest set），用于表明channel需要监听哪些感兴趣的事件，共有以下四种事件。
- Read  
对应SelectionKey.OP_READ(值为1<<0)，表示channel可以读取数据。
- Write  
对应SelectionKey.OP_WRITE(值为1<<2)，表示channel可以写入数据。
- Connect  
对应SelectionKey.OP_CONNECT(值为1<<3)，表示channel建立连接。
- Accept  
对应SelectionKey.OP_ACCEPT(值为1<<4)，表示channel可以接收传入的连接。 


如果想要设置多个事件，则将对应的key进行与操作。如下：

```
int interestSet = SelectionKey.OP_READ | SelectionKey.OP_WRITE;  
```

##### 5.2 SelectionKey
上面注册channel时register方法会返回一个SelectionKey对象。这个SelectionKey对象包含以下属性：  
- **Interest Set**  
Interest Set中是channel感兴趣的事件集合，你可以通过下面的方法获取Interest Set。
```
int interestSet = selectionKey.interestOps();

boolean isInterestedInAccept  = interestSet & SelectionKey.OP_ACCEPT;
boolean isInterestedInConnect = interestSet & SelectionKey.OP_CONNECT;
boolean isInterestedInRead    = interestSet & SelectionKey.OP_READ;
boolean isInterestedInWrite   = interestSet & SelectionKey.OP_WRITE;
```
- **Ready Set**  
Ready Set 表示的是channel已经准备好的事件集合。通常在选择一个channel之后来访问这个集合。

```
int readySet = selectionKey.readyOps();
selectionKey.isAcceptable();
selectionKey.isConnectable();
selectionKey.isReadable();
selectionKey.isWritable();
```

- **Channel + Selector**  
获取channel和selector比较简单。
```
Channel  channel  = selectionKey.channel();
Selector selector = selectionKey.selector();    
```
- **Attaching Objects**  
可以将对象附加到SelectionKey对象上，这是识别channel或者将信息附加到channel上的方便方法。

```
selectionKey.attach(theObject);

Object attachedObj = selectionKey.attachment();
```
也可以在register的时候附加对象。

```
SelectionKey key = channel.register(selector, SelectionKey.OP_READ, theObject);
```
##### 5.3 Select Channel
通过Selector的select方法我们可以选择一个channel。select方法主要有三个：
- **select()**：这个方法会阻塞直到至少一个channel准备好。
- **select(long timeout)**：和select()功能类似，只是制定了阻塞时间。
- **selectNow()**：这个不会阻塞，不管有没有准备好的channel，都会立即返回。  

上面select方法的返回值是一个int，表示有多少个通道准备好。如果没有对第一个准备好的通道做任何操作，那么会有两个准备好的通道，但是在每个select()调用之间只有一个通道准备好。

- **wakeUp()**：这个方法是用来唤醒等待在 select() 和 select(timeout) 上的线程的。如果 wakeup() 先被调用，此时没有线程在 select 上阻塞，那么之后的一个 select() 或 select(timeout) 会立即返回，而不会阻塞，当然，它只会作用一次。

##### 5.5 Full Selector Example

```
Selector selector = Selector.open();

channel.configureBlocking(false);

SelectionKey key = channel.register(selector, SelectionKey.OP_READ);


while(true) {

  int readyChannels = selector.select();

  if(readyChannels == 0) continue;


  Set<SelectionKey> selectedKeys = selector.selectedKeys();

  Iterator<SelectionKey> keyIterator = selectedKeys.iterator();

  while(keyIterator.hasNext()) {

    SelectionKey key = keyIterator.next();

    if(key.isAcceptable()) {
        // a connection was accepted by a ServerSocketChannel.

    } else if (key.isConnectable()) {
        // a connection was established with a remote server.

    } else if (key.isReadable()) {
        // a channel is ready for reading

    } else if (key.isWritable()) {
        // a channel is ready for writing
    }

    keyIterator.remove();
  }
}
```

#### 6.NIO Pipe
NIO Pipe是两个线程之间一个单向的数据连接。Pipe有一个source channel和一个sink channel。将数据写入sink channel，然后可以从source channel读取这些数据。  
![image](/images/pipe-internals.png)
```
//open pipe
Pipe pipe = Pipe.open();

//write data to sink channel
Pipe.SinkChannel sinkChannel = pipe.sink();
String newData = "New String to write to file..." + System.currentTimeMillis();
ByteBuffer buf = ByteBuffer.allocate(48);
buf.clear();
buf.put(newData.getBytes());
buf.flip();
while(buf.hasRemaining()) {
    sinkChannel.write(buf);
}

//read data from source channel
Pipe.SourceChannel sourceChannel = pipe.source();
ByteBuffer buf = ByteBuffer.allocate(48);
int bytesRead = sourceChannel.read(buf);
```


