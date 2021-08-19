# Netty

## IO读写原理

用户程序进行IO的读写，基本会用到read&write两大系统调用。read系统调用是从内核缓冲区复制到进程缓冲区内

而write系统调用则是将数据从进程缓冲区复制到内核缓冲区。这两个系统调用都不负责进行内核缓冲区和磁盘的数据交互，底层的读写交换是通过操作系统kernel内核完成的

## 内核缓冲区与进程缓冲区

缓冲区的目的是减少IO的系统调用，因为系统调用需要保存当前的进程信息和状态，而结束调用之后需要恢复之前的信息，为了减少这种频繁调用时机，设置了缓冲区

每当用户想要读数据的时候，其实是在读进程缓冲区而不是调用IO，有时候也会是调用IO，所以用户程序的IO读写程序，大多数情况下，并没有进行实际的IO操作，而是在读写自己的进程缓冲区

![image-20210121160844501](D:\学习笔记\image-20210121160844501.png)

上图是典型的JAVA服务端处理网络请求的典型过程：

(1) 客户端请求 

Linux通过网卡，读取到客户端的请求数据，将数据写入到内核缓冲区内

(2) 获取请求数据

服务器从内核缓冲区读取数据到JAVA进程缓冲区

(3) 服务端业务处理

JAVA服务端处理客户端的数据

(4) 服务端返回数据

服务端将构建好的相应，从JAVA进程缓冲区写入内核缓冲区

(5) 发送给客户端

Linux内核通过网络IO,将内核的缓冲区中的数据写入网卡，网卡通过底层的通讯协议将数据发送给目标客户端

## Linux五种主要的IO模型

服务器编程经常需要构造高性能的IO模型，常见的IO模型主要有四种

(1) 同步阻塞IO(blocking IO)

首先，解释一下这里的阻塞与非阻塞

阻塞IO：是指用户程序线程发起IO系统调用的时候开始阻塞等待内核准备数据并且将数据写入到用户缓冲区，这个过程用户程序是挂起的，阻塞的，用户空间程序需要等待IO操作完成。java中，默认创建的socket都是阻塞的

其次，解释一下同步与异步

同步IO：是指用户程序线程主动向内核发起系统调用，用户空间线程是主动方，而内核空间则是被动接受的乙方，异步IO则反过来 内核空间是主动方，用户程序是被动方

(2) 同步非阻塞IO(Non-blocking IO)

同步非阻塞IO是指用户空间线程主动向内核空间发起系统调用，不需要等待内核IO操作完成后，内核会立刻返回一个状态值，用户接受到状态值立刻返回，然后执行用户程序，处于非阻塞状态

非阻塞IO要求socket被设置为NON BLOCK

(3) IO多路复用

即经典的Reactor设计模式，有时候也称为异步阻塞IO，JAVA中的Selector和Linux中的epoll都是这种模型

(4) 异步IO (Asynchronous IO)

异步IO，指的是用户空间与内核空间的调用反过来，用户空间线程变成被动接口，而内核空间是主动调用者，这种情况下一般需要用户空间线程向内核注册驱动或者设置回调函数

### 同步阻塞IO



![image-20210121170346271](D:\学习笔记\image-20210121170346271.png)

举个例子，发起一个blocking socke的读系统调用，流程大概是这样的：

(1) 当用户线程调用了read系统调用，就开始了IO的第一个阶段，数据准备(数据准备一般是指内核从网卡读取数据到内核缓冲区)，很多时候数据还没有到达(比如说还没有接受到一个完整的socket数据包)，这个时候kernel就要等待足够的数据到来

(2) 当数据准备好以后，会将数据从内核缓冲区复制到用户进程缓冲区，然后返回结果

(3) 从开始IO读的read系统调用开始，用户线程就进入阻塞状态，直到kernel返回结果，用户才解除block状态，开始重新运行起来开始处理数据

Bio的优点：程序简单，在阻塞等待数据的时候，用户线程挂起，用户线程基本不会占用CPU资源

BIO的缺点：一般情况下，会为每个连接配套一个独立的线程，换句话说，一条线程维护一个连接成功的IO读写。在并发量很小的时候，不会有什么问题，但是高并发下需要大量的线程来维护大量的网络连接，内存，线程切换开销会非常大，基本上 BIO在高并发下不可能被用到

### 同步非阻塞IO

在Linux系统下，需要设置socket使其变为non-blocking，NIO模型中一旦开始IO系统调用，会出现两种情况

(1) 在内核缓冲区数据还没有准备好的情况下，用户线程发起IO系统调用，内核会立刻返回一个调用失败的信息

(2) 在内核缓冲区数据准备好的情况下，用户发起IO系统调用，内核会将内核缓冲区的数据拷贝到用户进程缓冲区，并且用户程序线程是阻塞的，拷贝结束后内系统调用返回成功，应用进程开始运行处理数据

![image-20210121171538187](D:\学习笔记\image-20210121171538187.png)

NIO特点：应用程序线程需要不断的发起IO系统调用，轮询数据是否已经准备好，如果没有准备好，继续轮询，直到系统调用完成为止

优点：每次发起的 IO 系统调用，在内核的等待数据过程中可以立即返回。用户线程不会阻塞，实时性较好。

缺点：需要不断的发起IO系统调用检查数据是否准备成功，这种不断的轮询需要不断的访问内核，占用大量的CPU资源，系统利用率较低

总之，NIO模型在高并发场景下，也是不可用的。一般 Web 服务器不使用这种 IO 模型。一般很少直接使用这种模型，而是在其他IO模型中使用非阻塞IO这一特性。java的实际开发中，也不会涉及这种IO模型

JAVA的NIO(new NIO)不是IO模型中的NIO，而是IO多路复用模型

### IO多路复用模型

如何避免NIO中的轮询等待的问题呢？这就是IO多路复用模型

IO多路复用模型，就是一种新的系统调用，一个进程监视多个文件描述符，某个描述符就绪，内核通知程序进行相应的IO系统调用

目前支持IO多路复用的系统模型，有select,epoll等等

首先不会发起IO系统调用，而是先发起select/epoll系统调用，IO多路复用基本原理就是select/epoll系统调用，单个线程不断轮询selct/epoll负责的成千上百条socket连接，一旦某个连接有数据到达了，就返回该连接

![image-20210121174137642](D:\学习笔记\image-20210121174137642.png)

首先进行IO系统调用，但是这里有个前提，需要将目标网络连接提前注册到select/epoll的可可查询列表中。

(1) 当进行查询的时候，kernel会查询所有select的可查询列表，当任何一个socket中数据准备好了，select就会返回，用户线程调用了select，那么该线程就会被阻塞，直到返回

(2) 用户线程获取到了目标连接后，发起read系统调用，用户线程阻塞，内核开始复制数据，然后kernel返回结果

(3) 用户解除block状态，开始处理数据

IO多路复用模型，是建立在操作系统内核提供的多路分离系统调用的select/epoll基础之上的，和NIO模型相似，多路复用IO需要轮询，负责select/epoll查询调用的线程，需要不断的进行轮询

并且NIO和IO多路复用模型是有一定关系的，对于每一个可以查询的socket，对设置为了non blocking模型

多路复用优点：可以同时处理成千上万个连接，而不用创建成千上万个线程，大大减少了系统的开销

缺点：本质上其实还是同步阻塞的IO，只不过都是在读写事件发生后，自己只负责读写，而查询则是由select/epoll完成

### 异步IO

![image-20210121175115902](D:\学习笔记\image-20210121175115902.png)

(1) 当用户线程发起read系统调用，立刻可以去做其他的事情，用户线程不阻塞

（2）内核（kernel）就开始了IO的第一个阶段：准备数据。当kernel一直等到数据准备好了，它就会将数据从kernel内核缓冲区，拷贝到用户缓冲区（用户内存）。

（3）kernel会给用户线程发送一个信号（signal），或者回调用户线程注册的回调接口，告诉用户线程read操作完成了。

（4）用户线程读取用户缓冲区的数据，完成后续的业务操作。

异步IO模型的特点：

在内核kernel的等待数据和复制数据的两个阶段，用户线程都不是block(阻塞)的。用户线程需要接受kernel的IO操作完成的事件，或者说注册IO操作完成的回调函数，到操作系统的内核。

异步IO模型缺点：

需要完成事件的注册与传递，这里边需要底层操作系统提供大量的支持，去做大量的工作。

### 信号驱动IO

我们也可以用信号，让内核在描述符就绪(也就是数据)就绪时发送一个信号通知我们，通过sigaction系统调用安装一个信号处理函数，该系统调用会立刻返回，主要来自信号处理函数的通知，那么就可以表示数据已准备好被处理，或者是数据已准备好被读取，

![image-20210121182950396](D:\学习笔记\image-20210121182950396.png)

该过程数据拷贝期间用户程序也是阻塞的

和异步IO的主要区别是异步IO拷贝数据不阻塞，并且是由内核通知我们IO操作何时完成，而信号驱动IO是由内核通知我们何时启动一个IO操作

## 文件描述符

文件描述符是计算机科学中的一个术语，用来表示指向文件的引用抽象化概念。

文件描述符是一个非负整数，实际上它是一个索引值，指内核会为每一个进程所维护一个该进程打开文件的记录表，当程序打开一个文件的时候，内核向进程返回一个文件描述符

## 线程模型

### 传统阻塞I/O服务模型

一个Client请求对应一个线程进行读取消息，业务处理，回复消息等等

开销大，如果多个请求要开启多个线程。

![image-20210125165519884](D:\学习笔记\image-20210125165519884.png)

可以对传统的线程I/O模型进行改进

1）**基于I/O复用模型**：设置一个相当于MVC的请求转发器，用来接受所有来自客服端的请求，然后通过相应的事件调用不同的Handler，多个连接公用一个阻塞对象，应用程序只需要在一个阻塞对象上等待

2) **基于线程池复用线程资源**：不用每次请求来的时候都创建线程，结束的时候销毁线程

![image-20210125165729634](D:\学习笔记\image-20210125165729634.png)

所以就有了下面的Reactor模型

### Reactor模型

![image-20210125165947992](D:\学习笔记\image-20210125165947992.png)

Reactor：Reactor在一个单独的线程中运行，负责监听和分发事件

Handlers：处理程序执行I/O事件要完成的实际操作

根据Reactor的数量和处理资源的不同，有三种典型的实现

#### 单Reactor 单线程

一个Reactor线程

优点：没有线程竞争

缺点：多核CPU下无法发挥性能，CPU资源利用率低


1）Reactor 对象通过 Select 监控客户端请求事件，收到事件后通过 Dispatch 进行分发；

2）如果是建立连接请求事件，则由 Acceptor 通过 Accept 处理连接请求，然后创建一个 Handler 对象处理连接完成后的后续业务处理；

3）如果不是建立连接事件，则 Reactor 会分发调用连接对应的 Handler 来响应；

4）Handler 会完成 Read→业务处理→Send 的完整业务流程。

![image-20210125170320092](D:\学习笔记\image-20210125170320092.png)

#### 单Reactor 多线程

Reactor线程负责监听和分发事件，如果是建立连接则去accept,如果是业务，则变为handler进行处理，但是只负责读取数据和发送数据，真正的业务处理丢到Workers线程池中

![image-20210125170549723](D:\学习笔记\image-20210125170549723.png)

方案说明：

1）Reactor 对象通过Select监控客服端请求事件，收到事件后通过dispatch进行转发

2）如果是建立连接事件，则由Acceptor通过accept处理建立请求，然后创建一个handler对象处理连接完成后的各种事件

3）如果不是建立连接，Reactor会分发调用连接对应的Handler来相应

4）Handler只负责响应事件，不做具体业务处理，通过read读取数据后，会分发给后面的Worker线程池进行业务处理

5）Worker 线程池会分配独立的线程完成真正的业务处理，如何将响应结果发给 Handler 进行处理；

6）Handler 收到响应结果后通过 Send 将响应结果返回给 Client。

优点：可以充分利用CPU多核的处理能力

缺点：多线程数据共享和访问比较负责，Reactor承担所有事件的监听和响应，在单线程中运行，高并发下容易成为瓶颈

#### 主从Reactor 多线程

![image-20210125171240230](D:\学习笔记\image-20210125171240230.png)

针对单Reactor多线程模型中，Reactor在单线程中运行，高并发下容易出现瓶颈，所以可以让Reactor在多线程中运行

Reactor主线程轮询会将客户端来的请求进行分发，如果发现是建立连接请求，则在自己的线程中处理，然后创建对应的Handlers交给Reactor子线程来处理Handler业务，并且返回数据也不需要从子线程到主线程，直接通过Reactor子线程响应给客服端

然后子线程轮询查询，如果有则去执行read读取数据，真正的业务还是交给Worker线程池来处理

- 1）Reactor 主线程 MainReactor 对象通过 Select 监控建立连接事件，收到事件后通过 Acceptor 接收，处理建立连接事件；
- 2）Acceptor 处理建立连接事件后，MainReactor 将连接分配 Reactor 子线程给 SubReactor 进行处理；
- 3）SubReactor 将连接加入连接队列进行监听，并创建一个 Handler 用于处理各种连接事件；
- 4）当有新的事件发生时，SubReactor 会调用连接对应的 Handler 进行响应；
- 5）Handler 通过 Read 读取数据后，会分发给后面的 Worker 线程池进行业务处理；
- 6）Worker 线程池会分配独立的线程完成真正的业务处理，如何将响应结果发给 Handler 进行处理；
- 7）Handler 收到响应结果后通过 Send 将响应结果返回给 Client。

优点：职责明确，交互简单，父线程只需要接受新连接，子线程完成后续的业务处理

### Proactor模型

Reactor模型是非阻塞同步网络模型，而Proactor是非阻塞异步网络模型，但是编程复杂度高，需要操作系统支持，所有Linux还是采用的Reactor模型

![image-20210125172701032](D:\学习笔记\image-20210125172701032.png)

### Netty线程模型

采用了Reactor主从线程模型

![image-20210125173150194](D:\学习笔记\image-20210125173150194.png)

**特别说明的是，虽然Netty使用了Reactor线程模型，但是创建的两个NioEventLoopGroup都是线程池，bossGroup只是在Bind某个端口后，获得其中一个线程作为MainReactor，专门处理accept事件。而workerGroup会被各个SubReactor和Worker线程充分使用**

## Netty框架的架构设计

![image-20210125174719738](D:\学习笔记\image-20210125174719738.png)

**Netty 功能特性如下：**

1）传输服务：支持 BIO 和 NIO；

2）容器集成：支持 OSGI、JBossMC、Spring、Guice 容器；

3）协议支持：HTTP、Protobuf、二进制、文本、WebSocket 等一系列常见协议都支持。还支持通过实行编码解码逻辑来实现自定义协议；

4）Core核心：可扩展的事件模型，通用通信API，支持零拷贝的ByteBuf缓冲对象

### 模块组件

**【bootStrap、 ServerBootStrap】**

bootStrap的意思是引导，一个Netty应用通过由一个BootStrap开始，主要作用是配置整个Netty程序，串联各个组件，Netty中的bootStrap 是客服端引导类，ServerBootStrap是服务端引导类

**【Future、ChannelFuture】：**

正如前面介绍，在 Netty 中所有的 IO 操作都是异步的，不能立刻得知消息是否被正确处理。

但是可以过一会等它执行完成或者直接注册一个监听，具体的实现就是通过 Future 和 ChannelFutures，他们可以注册一个监听，当操作执行成功或失败时监听会自动触发注册的监听事件。

**【Channel】**：

Netty网络通信的组件，能够用于执行网络I/O操作，Channel为用户提供：

1）当前网络连接的通道状态(例如是否打开?是否已连接？)

2）网络连接的配置参数

3）交互端的IP地址

4）提供异步的网络I/O操作，例如(建立连接，读写，绑定端口)

常用的Channel如下：

NiosocketChannel：异步的客户端Tcp Socket连接

NioServerSocketChannel：异步的服务端Tcp Socket连接

NioDatagramChannel ：异步的UDP连接

NioSctpChannel：异步的客户端Sctp连接

NioSctpServerChannel，异步的 Sctp 服务器端连接

***【Selector】：***

Netty基于Selector对象实现IO多路复用，通过Selector一个线程可以监听多个连接的Channel事件

当向一个Selector注册Channel后，Selector内部的机制就可以自动不断的查询这些注册的Channel是否有已就绪的事件(例如可读，可写)等

**【NioEventLoop】：**

NioEventLoop中维护了一个线程和任务队列，支持异步提交执行任务，线程启动时会调用NioEventLoop的run方法，执行IO任务和非IO任务

IO任务：selectionKey中ready的事件，如accept,connect,read,write等

非IO任务：添加到taskQueue中的任务，如bind()，register

**【NioEventLoopGroup】：**

NioEventLoopGroup相当于一个线程池，管理了一组线程，里面的每一个线程(NioEventLoop)负责多个Channel通道事件，一个通道对应一个线程

**【ChannelHandler】**：

ChannelHandler是一个接口，处理I/O事件或者拦截I/O事件，并转发到其ChannelPipline(业务处理链)中的下一个程序

ChannelHandler 本身并没有提供很多方法，因为这个接口有许多的方法需要实现，方便使用期间，可以继承它的子类：

> ChannelInboundHandler 用于处理入站 I/O 事件。
>
> ChannelOutboundHandler 用于处理出站 I/O 操作。

或者使用以下适配器类：

> ChannelInboundHandlerAdapter 用于处理入站 I/O 事件。
>
> ChannelOutboundHandlerAdapter 用于处理出站 I/O 操作。
>
> ChannelDuplexHandler 用于处理入站和出站事件。

**【ChannelHandlerContext】**：

保存 Channel 相关的所有上下文信息，同时关联一个 ChannelHandler 对象。

**【ChannelPipline】**：

保存 ChannelHandler 的 List，用于处理或拦截 Channel 的入站事件和出站操作。

ChannelPipeline 实现了一种高级形式的拦截过滤器模式，使用户可以完全控制事件的处理方式，以及 Channel 中各个的 ChannelHandler 如何相互交互。

下图引用 Netty 的 Javadoc 4.1 中 ChannelPipeline 的说明，描述了 ChannelPipeline 中 ChannelHandler 通常如何处理 I/O 事件。

I/O 事件由 ChannelInboundHandler 或 ChannelOutboundHandler 处理，并通过调用 ChannelHandlerContext 中定义的事件传播方法。

例如：ChannelHandlerContext.fireChannelRead（Object）和 ChannelOutboundInvoker.write（Object）转发到其最近的处理程序。

在Netty中每个Channel都有仅有一个ChannelPipline与之对应

![image-20210125200108483](D:\学习笔记\image-20210125200108483.png)

一个Channel包含了一个ChannelPipeline,而ChannelPipline里面又维护了一个由ChannelHandlerContext组成的双向链表，而每个ChannelHandlerContext又关联这一个ChannelHandler

## Netty框架工作原理

```java
publicstaticvoidmain(String[] args) {

       // 创建mainReactor

       NioEventLoopGroup boosGroup = newNioEventLoopGroup();

       // 创建工作线程组

       NioEventLoopGroup workerGroup = newNioEventLoopGroup();



       finalServerBootstrap serverBootstrap = newServerBootstrap();

       serverBootstrap

                // 组装NioEventLoopGroup

               .group(boosGroup, workerGroup)

                // 设置channel类型为NIO类型

               .channel(NioServerSocketChannel.class)

               // 设置连接配置参数
               // TCP队列长度
               .option(ChannelOption.SO_BACKLOG, 1024)

               .childOption(ChannelOption.SO_KEEPALIVE, true)

               .childOption(ChannelOption.TCP_NODELAY, true)

               // 配置入站、出站事件handler

               .childHandler(newChannelInitializer<NioSocketChannel>() {

                   @Override

                   protectedvoidinitChannel(NioSocketChannel ch) {

                       // 配置入站、出站事件channel

                       ch.pipeline().addLast(...);

                       ch.pipeline().addLast(...);

                   }

   });



       // 绑定端口

       intport = 8080;

       serverBootstrap.bind(port).addListener(future -> {

           if(future.isSuccess()) {

               System.out.println(newDate() + ": 端口["+ port + "]绑定成功!");

           } else{

               System.err.println("端口["+ port + "]绑定失败!");

           }

       });

}
基本过程描述如下：
   1）初始化创建两个NioEventLoopGroup,BossGroup负责分发事件和处理accept连接，WorkerGroup负责具体的业务IO操作
  2）基于ServerBootStrap引导类，配置Group，通道，参数，Handler等
  3）绑定端口，开始工作  
```

![image-20210125201402507](D:\学习笔记\image-20210125201402507.png)

Server端包含1个Boss NioEventLoopGroup和1个Worker NioEventLoopGroup

NioEventLoopGroup里面包含多个事件循环NioEventLoop，而每个NioEventLoop里面有一个selector,和一个事件循环线程

**每个Boss NioEventLoop循环执行的任务包含三步**：

1）通过Selector轮询Accept事件

2）发生Accept事件后创建对应的NioSocketChannel注册到某个WorkerGroup里面某个NioEventLoop的Selector上，使得Worker可以开始监听读写事件

3）处理任务队列中的任务，runAllTasks,其中包括调用eventloop.execute或者schedule执行的任务

**每个Worker NioEventLoop循环执行的任务包含三步**

1）通过Selector循环read,write事件

2）处理I/O事件，在NioSocketChannel可读可写事件发生时候进行处理

3）处理任务队列中的任务

## 异步和事件驱动

### JAVA网络编程

早期的网络编程开发人员，需要去花费大量的时间学习复杂的C语言套接字库

![](D:\学习笔记\image-20210316200950782.png)

![image-20210316201002613](D:\学习笔记\image-20210316201002613.png)

缺点就是都是为每一个客户端的连接创建一个新的线程。开销大，巨大上下文切换。

### JAVA NIO

![image-20210316201120287](D:\学习笔记\image-20210316201120287.png)

new Input/OutPut

### 选择器

![image-20210316201142220](D:\学习笔记\image-20210316201142220.png)

使用JAVA nio的selector。

### Netty简介

![image-20210316201212638](D:\学习笔记\image-20210316201212638.png)

以上是特点。

####  异步和事件驱动

- 非阻塞网络调用可以使得我们不必等待一个操作的完成。完全异步的IO正是基于这个特性构建的。
- 选择器使得我们能够通过较少的线程便可监视许多连接上的事件

### Netty核心组件

#### Channel

可以视为传入(入站)或者传出(出战)数据的通道。类似于现实中的管子。因此它可以被打开或者连接或者断开连接

![image-20210316201605465](D:\学习笔记\image-20210316201605465.png)

#### 回调

一个回调其实就是一个方法，有些函数需要你先传入给它一个函数，也就是方法的引用，然后调用传入函数。这个过程称为方法的回调。

Netty在内部使用使用了回调来处理事件。当一个回调被触发的时候，相关的事件的ChannelHandle接口的实现处理，比如：当一个新的连接建立的时候，ChannelHandle的channelActive()回调方法将会被调用。

![image-20210316202010695](D:\学习笔记\image-20210316202010695.png)

#### Future

该对象可以看作是一个异步操作结果的占位符；它将在未来某个时刻被完成。

JUC包里面的方法比较繁琐，并且还阻塞。所以Netty实现了自己的ChannelFuture，可以向ChannelFuture注册ChannelFutureListener实例。

监听器的回调方法operationComplete()，会将在对应的操作完成后调用。

![image-20210316202239344](D:\学习笔记\image-20210316202239344.png)

![image-20210316202255527](D:\学习笔记\image-20210316202255527.png)

#### 事件和ChannelHandle

Netty使用不同的事件来通知我们状态的改变或者是操作的状态。使得我们能够基于发生的事件触发适当的动作。这些动作可能是：

- 记录日志
- 数据转换
- 流控制
- 应用程序逻辑

Netty事件基本是基于入站和出战数据流相关性进行分类的。

入站

- 连接已被激活或者连接失活
- 数据读取
- 用户事件
- 错误事件

出战

- 打开或者关闭远程结点的连接
- 将数据写入到套接字

![image-20210316202646885](D:\学习笔记\image-20210316202646885.png)

每个事件都可以分发给ChannelHandle类中某个用户实现的方法。

#### 放在一起

##### Futru丶回调和ChannelHandle

Netty的异步模型是建立在Future和回调的概念之上的。而将事件派发到ChannelHandle上

##### 选择器，事件和EventLoop

Netty将Selector从应用程序中脱离出来。并且每个Channel对应一个EventLoop，用以处理所有事件如 ：

- 注册感兴趣的事件
- 将事件派发给ChannelHandle
- 安排进一步的动作

## 第一款Netty应用程序

EchoServer服务端

```java
package com.netty;

import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelFutureListener;
import io.netty.channel.ChannelHandler;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;
import io.netty.handler.codec.http.HttpClientUpgradeHandler;
import io.netty.util.CharsetUtil;

/**
 * @program: TheFirstNettyApplication
 * @description:
 * @author: Mr.Feng
 * @create: 2021-03-16 20:36
 **/
@ChannelHandler.Sharable
public class EchoServerHandle extends ChannelInboundHandlerAdapter {


    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        ByteBuf in = (ByteBuf) msg;
        System.out.println(
                "Server received: " + in.toString(CharsetUtil.UTF_8));
        ctx.write(in); //此处不刷新
    }

    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
       ctx.writeAndFlush(Unpooled.EMPTY_BUFFER)
               .addListener(ChannelFutureListener.CLOSE);//增加一个通道关闭的监听，如果客户端关闭通道，那么就触发该监听。
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        cause.printStackTrace();
        ctx.close();
    }
}

```

如果不捕获异常呢？

每个Chaanel默认有一个与之相关的ChannelPipeline，其持有一个ChannelHandle实例链，并且该实例链处理完数据会传递给下一个实例链，如果不捕获，那么将会出现一直传递，直到末尾出现错误被记录。

```java
package com.netty;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.channel.socket.nio.NioSocketChannel;

import java.net.InetSocketAddress;
import java.util.concurrent.Executors;

/**
 * @program: TheFirstNettyApplication
 * @description:
 * @author: Mr.Feng
 * @create: 2021-03-16 21:06
 **/
public class EchoServer {

    private final int prot;

    public EchoServer(int port){
        this.prot = port;
    }

    public static void main(String[] args) throws InterruptedException {
        if (args.length != 1){
            System.err.println(
                    "Usage: " + EchoServer.class.getSimpleName() +
                            " <port>");
        }
        int port = Integer.parseInt(args[0]);
        new EchoServer(port).start();
    }

    public void start() throws InterruptedException {
        EventLoopGroup group = new  NioEventLoopGroup();
        ServerBootstrap bootstrap = new ServerBootstrap();
        try {
            bootstrap.group(group)
                    .channel(NioServerSocketChannel.class)
                    .localAddress(new InetSocketAddress(prot))
                    .childHandler(new ChannelInitializer<NioSocketChannel>() { //每当建立连接的时候都会生成一个子NioSocketChannel通道
                        @Override
                        protected void initChannel(NioSocketChannel nioSocketChannel) throws Exception {
                            nioSocketChannel.pipeline().addLast(new EchoServerHandle());
                        }
                    });
            ChannelFuture future = bootstrap.bind().sync(); //异步进行绑定参数设置，同步等待绑定完毕
            /**
             closeFutrue()直到发生通道关闭事件。因为EchoServerHandle注册了一个通道关闭事件，那么客户端关闭
             后会通知，然后关闭，也是异步的，使用sync关键字使其同步
             **/
            future.channel().closeFuture().sync();
        }catch (Exception e){
            e.printStackTrace();
        }finally {
            group.shutdownGracefully().sync();
        }




    }
}

```

![image-20210316212325716](D:\学习笔记\image-20210316212325716.png)

你创建了一个 ServerBootstrap 实例。因为你正在使用的是 NIO 传输，所以你指定了 NioEventLoopGroup 来接受和处理新的连接，并且将 Channel 的类型指定为 NioServerSocketChannel 。在此之后，你将本地地址设置为一个具有选定端口的 InetSocketAddress 。服务器将绑定到这个地址以监听新的连接请求。在 处，你使用了一个特殊的类——ChannelInitializer。这是关键。当一个新的连接被接受时，一个新的子 Channel 将会被创建，而 ChannelInitializer 将会把一个你的EchoServerHandler 的实例添加到该 Channel 的 ChannelPipeline 中。正如我们之前所解释的，这个 ChannelHandler 将会收到有关入站消息的通知。虽然 NIO 是可伸缩的，但是其适当的尤其是关于多线程处理的配置并不简单。

接下来你绑定了服务器 ，并等待绑定完成。（对 sync()方法的调用将导致当前 Thread阻塞，一直到绑定操作完成为止）。在 处，该应用程序将会阻塞等待直到服务器的 Channel关闭（因为你在 Channel 的 CloseFuture 上调用了 sync()方法）。然后，你将可以关闭EventLoopGroup，并释放所有的资源，包括所有被创建的线程 。

- EchoServerHandle实现了业务逻辑
- 引导过程中所需要的步骤如下
  - 创建一个ServerBootstrap的实例以引导和绑定服务器
  - 创建并且分配一个NioEventLoopGroup实例中进行事件的处理，如新连接以及读/写数据
  - 指定服务器的IP地址和端口
  - 向Channel的PipeLine中添加ChannelHandle业务逻辑
  - 进行服务器绑定

Client客户端

```java
package com.netty.client;

import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelHandler;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.SimpleChannelInboundHandler;
import io.netty.channel.socket.nio.NioSocketChannel;
import io.netty.util.CharsetUtil;

/**
 * @program: TheFirstNettyApplication
 * @description:
 * @author: Mr.Feng
 * @create: 2021-03-16 21:20
 **/
@ChannelHandler.Sharable
public class EchoCilentHandle extends SimpleChannelInboundHandler<ByteBuf> {

    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        ctx.writeAndFlush(Unpooled.copiedBuffer("Netty rocks",CharsetUtil.UTF_8));
    }


    @Override
    protected void channelRead0(ChannelHandlerContext channelHandlerContext, ByteBuf in) throws Exception {
        System.out.println("Client received"+ in.toString(CharsetUtil.UTF_8));
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        cause.printStackTrace();
        ctx.close();
    }
}

```

需要注意的是channelRead0()函数，每当调用该方法的时候，服务器发送的消息可能会被分块接受，也可以会被一次性接收，比如说服务端发送5字节的数据。可能channelRead0会一次性接受到5字节，也可能被调用二次，第一次拿的3字节的ByteBuf(字节容器)，第二次调用传的是2个字节的ButeBuf

**SimpleChannelInboundHandle和ChannelInboundHandle的区别**

为什么客户端用了**SimpleChannelInboundHandle**而服务端用了**ChannelInBoundHandle**，两者有什么区别呢？

**SimpleChannelInboundHandle**适用于不写出数据，而是直接读入输出并且输出，当该方法返回的时候，**SimpleChannelInboundHandle**会负责释放保存当前消息的ByteBuf的内存引用，相当于释放ByteBuf的占用的资源，调用ByteBuf.release()方法

而服务端使用**ChannelInBoundHandle**，因为服务端还要向客户端写入，所以不能使用**SimpleChannelInboundHandle**，因为当channelRead0()方法返回完成的时候可能还没有发送完数据，但是该方法返回后又要进行释放ByteBuf的资源。也就是清除ByteBuf的内存引用，可能会造成数据丢失。所以选择了在readComplete()处调用writeAndFlush方法时释放数据

```java
import io.netty.bootstrap.Bootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioSocketChannel;

import java.net.InetSocketAddress;

/**
 * @program: TheFirstNettyApplication
 * @description:
 * @author: Mr.Feng
 * @create: 2021-03-16 22:04
 **/
public class EchoClient {
    private final String host;
    private final int port;
    public EchoClient(String host,int port){
        this.host = host;
        this.port = port;
    }
    public void start() throws InterruptedException {
        EventLoopGroup group = new NioEventLoopGroup();
        Bootstrap bootstrap = new Bootstrap();
        try {
            bootstrap.group(group)
                    .remoteAddress(new InetSocketAddress(host,port))
                    .channel(NioSocketChannel.class)
                    .handler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel socketChannel) throws Exception {
                            socketChannel.pipeline().addLast(new EchoCilentHandle());
                        }
                    });
            ChannelFuture future = bootstrap.connect().sync();
            future.channel().closeFuture().sync(); //同步等待通道关闭
        }catch (Exception e){
            e.printStackTrace();
        }finally {
            group.shutdownGracefully().sync();
        }
    }

    public static void main(String[] args) throws InterruptedException {
        if (args.length != 2) {
            System.err.println(
                    "Usage: " + EchoClient.class.getSimpleName() +
                            " <host> <port>");
            return;
        }
        String host = args[0];
        int port = Integer.parseInt(args[1]);
        new EchoClient(host, port).start();
    }

}

```

![image-20210316224728475](D:\学习笔记\image-20210316224728475.png)

第一个Netty程序。

## Netty的组件与设计

### Channel丶EventLoop和ChannelFuture

- Channel—Socket；
- EventLoop—控制流、多线程处理、并发；
- ChannelFuture—异步通知。

#### Channel接口

基本的I/O操作(bind(),connect(),read()和write())依赖于底层网络传输所提供的原语，JAVA中其最基本的构造是Socket。Netty的Channel接口大大降低了直接使用Socket类的复杂性。

- EmbeddChannel
- LocalServerChannel
- NioDatagramChannel
- NioSctpChannel
- NioSocketChannel

#### EventLoop接口

EventLoop定义了接口的核心抽象，用于处理连接的生命周期中所发生的事情。

高层次上说明了Channl,EventLoop,Thread以及EventLoopGroup之间的关系。

![image-20210319205712660](D:\学习笔记\image-20210319205712660.png)

这些关系是

- 一个EventLoopGroup包含一个或者多个EventLoop
- 一个EventLoop在它的生命周期中只有一个Thread绑定。
- 所有由EventLoop处理的IO事件都将在它专有的Thread上被处理。
- 一个Channel在它的生命周期内只能注册于一个EventLoop
- 一个EventLoop可能会被分配给一个或者多个Channel

#### ChannelFutrue

因为Netty所有的I/O操作都是异步的，所以一个操作不会立刻返回，所以我们需要知道该操作时候会返回，成功了还是失败，所以需要ChannelFuture来获取结果。

### ChannelHandle和ChannelPipeline

####  ChannelHandle

从开发者的角度来看，Netty的主要组件是ChannelHandle。ChannelHandle里面的方法是由网络事件所触发的。

#### ChannelPipeline

ChannelPipeline提供了ChannelHandle链的容器，可以获取ChannelPipeline来将多个ChannelHandle链接起来。当Channel被创建的时候，会被自动分配到它所属的ChannelPipeline。

ChannelHandle安装到ChannelPipeline的过程如下：

- 一个ChannelInitializer的实现被注册到了ServerBootStrap

- 当 ChannelInitializer.initChannel()方法被调用时，ChannelInitializer

  将在 ChannelPipeline 中安装一组自定义的 ChannelHandler；

- ChannelInitializer将它自己从ChannelPipeLine移除。

![image-20210319210643298](D:\学习笔记\image-20210319210643298.png)

![image-20210319210728834](D:\学习笔记\image-20210319210728834.png)

通过使用作为参数传递到每个方法的ChannelHandleContext，调用该参数的API可以使得事件被传递给当前ChannelPipeline的下一个链上的Channelhandle

当ChannelHandle添加到ChannelPipeline上的时候，它将会被分配一个ChannelHandleContext，其代表了ChannelPipeline和ChannelHandle的绑定。

Netty中有两种发送消息的方式，可以直接写到Channel上，该方式会导致消息从ChannelPipeline的尾端开始流动，第二种是写到ChannelHandleContext，该方式会到ChannelPipeline链上的下一个ChannelHandle。

![image-20210319211310568](D:\学习笔记\image-20210319211310568.png)

接下来我们将研究 3 个 ChannelHandler 的子类型：编码器、解码器和 SimpleChannel

InboundHandler<T> —— ChannelInboundHandlerAdapter 的一个子类。

#### 编码器和解码器

当你通过Netty发送或者接受消息的的时候，就将会发生一次数据转换。入站消息就会被解码，出战消息就会被编码。因为入站将字节转换为我们想要的格式比如字符串，而出战是将字符串转换为字节，因为网络数据总是以字节进行传输的。

通常，编解码的名称类似于 ByteToMessageDecoder，MessageToByteEncode。所有由Netty提供了编码器/解码器都实现了ChannelOutboundHandle或者ChannelInboundHandle接口。

#### 抽象类SimpleChannelInboundHandler

该类适用于接受消息，而不适用于向远端发送院系。

#### 引导

两种类型的引导，一种是客户端，一种是服务端。

![image-20210319211801084](D:\学习笔记\image-20210319211801084.png)

第一个绑定端口或者连接主机很容易理解。

第二行为什么Bootstrap需要一个EventLoopGroup而ServerBootstrap需要两个EventLoopGroup

因为当服务器绑定端口的时候需要多个EventLoopGroup来绑定多个ServerChannel，用于accpet建立连接，而当与远端连接建立的时候，需要创建一个子Channel，然后需要将子Channel注册到一个EventLoop中，所以说需要两个Group，第一个Group里面的EventLoop用于绑定端口，第二个Group用于处理每个建立连接创建的子Channel发生的所有I/O事件等。职责不同。

而客户端则不需要，只需要和服务端建立连接，所以只需要一个EventLoop

![image-20210319212248502](D:\学习笔记\image-20210319212248502.png)

## 传输

流经网络的数据总是相同的类型：字节。

### 案例研究：数据迁移

#### 不通过Netty使用OIO和NIO

![image-20210319212553454](D:\学习笔记\image-20210319212553454.png)

```java
public class PlainOioServer { 
public void serve(int port) throws IOException { 
final ServerSocket socket = new ServerSocket(port);
try { 
for (;;) { 
final Socket clientSocket = socket.accept();
System.out.println(
 "Accepted connection from " + clientSocket);
new Thread(new Runnable() { 
@Override
public void run() { 
OutputStream out;
try { 
out = clientSocket.getOutputStream();
out.write("Hi!\r\n".getBytes( 
 Charset.forName("UTF-8")));
out.flush();
clientSocket.close(); 
 } 
catch (IOException e) { 
e.printStackTrace();
 } 
finally { 
try { 
clientSocket.close();
 } 
catch (IOException ex) { 
// ignore on close
 } 
 } 
 } 
}).start();
 } 
 } 
catch (IOException e) { 
e.printStackTrace();
 } 
 } 
}
```

这段代码完全可以处理中等数量的并发客户端。每来一个连接都需要一个线程。耗费资源。

决定改为异步网络编程

![image-20210319212727862](D:\学习笔记\image-20210319212727862.png)

```java
public class PlainNioServer { 
public void serve(int port) throws IOException { 
ServerSocketChannel serverChannel = ServerSocketChannel.open();
serverChannel.configureBlocking(false);
 ServerSocket ssocket = serverChannel.socket();
 InetSocketAddress address = new InetSocketAddress(port);
 ssocket.bind(address); 
Selector selector = Selector.open(); 
serverChannel.register(selector, SelectionKey.OP_ACCEPT); 
 final ByteBuffer msg = ByteBuffer.wrap("Hi!\r\n".getBytes());
 for (;;) { 
 try { 
selector.select();
 } catch (IOException ex) { 
ex.printStackTrace();
// handle exception
break;
 } 
Set<SelectionKey> readyKeys = selector.selectedKeys(); 
 Iterator<SelectionKey> iterator = readyKeys.iterator();
 while (iterator.hasNext()) { 
 SelectionKey key = iterator.next();
 iterator.remove();
try { 
if (key.isAcceptable()) { 
ServerSocketChannel server = 
 (ServerSocketChannel)key.channel();
SocketChannel client = server.accept();
client.configureBlocking(false);
client.register(selector, SelectionKey.OP_WRITE | 
 SelectionKey.OP_READ, msg.duplicate());
 System.out.println(
 "Accepted connection from " + client);
 } 
if (key.isWritable()) { 
SocketChannel client = 
 (SocketChannel)key.channel();
ByteBuffer buffer = 
 (ByteBuffer)key.attachment();
while (buffer.hasRemaining()) { 
if (client.write(buffer) == 0) { 
break;
 } 
 } 
client.close();
 } 
 } catch (IOException ex) { 
key.cancel();
try { 
key.channel().close();

 } catch (IOException cex) { 
// ignore on close
 } 
 } 
 } 
 } 
 } 
}
```

可以看到，OIO和NIO想干的事情一样，但是代码却完全不同。

####  通过Netty来使用OIO和NIO

而Netty就解决了差异化，做相同的事情不同的传输方式只会改动一点。

![image-20210319212919345](D:\学习笔记\image-20210319212919345.png)

我们现在使用Netty和非阻塞I/O来实现相同的事情。

![image-20210319213230351](D:\学习笔记\image-20210319213230351.png)

可以看到只有两处不一样，NioSocketChannel和NioEventLoopGroup。

因为 Netty 为每种传输的实现都暴露了相同的 API，所以无论选用哪一种传输的实现，你的代码都仍然几乎不受影响。在所有的情况下，传输的实现都依赖于 interface Channel、

ChannelPipeline 和 ChannelHandler。

### 传输API

![image-20210319213450953](D:\学习笔记\image-20210319213450953.png)

如图，每个Channel会被分配一个ChannelPipeLine和ChannelConfig。

ChannelHandler的典型用途：

- 将数据从一种格式转换为另外一种格式
- 提供异常的通知
- 提供当Channel注册到EventLoop或者注销从EventLoop注销的通知

![image-20210319213651415](D:\学习笔记\image-20210319213651415.png)

writeAndFlush()相当于不用流经到ChannelPipeline的尾端，直接写入底层的Socket通信进行传输。

![image-20210319213825412](D:\学习笔记\image-20210319213825412.png)

Netty的Channel是线程安全的。

![image-20210319213921802](D:\学习笔记\image-20210319213921802.png)

### 内置的传输

下列显式了Netty提供的传输

- NIO：JAVA提供的NIO非阻塞I/O传输。
- Epoll：Linux下专属I/O，可以使用Linux操作系统提供的Epoll原语，性能好，比JAVA提供的NIO速度要快。完全非阻塞的。
- OIO：JVAA原生阻塞I/O传输。
- Local ：JVM本地传输，不需要经过网络。
- Embedded : 该传输用于测试，允许使用ChannelHandle而又不需要真正基于网络传输的。

#### Nio-非阻塞I/O

选择器基本的概念就相当于一个注册表，注册每个通道所关心的事件，如果发生了则执行相应的操作。

可以关心的事情有

- Accept接受事件
- Connet连接事件
- REDA读数据事件
- WRITE写数据事件

![image-20210319214612704](D:\学习笔记\image-20210319214612704.png)

![image-20210319214624027](D:\学习笔记\image-20210319214624027.png)

零拷贝：是一种目前只有在Nio和Epoll传输的使用才有的特性。可以使得减少数据的拷贝和上下文的切换。

#### Epoll—用于Linux的本地非阻塞传输

JAVA NIO虽然也提供了相应的抽象，但是因为JDK为了跨平台，所以必须做出妥协，性能会比Epoll慢。

该Linux下的函数提供了比旧的select，poll系统调用更好的性能。

并且Netty也为Linux提供了一组Nio API

如果想要使用Epoll的话，首先必须是在Linux操作系统上，并且需要将NioEventLoopGroup和NioSocketChannel改为EpollEventLoopGroup和EpollSocketChannel。

####  OIO—旧的阻塞I/O

你可能会想为什么Netty是异步的，怎么实现的阻塞方式呢？

那是因为它指定了等待一个I/O操作的最大时间，如果在指定的时间内还没有没有完成相应的操作，那么会抛出异常，Netty将捕获这个异常将继续处理循环，在下一次的时候继续尝试。

#### 用于JVM内部通信的Local传输

支持在同一个JVM中运行的客户端和服务端之间的异步通信。并且服务端相关的Channel不需要绑定网络地址，只需要绑定端口。

#### Embedded传输

额外的传输，可以将一组ChannelHandler作为帮助器类嵌入到其他的ChannelHandler内部。

### 传输的用例

![image-20210319221444341](D:\学习笔记\image-20210319221444341.png)

## ByteBuf(Netty字节容器)

JAVA NIO的Bytebuffer偏复杂了一些，Netty使用了ByteBuf来替代ByteBuffer

ByteBuf API的优点

- 不需要调用flip()方法
- 同时具有read索引下标和写索引下标
- 支持方法的链式调用
- 支持引用计数
- 支持池化
- 通过内置的复合缓冲区实现了透明的零拷贝

### ByteBuf类

#### 它是如何工作的

维护了两个不同的索引，readindex,writeindex，一个用于读取，一个用于写入。

![image-20210319222832304](D:\学习笔记\image-20210319222832304.png)

#### ByteBuf的使用模式

有多种不同的申请ByteBuf的方式，比如面向堆内存的，面向直接缓冲区的。

##### 堆缓冲区

最常用的ByteBuf模式是将数据存储在JVM的堆空间里面。这种模式被称为支撑数组，它能在没有池化的情况下快速分配和释放。这种模式被称为支持数组。

![image-20210319223418816](D:\学习笔记\image-20210319223418816.png)

##### 直接缓冲区

直接缓冲区是另外一种ByteBuf模式，直接与操作系统进行交互，通过JVM堆上的DirectByteBuffer来间接操作该直接缓冲区。相当于在JVM之外开辟了一块内存空间。

允许JVM通过JNI本地调用来分配内存，为了提高性能。

ByteBuffer的java doc明确指出：直接缓冲区的内容将驻留在会被垃圾收集回收的堆之外。经过网络传输的时候不需要走JVM 堆栈 速度更快。并且如果你的缓冲区不是直接缓冲区，那么会创建一个临时直接缓冲区然后将数据放入到临时直接缓冲区中，然后写出去。

缺点就是申请和分配都比较耗费资源。

![image-20210319225005095](D:\学习笔记\image-20210319225005095.png)

##### 复合缓冲区

为多个ByteBuf提供了一个聚合视图。在这里你可以添加或者删除ByteBuf实例，这是一个JDK ByteBuf完全缺失的特性。

Netty通过一个ByteBuf子类——CompositeByteBuf——实现了这个模式，它提供了将多个缓冲区表示为单个合并缓冲区的虚拟表示。

该类中的ByteBuf可能会同时包含一个或者多个ByteBuf，所以可能同时包含直接缓冲区和非直接缓冲区。如果只有一个实例，那么调用hashArray()就返回该值，否则多个实例的情况下都返回false。

```java
 ByteBuf buffer = Unpooled.buffer(10);
        ByteBuf byteBuf = Unpooled.directBuffer(8);
        CompositeByteBuf byteBufs = Unpooled.compositeBuffer();
        byteBufs.addComponents(buffer,byteBuf);

        byteBufs.forEach(System.out::println);
```

输出结果：

```java
UnpooledSlicedByteBuf(ridx: 0, widx: 0, cap: 0/0, unwrapped: UnpooledByteBufAllocator$InstrumentedUnpooledUnsafeHeapByteBuf(ridx: 0, widx: 0, cap: 10))
UnpooledSlicedByteBuf(ridx: 0, widx: 0, cap: 0/0, unwrapped: UnpooledByteBufAllocator$InstrumentedUnpooledUnsafeNoCleanerDirectByteBuf(ridx: 0, widx: 0, cap: 8))
```

![image-20210320151230712](D:\学习笔记\image-20210320151230712.png)

使用原生JAVA NIO比较复杂，还要有数组。

![image-20210320151256506](D:\学习笔记\image-20210320151256506.png)

![image-20210320151313369](D:\学习笔记\image-20210320151313369.png)

这种方式非常适合JDK NIO里面的Scatter/Gather I/O操作。分散聚合操作。

### 字节级操作

ByteBuf提供了许多超出基本读，写操作的方法用于修改它的数据。

#### 随机访问索引

ByteBuf的索引是从零开始的，第一个字节的索引是零，最后一个字节的索引是容量-1。

![image-20210320151914349](D:\学习笔记\image-20210320151914349.png)

#### 顺序访问索引

同时具有读写索引，这个过程中读索引不能超过写索引，写索引不能超过容量-1。

![image-20210320152107623](D:\学习笔记\image-20210320152107623.png)

#### 可丢弃字节

上图有可丢弃字节，可以使用discardReadBytes()方法，可以丢弃它们并且回收空间。

![image-20210320152222847](D:\学习笔记\image-20210320152222847.png)

相当于就是将readIndex置为丢弃字节的位置，然后整体向左移动丢弃的字节数量。扩大可写字节数量。我们不是非常建议这样做，因为数据整体移动开销大，除非内存非常宝贵的时候。

#### 可读字节

如果调用一个缓冲区的 readBytes方法，那么readIndex会移动，如果传入的是一个ByteBuf，那么该传入的ByteBuf的writeIndex也会移动。

```java
readBytes(ByteBuf dest);
```

![image-20210320152523318](D:\学习笔记\image-20210320152523318.png)

#### 可写字节

如果调用ByteBuf的wirteByte()方法，那么writeIndex会移动，如果传入的参数是一个ByteBuf，那么该ByteBuf的readIndex也会移动。

```java
writeBytes(ByteBuf dest)
```

```java
// Fills the writable bytes of a buffer with random integers.
ByteBuf buffer = ...;
while (buffer.writableBytes() >= 4) { 
buffer.writeInt(random.nextInt());
}
```

需要保证目标buffer可以容纳写入的数量，否则就会抛出异常。

#### 索引管理

可以调用markReaderIndex()，markWriterIndex()，resetWriterIndex()，resetReaderIndex()来标记和重置ByteBuf的读写索引，类似于ByteBuffer的position(int newPosition)一样的效果。

​	![image-20210320153207551](D:\学习笔记\image-20210320153207551.png)

可以使用clear来同时清零读写索引，轻量级的，不会移动数据。

![image-20210320153237721](D:\学习笔记\image-20210320153237721.png)

#### 查找操作

最简单的方式就是使用indexOf()操作来获取想要的值第一次出现的索引index。

较复杂的查找可以使用**ByteBufProcessor**。

```java
boolean process(byte value)
```

假设你的应用程序需要和所谓的包含有以NULL结尾的内容的Flash套接字集成。

![image-20210320153741102](D:\学习笔记\image-20210320153741102.png)

#### 派生缓冲区

派生缓冲区为ByteBuf提供了专门以特别的方式来提供呈现其内容的视图。

- duplicate() : 生成一个相同的ByteBuf。

- slice(int index , int length) ： 生成一个该ByteBuf内容的子集。
- Unpooled.unmodifiableBuffer(...) 生成只读的ByteBuf
- readSlice(int i) 

上面的方法都具有自己的读写索引，但是修改会互相影响。

如果需要一个真实拷贝复制，那么调用copy方法。

```java
      Charset charset = Charset.forName("UTF-8");
        ByteBuf byteBuf = Unpooled.copiedBuffer("Netty in Action rocks ！", charset);
        ByteBuf slice = byteBuf.slice(0, 15);
        byteBuf.setByte(0,'J');
       assert byteBuf.getByte(0) == slice.getByte(0);
```

上面会成立，因为是共享的。

```java
        Charset charset = Charset.forName("UTF-8");
        ByteBuf byteBuf = Unpooled.copiedBuffer("Netty in Action rocks ！", charset);
        ByteBuf byteBuf1 = byteBuf.readSlice(3); //读取三个字节，该三个字节形成一个ByteBuf。
        byte[] bytes1 = new byte[byteBuf1.readableBytes()];
        while (byteBuf1.isReadable()){
            byteBuf1.readBytes(bytes1);
        }
        System.out.println(new String(bytes1).toString());
        byte[] bytes = new byte[byteBuf.readableBytes()];
        while (byteBuf.isReadable()){
            byteBuf.readBytes(bytes);
        }
        System.out.println(new String(bytes));
```

![image-20210320162513960](D:\学习笔记\image-20210320162513960.png)

#### 读写操作

- get()和set()操作从指定的索引开始，并且保持索引不变。
- read()和write()操作，索引变化。

get方法合集：

![image-20210320163011495](D:\学习笔记\image-20210320163011495.png)

![image-20210320163051616](D:\学习笔记\image-20210320163051616.png)

![image-20210320163112228](D:\学习笔记\image-20210320163112228.png)

read操作

![image-20210320163253623](D:\学习笔记\image-20210320163253623.png)

![image-20210320163301444](D:\学习笔记\image-20210320163301444.png)

![image-20210320163324909](D:\学习笔记\image-20210320163324909.png)

#### 更多的操作

还有一些其他的操作，比如isReadable()，isWriteable()等

![image-20210320163405965](D:\学习笔记\image-20210320163405965.png)

### ByteBufHolder接口

我们经常除了数据负载之外，还要存放一些属性值，比如Http的响应里面的状态码和Cookie。

该接口可以提供高级特性，比如缓冲池化，可以从池中获取ByteBuf，使用完可以还回池子。

![image-20210320165037339](D:\学习笔记\image-20210320165037339.png)

### ByteBuf分配

在该节中我们将描述管理ByteBuf实例的不同方式。

#### 按需分配 ：ByteBufAllocator接口

为了降低分配和释放内存的开销，Netty通过ByteBufAllocator实现了ByteBuf的池化。

![image-20210320165541428](D:\学习笔记\image-20210320165541428.png)

可以通过Channel(每个Channel都可以有不同的ByteBufAllocator)或者绑定到ChannelHandle的ChannelHandleContext上获取一个ByteBufAllocator。

![image-20210320170106341](D:\学习笔记\image-20210320170106341.png)

Netty提供了两种实现：PooledByteBufAllocator和UnpooledByteBufAllocator。前者池化了ByteBuf的实例以提高性能，并最大限度减少内存碎片。因为使用了jemalloc的高效方法来分配内存。第二种则是每次都会创建一个新的ByteBuf。

#### Unpooled 缓冲区

创建未池化的ByteBuf

![image-20210320171620438](D:\学习笔记\image-20210320171620438.png)

#### ByteBufUtil类

用于操作ByteBuf的静态辅助方法。

hexdump()以16进制的方式打印ByteBuf的内容，equlas方法比较两个缓冲区是否相等 等等。

### 引用计数

引用计数是一种在某个对象所持有的资源不再被其他对象所引用的时候一种优化手段。

只要引用计数大于0，那么就不会被释放，如果等于0相当于资源没被使用，那么就释放掉。

![image-20210320172014272](D:\学习笔记\image-20210320172014272.png)

## ChannelHandler和ChannelPipeline

### ChannelHandler家族

#### Channel生命周期

![image-20210320172149544](D:\学习笔记\image-20210320172149544.png)

![image-20210320172158017](D:\学习笔记\image-20210320172158017.png)

#### ChannelHandler生命周期

![image-20210320172341923](D:\学习笔记\image-20210320172341923.png)

Netty 定义了下面两个重要的 ChannelHandler 子接口：

ChannelInboundHandler——处理入站数据以及各种状态变化；

ChannelOutboundHandler——处理出站数据并且允许拦截所有的操作。

在接下来的章节中，我们将详细地讨论这些子接口。

#### ChannelInboundHandler接口

![image-20210320172422950](D:\学习笔记\image-20210320172422950.png)

![image-20210320172443679](D:\学习笔记\image-20210320172443679.png)

![image-20210320172458943](D:\学习笔记\image-20210320172458943.png)

SimpeDiscardHandler会自动释放资源。



#### ChannelOutboundHandler接口

![image-20210320172525695](D:\学习笔记\image-20210320172525695.png)

ChannelPromise是ChannelFutrue的一个子类。

#### ChannelHandler适配器

你可以使用 ChannelInboundHandlerAdapter 和 ChannelOutboundHandlerAdapter

类作为自己的 ChannelHandler 的起始点。

![image-20210320172627200](D:\学习笔记\image-20210320172627200.png)

可以理解为适配器模式。

#### 资源管理

Netty使用引用计数来处理池化的ByteBuf,所以在使用完某个ByteBuf后，调整引用计数是很重要的。

Netty提供了检测是否出现内存泄漏的功能。

![image-20210320172954922](D:\学习笔记\image-20210320172954922.png)

![image-20210320173016513](D:\学习笔记\image-20210320173016513.png)

ReferenceCountUtil.release();

![image-20210320173104068](D:\学习笔记\image-20210320173104068.png)

总之，如果一个消息被消费了或者丢弃了，并且没有传递给ChannelPipeline的下一个ChannelOutBoundHandler，那么用户就有责任释放。

### ChannelPipeline接口

每个Channel都会被分配一个新的ChannelPipeline，关联是永久性的。Channel不能附加别的，也不能删除分配的ChannelPipeline。事件被处理，然后通过调用ChannelHandleContext实现转发给下一个ChannelHandler。

![image-20210320173944728](D:\学习笔记\image-20210320173944728.png)

Netty总是将入站口左侧作为头部，右侧作为尾部。

#### 修改ChannelPipeline

ChannlHandler可以通过添加或者删除其他的ChannelHandler来实时的修改ChannelPipeLine的布局。这是ChannelHandler最重要的功能之一。

![image-20210320174316179](D:\学习笔记\image-20210320174316179.png)

![image-20210320174357197](D:\学习笔记\image-20210320174357197.png)

![image-20210320174415804](D:\学习笔记\image-20210320174415804.png)

#### 触发事件

![image-20210320174650318](D:\学习笔记\image-20210320174650318.png)

调用当前调用位置ChannelInboundHandler的下一个xxx方法，比如上图的调用下一个Registered方法。

![image-20210320174855761](D:\学习笔记\image-20210320174855761.png)

将调用ChannelPipeline中的下一个ChannelOutboundHandler的bind()方法。

看当前ChannelPipeLine的位置处于哪。

总结

- ChnnelPileline保存了与Channel相关联的ChannelHandler。
- ChannelPipeline可以根据需要，添加或者删除ChannelHandler。
- 有丰富的API调用。

### ChannelHandlerContext接口

ChannelHandlerContext代表了ChannelPipeline和ChannelHandler之间的关系。	ChannelHandlerContext的作用主要是为了让所关联的ChannelHandler和ChannelPipeline其他链上的ChannelHandler交互。

并且有许多和Channel和ChannelPipeline相同的方法，不同的是如果调用ChannelHandlerContext方法会到下一个ChannelHandler，而Channel和ChannelPipeline都是沿着整个ChannelPipeline传播。

![image-20210320175818139](D:\学习笔记\image-20210320175818139.png)

![image-20210320175906614](D:\学习笔记\image-20210320175906614.png)

#### 使用ChannelHandlerContext

![image-20210320175931745](D:\学习笔记\image-20210320175931745.png)

![image-20210320180014140](D:\学习笔记\image-20210320180014140.png)

channel或者channelPipeline的write将流经整个ChannelPipeline。

![image-20210320180151798](D:\学习笔记\image-20210320180151798.png)

#### ChannelHandler和ChannelHandlerContext的高级用法

![image-20210320180246234](D:\学习笔记\image-20210320180246234.png)

把引用缓存，可能会被使用到ChannelHandler之外，也可能被不同的线程使用。

也可以使用多个ChannelPipe共享同一个ChannelHandler，所以一个ChannelHandler也可以绑定多个ChannelHandlerContext实例，对应的ChannelHandler必须使用@Sharable注解标注。

![image-20210320180711642](D:\学习笔记\image-20210320180711642.png)

![image-20210320181058233](D:\学习笔记\image-20210320181058233.png)

为什么错误呢？首先如果标注Sharable注解，意味ChannelHandler可能会被多个Channel引用，也就意味这会有多个ChannelPipeline，每个Channel又可能会被不同的EventLoop来执行，那么多个线程并发访问ChannelHandler，里面的又是++操作，就会出现线程安全问题，推荐使用原子类。CAS自增。

为何要共享同一个ChannelHandler，在多个ChannelPipeline安装同一个ChannelHandler的常见原因就是用于收集多个Channel的统计信息。

### 异常处理

#### 处理入站异常

![image-20210320182352266](D:\学习笔记\image-20210320182352266.png)

- ChannelHandler.exceptionCaught()的默认实现是简单地将当前异常转发给

ChannelPipeline 中的下一个 ChannelHandler；

- 如果异常到达了 ChannelPipeline 的尾端，它将会被记录为未被处理；

- 要想定义自定义的处理逻辑，你需要重写 exceptionCaught()方法。然后你需要决定

是否需要将该异常传播出去。

出现异常，会一直向下一个寻找，

#### 处理出战异常

- 每个出战操作都会返回一个ChannelFuture。

  ![image-20210320182659861](D:\学习笔记\image-20210320182659861.png)

![image-20210320182708839](D:\学习笔记\image-20210320182708839.png)

## EventLoop和线程模型

### 线程模型概述

线程模型确定了代码的执行方式。

### EventLoop接口

运行任何来处理连接的生命周期内发生的事件是网络框架的基本功能。Netty使用了事件循环EventLoop来适配该术语。

![image-20210320183224619](D:\学习笔记\image-20210320183224619.png)

该代码体现了事件循环的基本思想，其中每个任务都是一个Runnable的实例。

![image-20210320222148330](D:\学习笔记\image-20210320222148330.png)

在这个模型中，一个EventLoop将由一个永远都不会改动的Thread驱动，同时任务可以直接交给EventLoop实现，以立即执行或者调度执行。根据可配置和可用核心不同，可能会创建多个EventLoop实例以优化资源的使用。并且单个EventLoop可能会被服务于多个Channel。

![image-20210320222437426](D:\学习笔记\image-20210320222437426.png)

根据FIFO顺序执行队列的任务。

#### Netty 4 中的 I/O和事件处理

在Netty4中，所有的IO操作和事件都被分配给了EventLoop的那个线程来执行。

#### Netty 3 的I/O事件处理

在以前版本的线程模型中只保证了入站事件会在I/O线程中(也就是Nett4里面的EventLoop)。所有的出战事件都由调用线程处理，可能是I/O线程也可以是别的线程。这样的话，可能会导致多个线程同时访问同一个ChannelHandler，出现并发问题，需要同步。

### 任务调度

#### JDK的任务调度API

![image-20210320223009323](D:\学习笔记\image-20210320223009323.png)

![image-20210320223022804](D:\学习笔记\image-20210320223022804.png)

#### 使用EventLoop调度任务

![image-20210320223725686](D:\学习笔记\image-20210320223725686.png)

![image-20210320223735157](D:\学习笔记\image-20210320223735157.png)

eventLoop()提供了许多调度任务的方法，基于JUC之上的方法。

![image-20210320223832563](D:\学习笔记\image-20210320223832563.png)

### 实现细节

#### 线程管理

Netty的卓越性能取决于对于当前执行的Thread身份的确定，也就是说，确定它是否是分配给当前Channel以及EventLoop的那一个线程。

如果当前调用线程正是支撑EventLoop的线程，那么所提交的代码块会被立刻执行。否则EventLoop将调度该任务以编稍后执行，并将它放入内部队列中。当EventLoop下次去处理它的事件到时候，它会执行队列中的那些任务/事件。

这为什么就解释了任何的Thread与Channel直接交互而不要在ChannelHandler中进行同步呢？

![image-20210321132254209](D:\学习笔记\image-20210321132254209.png)

**为什么？**

**注意：每个EventLoop都有它的任务队列。**

这块没想明白

#### EventLoop/线程的分配

##### 异步传输

异步传输实现只使用了少量的EventLoop，而且在当前的线程模型中，可能会被多个Channel所共享使用。

![image-20210321132833915](D:\学习笔记\image-20210321132833915.png)

另外需要注意的点是EventlLoop方式对ThreadLocal的影响，因为一个EventLoop通常会被用于支撑多个EventLoop，所以这对于所相关联的Channel来说，ThreadLocal都是一样。

##### 阻塞传输 

与传统的Socket类似。

![image-20210321133039200](D:\学习笔记\image-20210321133039200.png)

OIO，一个EventLoop对应一个Channel。

## 引导

如何将上面所学的各种组件，比如ChannelHandler，EventLoop，ChannelPipeLine连接起来，就是引导。

分为客户端和服务端。

### Bootstrap

![image-20210321133159254](D:\学习笔记\image-20210321133159254.png)

为什么引导类是Cloneable的。

有时候可能会需要多个具有类似配置的或者完全相同的引导类。

![image-20210321133309239](D:\学习笔记\image-20210321133309239.png)

![image-20210321133317155](D:\学习笔记\image-20210321133317155.png)

### 引导客户端和无连接协议



Bootstrap被用于客户端或者使用了无连接协议的应用程序中。

![image-20210321133425640](D:\学习笔记\image-20210321133425640.png)

![image-20210321133443139](D:\学习笔记\image-20210321133443139.png)

#### 引导客端

![image-20210321133517814](D:\学习笔记\image-20210321133517814.png)

![image-20210321133540463](D:\学习笔记\image-20210321133540463.png)

#### Channel和EventLoopGroup 的兼容性

![image-20210321133639616](D:\学习笔记\image-20210321133639616.png)

![image-20210321133708122](D:\学习笔记\image-20210321133708122.png)

不能乱使用。

### 引导服务器

#### ServerBootstrap类

![image-20210321133734701](D:\学习笔记\image-20210321133734701.png)



#### 引导服务器

可以理解为options和attr都是为sockerChannel配置的，也就是相当于Nio里面的ServerSocketChannel的配置，当accept建立连接后，要创建一个Channel并且分配给worker的NioEventLoopGroup里面的一个EventLoop，channelHandler和childAtter都是配置worker的事件循环组的事件循环的。

![image-20210321134443300](D:\学习笔记\image-20210321134443300.png)

![image-20210321134452028](D:\学习笔记\image-20210321134452028.png)

### 从Channel引导客户端

假设你的服务器正在处理一个客户端的请求，这个请求需要你的服务器充当第三方系统的客户端。

在这种情况下，你可能需要在accept接受后的Channel的基础上，再新建一个客户端Channel。

一个很好的解决方案就是：通过已被接受的子Channel所属的EventLoop传递给BootStrap的group方法来共享该Eventloop。

![image-20210321135119570](D:\学习笔记\image-20210321135119570.png)

![image-20210321135135654](D:\学习笔记\image-20210321135135654.png)

### 在引导过程中添加多个Channelhandler

![image-20210321135307078](D:\学习笔记\image-20210321135307078.png)

### 使用Netty的ChannelOption和属性

![image-20210321135420554](D:\学习笔记\image-20210321135420554.png)

Netty提供lAttrubuteMap抽象，Attributekey<T>。使用这些属性。

### 引导DatagramChannel

![image-20210321135608788](D:\学习笔记\image-20210321135608788.png)

OIoDatagramChannel

### 关闭

![image-20210321135634750](D:\学习笔记\image-20210321135634750.png)

shutdownGracefully()也是异步方法，需要阻塞等待直到它关闭完成。

## 单元测试

特殊的Channel实现 ：EmbeddedChannel，它是Netty专门为改进针对ChannelHandler的单元测试提供的。

![image-20210321135846945](D:\学习笔记\image-20210321135846945.png)

![image-20210321135854985](D:\学习笔记\image-20210321135854985.png)

相当于可以在一端又接受入站消息，也接受出战消息。也就是发送入站消息，写可以发送出战消息。

写出站消息，会被读取出战的消息所捕获

写入站消息，会被入站捕获。

### 测试入站消息

特定的解码器将产生固定为3字节大小的帧。

![image-20210321144112053](D:\学习笔记\image-20210321144112053.png)

![image-20210321144137234](D:\学习笔记\image-20210321144137234.png)	

![image-20210321145802212](D:\学习笔记\image-20210321145802212.png)

![image-20210321145809426](D:\学习笔记\image-20210321145809426.png)

### 测试出战消息

为出战的每个数据，都将负数转换为绝对值的编码器。

![image-20210321145854368](D:\学习笔记\image-20210321145854368.png)

![image-20210321145908351](D:\学习笔记\image-20210321145908351.png)

![image-20210321145924094](D:\学习笔记\image-20210321145924094.png)

### 测试异常处理

![image-20210321150034537](D:\学习笔记\image-20210321150034537.png)

![image-20210321150050514](D:\学习笔记\image-20210321150050514.png)

## 编解码器框架

### 解码器

- 将字节解码为消息ByteToMessageDecoder和ReplayingDecode；
- 将一种消息类型解码为另外一种 MessageToMessageDecoder；

#### 抽象类ByteToMessageDecoder

![image-20210322103807803](D:\学习笔记\image-20210322103807803.png)

decodeLast相当于在通道关闭的时候调用一次。

![image-20210322103846607](D:\学习笔记\image-20210322103846607.png)

![image-20210322103857223](D:\学习笔记\image-20210322103857223.png)

一旦消息被解码或者编码，它会呗ReferenceCountUtil.release(message)调用自动释放。

ReferenceCountUtil.retain(message)方法来增加引用计数。上面的图就是in里面的数据会被释放，但是List集合out里面的数据不会被释放。

#### 抽象类ReplayingDecoder

扩展了ByteToMessageDecoder，使得我们上面的代码不用每次都判断是否大于一个Int的字节数量。

参数S代表了用于状态管理的类型，Void代表不需要状态管理

![image-20210322104248058](D:\学习笔记\image-20210322104248058.png)

如果没有足够的字节数会抛出Error并且被父类中被处理。

两种方式各有好处，ReplayingDecoder 稍慢于 ByteToMessageDecoder。‘

并且不是所有的ByteBuf操作ReplayingDecoder类都支持

还有更多的解码器

- LineBasedFrameDecoder   这个类在Netty内部也有使用，使用了行尾控制符(\n 或者 \r\n)来解析数据

- HttpObjectDecoder  一个Http数据的解码器

  

#### 抽象类 MessageToMessageDecoder

![image-20210322104630988](D:\学习笔记\image-20210322104630988.png)

将POJO转换为另外一种。

可以使用将Integer转换为String。

![image-20210322104800798](D:\学习笔记\image-20210322104800798.png)

```java
package netty;

import io.netty.channel.ChannelHandlerContext;
import io.netty.handler.codec.MessageToMessageDecoder;

import java.util.List;

/**
 * @program: TheFirstNettyApplication
 * @description:
 * @author: Mr.Feng
 * @create: 2021-03-22 10:47
 **/
public class IntegerToStringDecoder extends MessageToMessageDecoder<Integer> {
    @Override
    protected void decode(ChannelHandlerContext channelHandlerContext, Integer integer, List<Object> list) throws Exception {
        list.add(String.valueOf(integer));
    }
}

```

#### TooLongFrameException类

由于Netty是一个异步框架，所以需要在字节可以解码之前缓存它们，因此不能让解码器缓冲大量的数据以至于耗尽内存。Netty提供了TooLongFrameException类。

### 编码器

- 将消息转换为字节
- 将消息转换为另外一种消息类型

#### 抽象类MessageToByteEncoder

![image-20210322105437744](D:\学习笔记\image-20210322105437744.png)



![image-20210322105553497](D:\学习笔记\image-20210322105553497.png)

![image-20210322105608323](D:\学习笔记\image-20210322105608323.png)

```java
package netty;

import io.netty.buffer.ByteBuf;
import io.netty.channel.ChannelHandlerContext;
import io.netty.handler.codec.MessageToByteEncoder;

/**
 * @program: TheFirstNettyApplication
 * @description:
 * @author: Mr.Feng
 * @create: 2021-03-22 10:56
 **/
public class ShortToByteMessage extends MessageToByteEncoder<Short> {
    @Override
    protected void encode(ChannelHandlerContext channelHandlerContext, Short aShort, ByteBuf byteBuf) throws Exception {
        byteBuf.writeShort(aShort);
    }
}

```

#### 抽象类 MessageToMessageEncoder

![image-20210322105802529](D:\学习笔记\image-20210322105802529.png)

![image-20210322105853307](D:\学习笔记\image-20210322105853307.png)

​	

泛型里面写的是你想要编码的数据，很明显，因为你要获取想要进行编码的数据，并且编码之前的类型是什么，可以在泛型里面指定。

### 抽象的编解码器类

该类可以既具有编码的功能也具有解码的功能，意味这同时实现了ChannelInboundHandler和ChannelOutBoundHandler。但是我们不是特别推荐，因为可能会对代码的复用性造成影响。

#### 抽象类 ByteToMessageCode

![image-20210322110332019](D:\学习笔记\image-20210322110332019.png)

#### 抽象类 MessageToMessageCodec

![image-20210322110349419](D:\学习笔记\image-20210322110349419.png)

```java
public class IntegerToStringCodec extends MessageToMessageCodec<Integer,String> {

    /**
    * @Description:  encode方法会将String编码为Interger  decoder会将消息解码为String
    * @Param:
    * @return:
    * @Date: 2021/3/22
    */
    @Override
    protected void encode(ChannelHandlerContext ctx, String msg, List<Object> out) throws Exception {
      out.add(Integer.valueOf(msg));
    }

    @Override
    protected void decode(ChannelHandlerContext ctx, Integer msg, List<Object> out) throws Exception {
       out.add(String.valueOf(msg));
    }
}
```

Websocket协议，这个协议可以实现服务器和Web浏览器之间的全双工通信。

我们的WebSocketConvertHandler在参数化MessageToMessageCodec将使用INBOUND_IN类型的WebSocketFrame，以及OUTBOUND_IN类型的MyWebSocketFrame。

```java
package netty;

import io.netty.buffer.ByteBuf;
import io.netty.channel.ChannelHandlerContext;
import io.netty.handler.codec.MessageToMessageCodec;
import io.netty.handler.codec.http.websocketx.*;

import java.util.List;

/**
 * @program: TheFirstNettyApplication
 * @description:
 * @author: Mr.Feng
 * @create: 2021-03-22 11:21
 **/
public class WebSocketConvertHandler extends MessageToMessageCodec<WebSocketFrame, WebSocketConvertHandler.MyWebSocketFrame> {


    /**
    * @Description:  泛型的第一个位置是编码后的数据，第二个是解码后的数据
    * @Param:
    * @return:
    * @Date: 2021/3/22
    */
    @Override
    protected void encode(ChannelHandlerContext ctx, MyWebSocketFrame msg, List<Object> out) throws Exception {
        ByteBuf payload = msg.getData().duplicate().retain();
        switch (msg.getType()){
            case BINARY:
                out.add(new BinaryWebSocketFrame(payload));
                break;
            case COLSE:
                out.add(new CloseWebSocketFrame(true,0,payload));
                break;
            case PNIT:
                out.add(new PingWebSocketFrame(payload));
                break;
            case PONG:
                out.add(new PongWebSocketFrame(payload));
                break;
            case TEXT:
                out.add(new TextWebSocketFrame(payload));
                break;
            case CONTINUATION:
                out.add(new ContinuationWebSocketFrame(payload));
                break;
            default:
                throw new IllegalStateException(
                        "Unsupported websocket msg " + msg);

        }
    }

    @Override
    protected void decode(ChannelHandlerContext ctx, WebSocketFrame msg, List<Object> out) throws Exception {
        ByteBuf payload = msg.content().duplicate().retain();
        if (msg instanceof BinaryWebSocketFrame) {
            out.add(new MyWebSocketFrame(
                    MyWebSocketFrame.FrameType.BINARY, payload));
        } else
        if (msg instanceof CloseWebSocketFrame) {
            out.add(new MyWebSocketFrame (
                    MyWebSocketFrame.FrameType.COLSE, payload));
        } else
        if (msg instanceof PingWebSocketFrame) {
            out.add(new MyWebSocketFrame (
                    MyWebSocketFrame.FrameType.PNIT, payload));
        } else
        if (msg instanceof PongWebSocketFrame) {
            out.add(new MyWebSocketFrame (
                    MyWebSocketFrame.FrameType.PONG, payload));
        } else
        if (msg instanceof TextWebSocketFrame) {
            out.add(new MyWebSocketFrame (
                    MyWebSocketFrame.FrameType.TEXT, payload));
        } else
        if (msg instanceof ContinuationWebSocketFrame) {
            out.add(new MyWebSocketFrame (
                    MyWebSocketFrame.FrameType.CONTINUATION, payload));
        } else
        {
            throw new IllegalStateException(
                    "Unsupported websocket msg " + msg);
        }


    }

    public static final class MyWebSocketFrame {

        public enum FrameType{
            BINARY,
            COLSE,
            PNIT,
            PONG,
            TEXT,
            CONTINUATION
        }
        private final FrameType type;
        private final ByteBuf data;
        public MyWebSocketFrame(FrameType type,ByteBuf data){
            this.type = type;
            this.data = data;
        }

        public FrameType getType() {
            return type;
        }

        public ByteBuf getData() {
            return data;
        }
    }
}
```

#### CombinedChannelDuplexHandler类

```java
public class CombinedChannelDuplexHandler<I extendChannelInboundHandler,O extends ChannelOutboundHandler>
```

这个类充当了容器。

![image-20210322114411304](D:\学习笔记\image-20210322114411304.png)

![image-20210322114506323](D:\学习笔记\image-20210322114506323.png)

![image-20210322114515108](D:\学习笔记\image-20210322114515108.png)

## 预置的ChannelHandler和编解码器

主要内容

- 通过SSL/TLS来保护Netty程序
- 构建居于Netty的Http/Https应用程序
- 处理空闲的连接
- 解码基于分隔符的协议和基于长度的协议
- 写大型数据

### 通过SSL/TLS来保护Netty程序

JAVA为了支持SSL/TLS,提供了net.ssl包。SSLContext和SSLEngine类使得实现加密和解密都非常简单。

Netty通过一个SslHandler的ChannelHander实现利用这个API。其中SslHandler在内部使用SSLEngine来完成实际的工作。

![image-20210322115215489](D:\学习笔记\image-20210322115215489.png)

通过该Handler进行入站数据的解密，和出战数据的加密。

![image-20210322120655801](D:\学习笔记\image-20210322120655801.png)

sshHandler具有一些有用的方法，比如节点之间的验证和协商加密方式。可以通过配置SslHandler来修改它的行为。

![image-20210322120824149](D:\学习笔记\image-20210322120824149.png)

### 构建基于Netty的Http/Https应用程序

Netty为Http/Https提供了一系列的ChannelHandler

#### HTTP解码器丶编码器和编解码器

Http是居于请求/响应模式的：客户端向服务器发送一个请求，然后服务端响应请求。

![image-20210322121238393](D:\学习笔记\image-20210322121238393.png)

一个完整的Http请求，FullHttpRequest分为请求头，和多个HttpContent，和LastHttpContent标注了该Http请求的结束。

![image-20210322122338707](D:\学习笔记\image-20210322122338707.png)

FullHttpResponce



使用一个类可以自动将它们包装为一个完整的FullHttpRequest或者FullHttpResponse。

HttpObjectAggregator  aggregator聚合器罢了

![image-20210322122439366](D:\学习笔记\image-20210322122439366.png)

将Http请求编码为字节，这个用于客户端发请求进行编码

将http响应编码为字节。 这个适用于服务端响应编码。

将http请求解码为请求头+请求体+结束标注内容。这个适用于服务器接受来自客户端的请求，将请求字节解码。

将http响应解码为响应头+响应体+结束标记内容。这个适用于客户端接受来自服务端的响应然后将响应数据解码																															

```java
public class HttpPipelineInitializer extends ChannelInitializer<Channel> {
    
    private final  boolean client;
    public HttpPipelineInitializer(boolean client){
        this.client = client;
    } 
    @Override
    protected void initChannel(Channel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();
        if (client){
            pipeline.addLast(new HttpRequestEncoder());
            pipeline.addLast(new HttpResponseDecoder());
        }else{
            pipeline.addLast(new HttpRequestDecoder());
            pipeline.addLast(new HttpResponseEncoder());
        }
    }
}
```

#### 聚合HTTP消息

```java
public class HttpAggregatorInitializer extends ChannelInitializer<Channel> {
    private final boolean isClient;
    public HttpAggregatorInitializer(boolean isClient) {
        this.isClient = isClient;
    }
    @Override
    protected void initChannel(Channel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();
        if (isClient){
            pipeline.addLast("codec",new HttpClientCodec());
        }else {
            pipeline.addLast("codec",new HttpServerCodec());
        }
        pipeline.addLast("aggregator",new HttpObjectAggregator(512 * 1024 )); //创建最大的消息大小为512kb的HttpObjectAggregator
    }
}
```

HttpObjectAggregator它可以将多个消息部分合并为 FullHttpRequest 或者 FullHttpResponse 消息

HttpClientCodec等同于new HttpRequestEncoder() + new HttpResponseDecoder()；

HttpServerCodec等同于new HttpRequestDecoder()+new HttpResponseEncoder()；

### Http压缩

可以将数据压缩和解压，使得数据量小，但是可能会增加CPU的负载。

但是推荐文本数据![image-20210322124359926](D:\学习笔记\image-20210322124359926.png)

同时支持gzip和deflate编码。

一般在服务器端会进行压缩 —— **HttpContentCompressor**，客户端接收到信息后进行解压 —— **HttpContentDecompressor**

```java
public class HttpCompressionInitializer extends ChannelInitializer<Channel> {

    private final boolean isClient;
    public HttpCompressionInitializer(boolean isClient) {
        this.isClient = isClient;
    }
    @Override
    protected void initChannel(Channel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();
        if (isClient){
            pipeline.addLast("codec",new HttpClientCodec());
            pipeline.addLast("decompressor",new HttpContentDecompressor());
        }else {
            pipeline.addLast("codec",new HttpServerCodec());
            pipeline.addLast("compressor",new HttpContentCompressor());
        }
        pipeline.addLast(new HttpObjectAggregator(512 * 1024));
    }
}
```

```xml
压缩及其依赖
如果你正在使用的是 JDK 6 或者更早的版本，那么你需要将 JZlib（www.jcraft.com/jzlib/）添加到
CLASSPATH 中以支持压缩功能。 
对于 Maven，请添加以下依赖项： 
<dependency>
<groupId>com.jcraft</groupId>
<artifactId>jzlib</artifactId>
<version>1.1.3</version>
</dependency>
```

### 使用HTTPS

只需要在Http的基础上增加SSL/TLS

```java
public class HttpsCodecInitializer extends ChannelInitializer<Channel> {
    private final SslContext context;
    private final boolean isClient;
    public HttpsCodecInitializer(SslContext context, boolean isClient) {
        this.context = context;
        this.isClient = isClient;
    }
    @Override
    protected void initChannel(Channel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();
        pipeline.addLast("ssl",new SslHandler(context.newEngine(ch.alloc())));
        if (isClient){
            pipeline.addLast("codec", new HttpClientCodec());
        }else {
            pipeline.addLast("codec",new HttpServerCodec());
        }
        pipeline.addLast(new HttpObjectAggregator(512 * 1024));
    }
}
```

### Websocket

Websocket实现了客户端和服务端之间提供了真正的双向数据交换。替换了HTTP轮询的方案。

![image-20210322144531573](D:\学习笔记\image-20210322144531573.png)

![image-20210322144702971](D:\学习笔记\image-20210322144702971.png)

创建一个WebSocket服务器。

```java
package netty;

import io.netty.channel.*;
import io.netty.handler.codec.http.HttpObjectAggregator;
import io.netty.handler.codec.http.HttpServerCodec;
import io.netty.handler.codec.http.websocketx.BinaryWebSocketFrame;
import io.netty.handler.codec.http.websocketx.ContinuationWebSocketFrame;
import io.netty.handler.codec.http.websocketx.TextWebSocketFrame;
import io.netty.handler.codec.http.websocketx.WebSocketServerProtocolHandler;

/**
 * @program: TheFirstNettyApplication
 * @description:
 * @author: Mr.Feng
 * @create: 2021-03-22 14:49
 **/
public class WebSocketServerInitalizer extends ChannelInitializer<Channel> {
    @Override
    protected void initChannel(Channel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();
        pipeline.addLast(new HttpServerCodec());
        pipeline.addLast(new HttpObjectAggregator(512 * 1024));
        pipeline.addLast(new WebSocketServerProtocolHandler("/websocket"));
        pipeline.addLast(new TextFrameHandler());
        pipeline.addLast(new BinaryFrameHandler());
        pipeline.addLast(new ContinuationFrameHandler());
    }
    public static final class TextFrameHandler extends SimpleChannelInboundHandler<TextWebSocketFrame>{

        @Override
        protected void channelRead0(ChannelHandlerContext ctx, TextWebSocketFrame msg) throws Exception {

        }
    }
    public static final class BinaryFrameHandler extends SimpleChannelInboundHandler<BinaryWebSocketFrame>{


        @Override
        protected void channelRead0(ChannelHandlerContext ctx, BinaryWebSocketFrame msg) throws Exception {

        }
    }
    public static final class ContinuationFrameHandler extends SimpleChannelInboundHandler<ContinuationWebSocketFrame>{

        @Override
        protected void channelRead0(ChannelHandlerContext ctx, ContinuationWebSocketFrame msg) throws Exception {

        }
    }
}

```

### 空闲的连接和超时

连接管理。使得你的程序更加安全，高效和易用。

![image-20210322145505236](D:\学习笔记\image-20210322145505236.png)

IdleStateHandler连接空闲时间长，将会触发IdelStateEvent事件，可以通过inboundHandler重写userEventTriggered()来处理该事件

ReadTimeoutHandler，如果指定的时间没有任何的入站数据，则会抛出TimeoutException并关闭对应的Channel。

WriteTimeoutHandler，与上面同理。

```java
public class IdleStateHandlerInitializer extends ChannelInitializer<Channel> {
    @Override
    protected void initChannel(Channel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();
        pipeline.addLast(new IdleStateHandler(0,0,60, TimeUnit.SECONDS));
        pipeline.addLast(new HeartbeatHandler());
    }
    public static final class  HeartbeatHandler extends ChannelInboundHandlerAdapter {

        private static final ByteBuf HEARTBEAT_SEQUENCE = Unpooled.unreleasableBuffer(Unpooled.copiedBuffer("HEARTBEAT", CharsetUtil.ISO_8859_1));


        @Override
        public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
            if (evt instanceof IdleStateEvent){
                ChannelFuture future = ctx.writeAndFlush(HEARTBEAT_SEQUENCE);//意思就是如果写入完成后则关闭通过
                future.addListener(ChannelFutureListener.CLOSE);
            }else {
                super.userEventTriggered(ctx,evt);
            }
        }
    }

}
```

### 解码基于分隔符协议和基于长度的协议

#### 基于分隔符的协议

![image-20210322150808184](D:\学习笔记\image-20210322150808184.png)

DelimiterBasedFrameDecoder ：用户提供的分隔符来提取帧

LineBasedFrameDecoder ：使用\n或者\r\n分隔的帧的解码器

\n是换行，\r是回车 \n\r是回车加换行  enter键是回车加换行

![image-20210322150925295](D:\学习笔记\image-20210322150925295.png)

![image-20210322151301794](D:\学习笔记\image-20210322151301794.png)

```java
public class LineBasedHandlerInitializer extends ChannelInitializer<Channel> {
    @Override
    protected void initChannel(Channel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();
        pipeline.addLast(new LineBasedFrameDecoder(512 * 1024));
        pipeline.addLast(new FrameHandler());
    }
    public static final class FrameHandler extends SimpleChannelInboundHandler<ByteBuf> {
        /**
        * @Description: 底层会自动将数据根据 \n或者\r\n来提取帧，每次对该方法的一次调用都会传入一个帧
        * @Date: 2021/3/22
        */
        @Override
        protected void channelRead0(ChannelHandlerContext ctx, ByteBuf msg) throws Exception {

        }
    }

}
```

写一个符合下面的协议规范

- 传入数据是一系列的帧
- 每个帧都由一系列的元素组成，每个元素都由单个空格字符隔离
- 一个帧的内容代表一个命令，定义为一个命令名称后跟着有数目可变的参数。

我们用于这个协议的自定义解码器将定义以下类：

- Cmd—将帧的内容存储到ByteBuf中，一个ByteBuf用于名称，另外一个用于参数。

- CmdDecoder—从被重写了的decoder()方法中获取一行字符串，并将它的内容构建一个Cmd的实例。

- CmdHandler—从CmdDecoder获取解码的Cmd对象，并对它进行一些处理

- CmdHandlerInitializer—组装上面的Handler到pipeline中。

  

```java
/**
 * @program: TheFirstNettyApplication
 * @description:
 * @author: Mr.Feng
 * @create: 2021-03-22 15:25
 **/
public class CmdHandlerInitializer extends ChannelInitializer<Channel> {
    @Override
    protected void initChannel(Channel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();
        pipeline.addLast(new LineBasedFrameDecoder(512 * 1024));
        pipeline.addLast(new CmdDecoder());
        pipeline.addLast(new CmdHandler());
    }

    public static final class CmdDecoder extends ByteToMessageDecoder {

        private static final byte SEPARATOR = (byte) ' ';

        @Override
        protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
            ByteBuf buf = in.duplicate().retain();
            int separator = buf.indexOf(buf.readerIndex(), buf.writerIndex(), SEPARATOR);
            ByteBuf name = buf.slice(buf.readerIndex(), separator);
            ByteBuf content = buf.slice(separator + 1, buf.writerIndex());
            Cmd cmd = new Cmd(name, content);
            out.add(cmd);
        }
    }
    public static final class CmdHandler extends SimpleChannelInboundHandler<Cmd>{
        @Override
        protected void channelRead0(ChannelHandlerContext ctx, Cmd msg) throws Exception {
          //做一些事情。
        }
    }
    public static final class Cmd{
        private ByteBuf name;
        private ByteBuf content;

        public Cmd(ByteBuf name,ByteBuf content){
            this.name = name;
            this.content = content;
        }

        public ByteBuf getName() {
            return name;
        }

        public void setName(ByteBuf name) {
            this.name = name;
        }

        public ByteBuf getContent() {
            return content;
        }

        public void setContent(ByteBuf content) {
            this.content = content;
        }
    }

}
```

#### 基于长度的协议

![image-20210322154417970](D:\学习笔记\image-20210322154417970.png)

一个固定长度的，FixedLengthFrameDecoder。

一个可变长度的，LengthFieldBasedFrameDecoder，首先在帧的头部提取该帧的长度，然后获取指定的长度。

![image-20210322154524523](D:\学习笔记\image-20210322154524523.png)

![image-20210322154534308](D:\学习笔记\image-20210322154534308.png)

```java
/**
 * @program: TheFirstNettyApplication
 * @description:
 * @author: Mr.Feng
 * @create: 2021-03-22 15:46
 **/
public class LengthFieldInitializer extends ChannelInitializer<Channel> {
    @Override
    protected void initChannel(Channel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();
        pipeline.addLast(new LengthFieldBasedFrameDecoder(512 * 1024,0,8));
        pipeline.addLast(new LengthHandler());
    }
    public static final class LengthHandler extends SimpleChannelInboundHandler<ByteBuf>{

        @Override
        protected void channelRead0(ChannelHandlerContext ctx, ByteBuf msg) throws Exception {
            //根据每个8帧的长度来做一些事情。
        }
    }
}
```

### 写大型数据

由于是异步框架，并且可能出现网络延迟，拥堵。造成数据不能及时发送出去。

那么如果写许多大量的数据，那么可能会造成内存OOM溢出错误，Netty为解决这种情况提供了解决方案。

**首先可以使用NIO零拷贝，这种特性消除了将文件的内容从文件系统移动到网络栈的复制过程。**一切都在Netty的核心中发生，所以只需要使用一个FileRegion接口的实现。

下列代码展示了如何从FileInputStream创建一个DefaultFileRegion，并将其写入Channel，从而利用零拷贝特性来传输一个文件的内容。![image-20210322162648537](D:\学习笔记\image-20210322162648537.png)

FileRegion接口可以提供了零拷贝功能。

只适用于文件内容的直接传输，不包括应用程序对数据的任何处理。在需要将数据从文件系统复制到用户内存的时候，可以使用ChunkedWriteHandler    **Chunked 大块的**

**ChunkedWriteHandler支持异步写大型数据流，而又不会导致大量的内存消耗。为什么？**

![image-20210322164115729](D:\学习笔记\image-20210322164115729.png)

ChunkedWriteHandler处理的数据需要实现interface ChunkedInput<B>，其中类型参数B是readChunk()方法的返回类型。有默认4个该接口实现在Netty中

```java
public class ChunkedWriteHandlerInitializer extends ChannelInitializer<Channel> {

    private  final File file;
    private final SslContext context;

    public ChunkedWriteHandlerInitializer(File file,SslContext context){
        this.file = file;
        this.context = context;
    }
    @Override
    protected void initChannel(Channel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();
        pipeline.addLast(new SslHandler(context.newEngine(ch.alloc())));
        pipeline.addLast(new ChunkedWriteHandler());
        pipeline.addLast(new WriteStreamHandler());
    }
    public  final class WriteStreamHandler extends ChannelInboundHandlerAdapter{

        @Override
        public void channelActive(ChannelHandlerContext ctx) throws Exception {
            super.channelActive(ctx);
            ctx.writeAndFlush(new ChunkedStream(new FileInputStream(file))); //使用大块流写出，被WriteHandler进行拦截。
        }

    }
}
```

### 序列化大数据

JDK提供了ObjectOutputstream和ObjectInputStream来进行序列化。

但是性能不高效。

#### JDK序列化

如果你的应用程序必须要和使用了ObjectOutputstream和ObjectInputStream的远程结点进行交互，并且兼容性也是你最关心的，那么JDK序列化将是你的正确选择。

![image-20210322171328112](D:\学习笔记\image-20210322171328112.png)

对方是JDK，而我们是Netty，使用CompatibleObjectDecoder/Encoder

构建于JDK序列化之上的使用自定义的序列化来解码的解码器。

#### JBOSS Marshlling进行序列化

![image-20210322172842173](D:\学习笔记\image-20210322172842173.png)

```java
public class MarshallingInitializer extends ChannelInitializer<Channel> {
    private final UnmarshallerProvider unmarshallerProvider;
    private final MarshallerProvider provider;

    public MarshallingInitializer(UnmarshallerProvider unmarshallerProvider,MarshallerProvider provider){
        this.unmarshallerProvider =unmarshallerProvider;
        this.provider = provider;
    }
    @Override
    protected void initChannel(Channel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();
        pipeline.addLast(MarshallingInitializer.buildMarshallingDecoder());
        pipeline.addLast(MarshallingInitializer.builderMarshallingEncoder());
        pipeline.addLast(new ObjectHandler());
    }

    public static final class ObjectHandler extends SimpleChannelInboundHandler<Serializable> {

        @Override
        protected void channelRead0(ChannelHandlerContext ctx, Serializable msg) throws Exception {
            //Do something
        }
    }
    public static MarshallingDecoder buildMarshallingDecoder(){
       final MarshallerFactory factory = Marshalling.getProvidedMarshallerFactory("serial");
       final MarshallingConfiguration configuration = new MarshallingConfiguration();
        configuration.setVersion(5);
        UnmarshallerProvider provider = new DefaultUnmarshallerProvider(factory, configuration);
        MarshallingDecoder decoder = new MarshallingDecoder(provider);
        return decoder;
    }
    public static MarshallingEncoder builderMarshallingEncoder(){
        final MarshallerFactory factory = Marshalling.getProvidedMarshallerFactory("serial");
        final MarshallingConfiguration configuration = new MarshallingConfiguration();
        configuration.setVersion(5);
        MarshallerProvider provider = new DefaultMarshallerProvider(factory, configuration);
        MarshallingEncoder encoder = new MarshallingEncoder(provider);
        return encoder;
    }

}
```

#### 通过Protocol Buffers 序列化

ProtoBufVarint32FrameDecoder可以根据消息头的32位int分割出ByteBuf，解决半包问题。

ProtoBufVarint32LengthFieldPrepender 可以向ByteBuf里面追加一个32位整形的长度字段值，标识这个消息的长度。	

![image-20210322174558964](D:\学习笔记\image-20210322174558964.png)

```java

public class ProtoBufInitializer extends ChannelInitializer<Channel> {
    private final MessageLite messageLite;
    public ProtoBufInitializer(MessageLite messageLite){
        this.messageLite = messageLite;
    }
    @Override
    protected void initChannel(Channel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();

        pipeline.addLast(new ProtobufDecoder(messageLite));
        pipeline.addLast(new ProtobufEncoder());
        pipeline.addLast(new ObjectHandler());
    }
    public static final class ObjectHandler extends SimpleChannelInboundHandler<Object> {

        @Override
        protected void channelRead0(ChannelHandlerContext ctx, Object msg) throws Exception {
            //Do something
            
        }
    }
}

```

使用ProtoBuf实现真正的序列化和反序列化数据通信。

客户端：

```java
/**
 * @program: TheFirstNettyApplication
 * @description:
 * @author: Mr.Feng
 * @create: 2021-03-22 21:20
 **/
public class ProtoClientInitializer extends ChannelInitializer<SocketChannel> {
    @Override
    protected void initChannel(SocketChannel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();
        pipeline.addLast(new ProtobufVarint32FrameDecoder());
        pipeline.addLast(new ProtobufDecoder(DataInfoNetty.ResponseUser.getDefaultInstance()));
        pipeline.addLast(new ProtobufVarint32LengthFieldPrepender());
        pipeline.addLast(new ProtobufEncoder());
        pipeline.addLast(new ProtoClientHandler());
    }
}
```

```java
public class ProtoClientHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        DataInfoNetty.RequestUser build = DataInfoNetty.RequestUser.newBuilder().setAge(20).setPassword("123")
                .setUserName("奥力给").build();
        ctx.writeAndFlush(build);
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        DataInfoNetty.ResponseUser user = (DataInfoNetty.ResponseUser) msg;
        System.out.println(user.getMoney());
        System.out.println(user.getBankNo());
        System.out.println(user.getBankName());
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        cause.printStackTrace();
        ctx.channel().close();
    }
}

```

```java
public class ProtoClient {
    public static void main(String[] args) throws InterruptedException {
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        Bootstrap bootstrap = new Bootstrap();
        bootstrap.group(bossGroup)
                .channel(NioSocketChannel.class)
                .handler(new ProtoClientInitializer());

        ChannelFuture future = bootstrap.connect(new InetSocketAddress(8888)).sync();
        future.addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                if (future.isSuccess()){
                    System.out.println("连接成功");
                }else{
                    future.cause().printStackTrace();
                    future.channel().close();
                }
            }
        });
        future.channel().closeFuture().sync(); //主线程卡住，等待通道关闭才会继续向下运行，因为sync同步了，否则就是异步进行，当通道关闭的时候进行通知
        bossGroup.shutdownGracefully();
    }
}
```

服务端：

```java
public class ProtoServer {
    public static void main(String[] args) throws InterruptedException {
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        ServerBootstrap bootstrap = new ServerBootstrap();
        bootstrap.group(bossGroup,workerGroup)
                .channel(NioServerSocketChannel.class)
                .handler(new LoggingHandler(LogLevel.INFO))
                .childHandler(new ProtoServerInitializer());
        ChannelFuture sync = bootstrap.bind(8888).sync();
        sync.channel().closeFuture().sync(); //主线程卡住，等待通道关闭才会继续向下运行，因为sync同步了，否则就是异步进行，当通道关闭的时候进行通知
    }
}
```

```java
public class ProtoServerHandler extends SimpleChannelInboundHandler<DataInfoNetty.RequestUser> {
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, DataInfoNetty.RequestUser msg) throws Exception {
        System.out.println(msg.getUserName());
        System.out.println(msg.getAge());
        System.out.println(msg.getPassword());
        DataInfoNetty.ResponseUser bank = DataInfoNetty.ResponseUser.newBuilder().setBankName("中国工商银行")
                .setBankNo("6222222200000000000").setMoney(560000.23).build();
        ctx.writeAndFlush(bank);
    }
}
```

```java
public class ProtoServerInitializer extends ChannelInitializer<SocketChannel> {
    @Override
    protected void initChannel(SocketChannel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();
        pipeline.addLast(new ProtobufVarint32FrameDecoder());
        pipeline.addLast(new ProtobufDecoder(DataInfoNetty.RequestUser.getDefaultInstance()));
        pipeline.addLast(new ProtobufVarint32LengthFieldPrepender());
        pipeline.addLast(new ProtobufEncoder());
        pipeline.addLast(new ProtoServerHandler());
    }
}
```

ProtoBuf的*.proto文件以及对应的bash命令生成JAVA对象。

```bash
protoc --java_out=./ User.proto
```

```protobuf
syntax ="proto3";

package com.feng.netty.test;

option optimize_for = SPEED;
option java_package = "com.feng.netty.test";
option java_outer_classname = "DataInfoNetty";

message RequestUser{
   optional  string user_name = 1;
   optional  int32 age = 2;
   optional  string password = 3;
}
message ResponseUser{
   optional  string bank_no = 1;
   optional  double money = 2;
   optional  string bank_name = 3;
}
```

## WebSocket

实时Web使得用户在信息的的作者发布之后能够立刻收到信息，而不需要他们周期性的检查信息以获得更新。

### WebSocket简介

客户端和服务端之间可以在任意时刻传输消息，异步的处理消息回执。

### 我们的WebSocket示例程序

![image-20210323111543275](D:\学习笔记\image-20210323111543275.png)

### 添加WebSocket支持

从标准的Http/Https协议切换到WebSocket的时候，将会使用一种升级握手的机制。我们的应用程序采取以下约定：如果请求的URL 以/ws结尾，那么我们将该协议升级为WebSocket，否则，服务器使用基本的HTTPS

![image-20210323111851787](D:\学习笔记\image-20210323111851787.png)

#### 处理HTTP请求

URI是统一资源标识符，URL是统一资源定位符。

URL是URI的子集。

```java
/**
 * @program: TheFirstNettyApplication
 * @description:
 * @author: Mr.Feng
 * @create: 2021-03-23 11:27
 **/
public class HttpRequestHandler extends SimpleChannelInboundHandler<FullHttpRequest> {

    private final String wsUri;
    private static final File INDEX;

    static {
        URL location = HttpRequestHandler.class.getProtectionDomain().getCodeSource().getLocation();
        try {
            String path = location.toURI() + "index.html";
            path = !path.contains("file:") ? path : path.substring(5);
            INDEX = new File(path);

        } catch (URISyntaxException e) {
            throw new IllegalStateException(
                    "Unable to locate index.html", e);
        }
    }

    public HttpRequestHandler(String wsUri){
        this.wsUri = wsUri;
    }
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, FullHttpRequest request) throws Exception {
           if (wsUri.equalsIgnoreCase(request.getUri())){
               //之所以使用retain()，因为该方法结束后就会释放request信息，
               // 所以使用增加引用计数。
               ctx.fireChannelRead(request.retain());
           }else {
               if (HttpHeaders.is100ContinueExpected(request)) {
                   send100Continue(ctx);
               }
               RandomAccessFile file = new RandomAccessFile(INDEX,"r");
               HttpResponse response = new DefaultHttpResponse(request.getProtocolVersion(), HttpResponseStatus.OK);
               response.headers().set(HttpHeaders.Names.CONTENT_TYPE,"text/plain; charset=UTF-8");
               boolean keepAlive = HttpHeaders.isKeepAlive(request);
               if (keepAlive){
                   response.headers().set(HttpHeaders.Names.CONTENT_LENGTH, file.length());
                   response.headers().set( HttpHeaders.Names.CONNECTION,
                           HttpHeaders.Values.KEEP_ALIVE);
               }
               ctx.write(request);
               if (ctx.pipeline().get(SslHandler.class) == null){
                   //零拷贝传输 为什么get到SSL为空就使用零拷贝呢？  因为零拷贝不支持加密和压缩！！
                   ctx.write(new DefaultFileRegion(file.getChannel(),0,file.length()));

               }else {
                   //否则就使用大块异步写数据？ 如果使用加密和压缩那么只能使用写大块数据的API啦。
                   ctx.write(new ChunkedNioFile(file.getChannel()));
               }
               ChannelFuture future = ctx.writeAndFlush(LastHttpContent.EMPTY_LAST_CONTENT);
               if (!keepAlive){
                   future.addListener(ChannelFutureListener.CLOSE);
               }

           }

    }

    private void send100Continue(ChannelHandlerContext ctx) {
        FullHttpResponse response = new DefaultFullHttpResponse(
                HttpVersion.HTTP_1_1, HttpResponseStatus.CONTINUE);
        ctx.writeAndFlush(response);
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        cause.printStackTrace();
        ctx.close();
    }
}

```

#### 处理WebSocket帧

### ![image-20210323121146865](D:\学习笔记\image-20210323121146865.png)	

下面的聊天程序代码将使用

- CloseWebSocketFrame
- PingWebSocketFrame
- PongWebSocketFrame
- TextWebSocketFrame

TextWebSocketFrame是我们唯一需要真正处理的帧类型。

```java
public class TextWebSocketFrameHandler extends SimpleChannelInboundHandler<TextWebSocketFrame> {

    private final ChannelGroup group;

    public TextWebSocketFrameHandler(ChannelGroup group){
        this.group = group;
    }

    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
        if (evt == WebSocketServerProtocolHandler.ServerHandshakeStateEvent.HANDSHAKE_COMPLETE){
               //进入该逻辑标识WebSocket握手成功，然后将该ChannelHandler对应的管道中的HttpRequest移除，因为后续都是使用Websocket通话了。
                ctx.pipeline().remove(HttpRequestHandler.class);
                //向组里面的所有人通知客户端加入了
                group.writeAndFlush(new TextWebSocketFrame("Client" + ctx.channel() + "joined"));
                //将该通道加入组中。
                group.add(ctx.channel());
        }else {
            super.userEventTriggered(ctx,evt);
        }
    }

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, TextWebSocketFrame msg) throws Exception {
                  //也是为了增加引用计数，因为该操作是异步的，并且该方法结束会回收Msg，所以可能出现还没写完数据该数据已经被清空。
                   group.writeAndFlush(msg.retain());
    }
}
```

#### 初始化ChannelPipeHandler

![image-20210323123453188](D:\学习笔记\image-20210323123453188.png)

```java
public class ChatServerInitializer extends ChannelInitializer<Channel> {

    private final ChannelGroup group;
    public ChatServerInitializer(ChannelGroup group){
        this.group = group;
    }
    @Override
    protected void initChannel(Channel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();

        pipeline.addLast(new HttpServerCodec()); //用于解码Http响应和编码Http请求
        //写大块数据专用，不会造成内存OOM
        pipeline.addLast(new ChunkedWriteHandler());
        //将数据保证为FullHttpResponese，并且接受数据也是根据FullHttpRequest
        pipeline.addLast(new HttpObjectAggregator(512 * 1024 * 1024 * 1024));
        //websocket的专属请求
        pipeline.addLast(new HttpRequestHandler("/ws"));
        //该请求其他的ping,pong连接操作和关闭操作升级协议，都交给该Handler，Netty自带的
        pipeline.addLast(new WebSocketServerProtocolHandler("/ws"));
        //处理文本数据
        pipeline.addLast(new TextWebSocketFrameHandler(group));

    }
}
```

![image-20210323123749199](D:\学习笔记\image-20210323123749199.png)

WebSocket协议升级以后会移除所有Http相关的Handler和编解码器。

![image-20210323123820508](D:\学习笔记\image-20210323123820508.png)

#### 引导

```java
/**
 * @program: TheFirstNettyApplication
 * @description:
 * @author: Mr.Feng
 * @create: 2021-03-23 12:38
 **/
public class ChatServer {

    private final ChannelGroup group = new DefaultChannelGroup(ImmediateEventExecutor.INSTANCE);

    private final NioEventLoopGroup loopGroup = new NioEventLoopGroup();

    private Channel channel;

    public ChannelFuture start(InetSocketAddress address){
        ServerBootstrap bootstrap = new ServerBootstrap();
        bootstrap.group(loopGroup)
                .channel(NioServerSocketChannel.class)
                .childHandler(new ChatServerInitializer(group));
        ChannelFuture bind = bootstrap.bind(address);
        bind.syncUninterruptibly();
        channel = bind.channel();
        return bind;
    }
    public static void main(String[] args) {
        if (args.length != 1){
            System.out.println("Please give port");
            System.exit(1);
        }
        int port = Integer.parseInt(args[0]);
        final  ChatServer endpoint = new ChatServer();
        ChannelFuture future = endpoint.start(new InetSocketAddress(port));
        Runtime.getRuntime().addShutdownHook(new Thread(() -> endpoint.destory()));

        future.channel().closeFuture().syncUninterruptibly();

    }

    public void destory(){
        if (channel != null){
            channel.close();
        }
        group.close();
        loopGroup.shutdownGracefully();
    }


}

```

可以进行实施通信。

![image-20210323130906093](D:\学习笔记\image-20210323130906093.png)

如何进行在该基础上进行加密呢？

#### 如何进行加密

扩展ChatServerInitializer来创建一个SecureChatServerInitializer以完成我们的需求。

```java
public class SecureChatServerInitializer extends ChatServerInitializer {

    private final SslContext context;

    public SecureChatServerInitializer(ChannelGroup group,SslContext context) {
        super(group);
        this.context = context;
    }

    @Override
    protected void initChannel(Channel ch) throws Exception {
        ch.pipeline().addLast(new SslHandler(context.newEngine(ch.alloc())));
        super.initChannel(ch);
    }
}
```

```java
public class SecureChatServer extends ChatServer {

    public final SslContext context;

    public SecureChatServer(SslContext context){
        this.context = context;
    }

    @Override
    public ChannelInitializer<Channel> create(ChannelGroup group) {
        return new SecureChatServerInitializer(group,context);
    }

    public static void main(String[] args) throws Exception {
        if (args.length != 1) {
            System.err.println("Please give port as argument");
            System.exit(1);
        }
        int port = Integer.parseInt(args[0]);
        SelfSignedCertificate cert = new SelfSignedCertificate();
        SslContext sslContext = SslContext.newServerContext(cert.certificate(), cert.privateKey());
        final SecureChatServer endpoint = new SecureChatServer(sslContext);
        ChannelFuture future = endpoint.start(new InetSocketAddress(port));
        Runtime.getRuntime().addShutdownHook(new Thread() {
            @Override
            public void run() {
                endpoint.destory();
            }
        });
        future.channel().closeFuture().syncUninterruptibly();
    }
}
```

## UDP广播事件

一个UDP数据报都是一个单独的传输单元。

速度快，但是你的程序必须可以容忍消息丢失的情况，就是你不知道你发出去的数据能否能正确达到目标地址。

### UDP广播

面向连接的协议和无连接协议都支持点对点模式。

UDP提供了向多个接收者发送消息的额外传输模式。

- 多播：传输到一个预定义的主机组
- 广播：传输到网络上的所有主机。

### UDP示例应用程序

![image-20210323164918272](D:\学习笔记\image-20210323164918272.png)

我们将打开一个文件，把每一行都作为一个消息广播到一个指定的端口。

```java
public class LogEvent {
    private static final byte SEPARATOR = (byte) ':';
    private final InetSocketAddress source;
    //接受的文件名称
    private final String logfile;
    //接受的消息
    private final String msg;
    //接受的时间
    private final long receivedTime;

    public LogEvent(String logfile,String msg){
        this(null,-1,logfile,msg);
    }

    public LogEvent(InetSocketAddress source, int receivedTime, String logfile, String msg) {
            this.source = source;
            this.receivedTime = receivedTime;
            this.logfile = logfile;
            this.msg = msg;
    }

    public static byte getSEPARATOR() {
        return SEPARATOR;
    }

    public InetSocketAddress getSource() {
        return source;
    }

    public String getLogfile() {
        return logfile;
    }

    public String getMsg() {
        return msg;
    }

    public long getReceivedTime() {
        return receivedTime;
    }
}
```

### 编写广播者

![image-20210323165818140](D:\学习笔记\image-20210323165818140.png)

想要使用UDP，需要使用以上的消息容器和对应的Channel类型。

比如DatagramChannel和DatagramPacket。

![image-20210323170113180](D:\学习笔记\image-20210323170113180.png)

![image-20210323170126185](D:\学习笔记\image-20210323170126185.png)

将从文件取出一行数据，然后包装为LogEvent，然后再进行编码为DatagramPacket，将数据以UDP数据报的形式发送出去。广播到多个节点。

```java
public class LogEventBroadcaster {
    private final EventLoopGroup group;
    private final Bootstrap bootstrap;
    private final File file;

    public LogEventBroadcaster(InetSocketAddress address,File file){
        group = new NioEventLoopGroup();
        bootstrap = new Bootstrap();
        bootstrap.group(group)
                .channel(DatagramChannel.class)
                .option(ChannelOption.SO_BROADCAST, true)
                .handler(new LogEventEncoder(address));
        this.file = file;
    }
    public void run() throws Exception{
        Channel channel = bootstrap.bind(8888).sync().channel();
        long pointer = 0;
        for (;;){
            long length = file.length();
            if (length < pointer){
                length = pointer;
            }else if (length > pointer){
                RandomAccessFile randomAccessFile = new RandomAccessFile(this.file, "r");
                randomAccessFile.seek(pointer);
                String line;
                while ((line = randomAccessFile.readLine()) != null) {
                    channel.writeAndFlush(new LogEvent(null, -1,
                            file.getAbsolutePath(), line));
                }


                pointer = randomAccessFile.getFilePointer();
                randomAccessFile.close();
            }

            try {
                Thread.sleep(1000);
            }catch (Exception e){
                e.printStackTrace();
            }

        }


    }
    public void stop() {
        group.shutdownGracefully();
    }
    public static void main(String[] args) throws Exception {
        File file = new File("wudi.txt");
        File absoluteFile = file.getAbsoluteFile();

        System.out.println(absoluteFile);
        System.out.println(file.getName());
        LogEventBroadcaster broadcaster = new LogEventBroadcaster(
                new InetSocketAddress("255.255.255.255",
                        9999), new File("wudi.txt"));
        try {
            broadcaster.run();
        }
        finally {
            broadcaster.stop();
        }
    }
}
```

```java
public class LogEvent {
    public static final byte SEPARATOR = (byte) ':';
    private final InetSocketAddress source;
    private final String logfile;
    private final String msg;
    private final long receivedTime;

    public LogEvent(String logfile,String msg){
        this(null,-1,logfile,msg);
    }

    public LogEvent(InetSocketAddress source, int receivedTime, String logfile, String msg) {
            this.source = source;
            this.receivedTime = receivedTime;
            this.logfile = logfile;
            this.msg = msg;
    }

    public static byte getSEPARATOR() {
        return SEPARATOR;
    }

    public InetSocketAddress getSource() {
        return source;
    }

    public String getLogfile() {
        return logfile;
    }

    public String getMsg() {
        return msg;
    }

    public long getReceivedTime() {
        return receivedTime;
    }
}
```

```java
public class LogEventEncoder extends MessageToMessageEncoder<LogEvent> {
    private final InetSocketAddress remoteAddress;
    public LogEventEncoder(InetSocketAddress remoteAddress) {
        this.remoteAddress = remoteAddress;
    }
    @Override
    protected void encode(ChannelHandlerContext ctx, LogEvent msg, List<Object> out) throws Exception {
        byte[] bytes = msg.getLogfile().getBytes(CharsetUtil.UTF_8);
        byte[] bytes1 = msg.getMsg().getBytes(CharsetUtil.UTF_8);
        ByteBuf buffer = ctx.alloc().buffer(bytes.length + bytes1.length + 1);
        buffer.writeBytes(bytes);
        buffer.writeByte(LogEvent.SEPARATOR);
        buffer.writeBytes(bytes1);
        byte[] content = new byte[buffer.readableBytes()];
        buffer.readBytes(buffer,0,content.length);
        out.add(new DatagramPacket(content,content.length,remoteAddress));

    }
}

```

上面的步骤大致就是从文件读取每一行数据封装为一个LogEvent，然后将数据包装为DatagramPacket发到远程节点

下面代码的步骤大致就是将DatagramPacket的数据接受到然后解码为LogEvent，并且按照日志的格式打印输出。

```java
public class LogEventDecoder extends MessageToMessageDecoder<DatagramPacket> {
    @Override
    protected void decode(ChannelHandlerContext ctx,
                          DatagramPacket datagramPacket, List<Object> out) throws Exception {
        ByteBuf data = datagramPacket.content();
        int idx = data.indexOf(0, data.readableBytes(),
                LogEvent.SEPARATOR);
        String filename = data.slice(0, idx)
                .toString(CharsetUtil.UTF_8);
        String logMsg = data.slice(idx + 1,
                data.readableBytes()).toString(CharsetUtil.UTF_8);
        LogEvent event = new LogEvent(datagramPacket.sender(),
                System.currentTimeMillis(), filename, logMsg);
        out.add(event);
    }
}
```

```java
public class LogEventHandler
extends SimpleChannelInboundHandler<LogEvent> { 
@Override
public void exceptionCaught(ChannelHandlerContext ctx,
Throwable cause) throws Exception { 
cause.printStackTrace(); 
ctx.close();
 } 
@Override
public void channelRead0(ChannelHandlerContext ctx,
LogEvent event) throws Exception { 
StringBuilder builder = new StringBuilder(); 
builder.append(event.getReceivedTimestamp());
builder.append(" [");
builder.append(event.getSource().toString());
builder.append("] [");
builder.append(event.getLogfile());
builder.append("] : ");
builder.append(event.getMsg());
System.out.println(builder.toString()); 
 } 
}
```



![image-20210323214039944](D:\学习笔记\image-20210323214039944.png)

```java
public class LogEventDecoder extends MessageToMessageDecoder<DatagramPacket> {
    @Override
    protected void decode(ChannelHandlerContext ctx,
                          DatagramPacket datagramPacket, List<Object> out) throws Exception {
        ByteBuf data = Unpooled.buffer();
        int idx = data.indexOf(0, data.readableBytes(),
                LogEvent.SEPARATOR);
        String filename = data.slice(0, idx)
                .toString(CharsetUtil.UTF_8);
        String logMsg = data.slice(idx + 1,
                data.readableBytes()).toString(CharsetUtil.UTF_8);
        LogEvent event = new LogEvent(new InetSocketAddress(8888),
                System.currentTimeMillis(), filename, logMsg);
        out.add(event);
    }
}
```

```java
public class LogEventMonitor {
    private final EventLoopGroup group;
    private final Bootstrap bootstrap;
    public LogEventMonitor(InetSocketAddress address) {
        group = new NioEventLoopGroup();
                bootstrap = new Bootstrap();
        bootstrap.group(group)
                .channel(NioDatagramChannel.class)
                .option(ChannelOption.SO_BROADCAST, true)
                .handler( new ChannelInitializer<Channel>() {
                    @Override
                    protected void initChannel(Channel channel)
                            throws Exception {
                        io.netty.channel.ChannelPipeline pipeline = channel.pipeline();
                        pipeline.addLast(new LogEventDecoder());
                        pipeline.addLast(new LogEventHandler());
                    }
                } )
                .localAddress(address);
    }
    public Channel bind() {
        return bootstrap.bind().syncUninterruptibly().channel();
    }
    public void stop() {
        group.shutdownGracefully();
    }
    public static void main(String[] args) throws Exception {
        if (args.length != 1) {
            throw new IllegalArgumentException(
                    "Usage: LogEventMonitor <port>");
        }
        LogEventMonitor monitor = new LogEventMonitor(
                new InetSocketAddress(Integer.parseInt(args[0])));
        try {
            Channel channel = monitor.bind();
            System.out.println("LogEventMonitor running");
            channel.closeFuture().sync();
        } finally {

        }            monitor.stop();
        }
    }
```

![image-20210323233202172](D:\学习笔记\image-20210323233202172.png)

![image-20210323233311996](D:\学习笔记\image-20210323233311996.png)



