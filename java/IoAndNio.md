# IO和NIO

> 在以前，我们要进行不同机器上的网络通信，使用的是OIO，也就是ServerSocket和Socket那一套。
> 这套技术，会造成服务端每建立一个连接就需要开启一个线程处理，造成的后果就是线程资源浪费。对服务端来说，造成oom、cpu拉满等问题都是可能的。

```java
package cn.europa;

import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;
import java.net.ServerSocket;
import java.net.Socket;

public class Server {

    public static void main(String[] args) {
        Server server = new Server();
        server.start();
    }

    private void start() {
        try {

            ServerSocket serverSocket = new ServerSocket(9890);
            while (true) {
                Socket socket = serverSocket.accept();
                Thread thread = new Thread(new Task(socket));
                thread.start();
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private class Task implements Runnable {
        private final Socket socket;

        public Task(Socket socket) {
            this.socket = socket;
        }

        @Override
        public void run() {
            try {
                while (!Thread.interrupted()) {
                    InputStream inputStream = socket.getInputStream();
                    byte[] buffer = new byte[100];
                    int length = 100;
                    StringBuilder stb = new StringBuilder();
                    while (length == 100) {
                        length = inputStream.read(buffer);
                        stb.append(new String(buffer, 0, length));
                    }
                    System.out.println(Thread.currentThread().getName() + "线程读取--------------------------:\n" + stb.toString());
                    OutputStream outputStream = socket.getOutputStream();
                    outputStream.write("你好".getBytes());
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
}

```
```java
package cn.europa;

import java.io.IOException;
import java.io.OutputStream;
import java.net.InetSocketAddress;
import java.net.Socket;
import java.util.Scanner;

public class Client {

    public static void main(String[] args) {
        try {
            Socket socket = new Socket("127.0.0.1", 9890);
            while (!Thread.interrupted()) {
                OutputStream outputStream = socket.getOutputStream();
                Scanner scanner = new Scanner(System.in);
                System.out.println("请输入>>");
                while (scanner.hasNext()) {
                    System.out.println("请输入>>");
                    String next = scanner.next();
                    outputStream.write(next.getBytes());
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }

    }
}

```

## NIO的演进

> 在这里需要提前了解一个概念， Java里的nio并非单独是linux上的nio，其实他也支持multiplexing io模型，也就是多路复用。


### 1. 非阻塞式的IO模型

> linux里的非阻塞io模型，其实指的是在read、write这两大操作系统指令的时候，会非阻塞。比如如果没有数据进行操作，会直接返回空等。但是在Java nio中，也能看到类似的形式，但其实概念上是不一样的。Java nio在accept的时候，服务端可以一直循环并在没有accept到连接的时候处理其他事情。

```java
package cn.europa.nio;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;

public class NioServer {

    public static void main(String[] args) {
        try {
            ServerSocketChannel server = ServerSocketChannel.open();
            server.configureBlocking(false);
            server.bind(new InetSocketAddress(9999));

            while (true) {
                SocketChannel socketChannel = server.accept();

                if (socketChannel == null) {
                    System.out.println("----无建立连接，主线程干别的事情");
                    try {
                        Thread.sleep(2000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    continue;

                }
                new Thread(new Task(socketChannel)).start();
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private static class Task implements Runnable {

        private SocketChannel socketChannel;

        public Task(SocketChannel socketChannel) throws IOException {
            this.socketChannel = socketChannel;
            // 将此channel设置为非阻塞的。
//            socketChannel.configureBlocking(false);
        }

        @Override
        public void run() {
            while (!Thread.interrupted()) {
                ByteBuffer buffer = ByteBuffer.allocate(1024);
                try {
                    int length = 0;
                    StringBuilder stb = new StringBuilder();
                    while (length == 0) {
                        System.out.println(Thread.currentThread().getName() + "正在读取读取。");
                        length = socketChannel.read(buffer);
                        stb.append(new String(buffer.array()).trim());
                    }
                    System.out.println("线程" + Thread.currentThread().getName() + "读取数据-----------------------------" + stb.toString());
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
            if (socketChannel != null) {
                try {
                    socketChannel.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}

```
```java
package cn.europa.nio;

import java.io.IOException;
import java.io.OutputStream;
import java.net.InetSocketAddress;
import java.net.Socket;
import java.nio.ByteBuffer;
import java.nio.channels.SocketChannel;
import java.util.Scanner;

public class NioClient {

    public static void main(String[] args) {
        try {
            SocketChannel channel = SocketChannel.open(new InetSocketAddress("127.0.0.1", 9999));
            channel.configureBlocking(false);
            while (!Thread.interrupted()) {
                Scanner scanner = new Scanner(System.in);
                System.out.println("请输入>>");
                while (scanner.hasNext()) {
                    System.out.println("请输入>>");
                    String next = scanner.next();
                    channel.write(ByteBuffer.wrap(next.getBytes()));
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }

    }
}

```
### 2. NIO的三大组件

#### 2.1 Selector
> 出了上述之外，多路复用还有其他的强大点，这里需要了解到多路复用这个知识。也就是Linux里的select(早期)、epoll。mac os里的kqueue。在不同的操作系统里叫法不一样而已。他其实是一个内部不断循环的函数，将注册到selector上的事件循环到，并让你能够获取到当前事件。

```java
package cn.europa.reactor;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;
import java.util.Iterator;
import java.util.Set;

public class RServer {

    private static Selector selector;
    private static ServerSocketChannel serverChannel;

    public static void main(String[] args) {
        try {
            selector = Selector.open();
            serverChannel = ServerSocketChannel.open();

            serverChannel.configureBlocking(false);
            serverChannel.bind(new InetSocketAddress(9099));

            serverChannel.register(selector, SelectionKey.OP_ACCEPT);

            while (!Thread.interrupted()) {
                selector.select();
                Set<SelectionKey> keys = selector.selectedKeys();
                Iterator<SelectionKey> keyIt = keys.iterator();
                while (keyIt.hasNext()) {
                    SelectionKey key = keyIt.next();
                    if (key.isAcceptable()) {
                        ServerSocketChannel serverSocketChannel = (ServerSocketChannel) key.channel();
                        SocketChannel channel = serverSocketChannel.accept();
                        // 1、注册新io事件时一定要判断是否是null并且在null时重试。不然不会注册新事件。
                        if (channel == null) {
                            continue;
                        }
                        channel.configureBlocking(false);
                        channel.register(selector, SelectionKey.OP_READ);
                    }

                    if (key.isReadable()) {
                        new Thread(new IOHandler(key)).start();
                    }
                    // 2、需要remove掉这个key，nio中这个是累加的，不然下次会再次处理上一个io事件。
                    keyIt.remove();
                }
            }

        } catch (
                IOException e) {
            e.printStackTrace();
        }
    }

    private static class IOHandler implements Runnable {
        private SelectionKey key;

        public IOHandler(SelectionKey key) {
            this.key = key;
        }

        @Override
        public void run() {
            SocketChannel channel = (SocketChannel) key.channel();
            ByteBuffer buffer = ByteBuffer.allocate(100);
            int length = 0;
            StringBuilder stb = new StringBuilder();
            while (length == 0) {
                try {
                    length = channel.read(buffer);
                    stb.append(new String(buffer.array()).trim());
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
            System.out.println("线程" + Thread.currentThread().getName() + "正在处理-----------------------" + stb.toString());
        }
    }
}

```
#### 2.2 Buffer
> 不同于OIO中基于stream进行数据操作，NIO中会用缓冲区进行数据交换，服务端和客户端会基于这个缓冲区进行数据读取和写入。当然它可能比想象的复杂，包括需要了解翻转等概念。


#### 2.3 Channel

> 关于channel，在nio中，不同于socket、一个连接就是一个通道，在底层就是操作系统上的文件描述符的操作，对于连接，在服务器端会有监听连接的文件描述符，监听成千上万的客户端连接。对于传输数据，客户端和服务端都有一个传输文件描述符，建立起专项通道进行数据的传输。



### 3. NIO的问题

> NIO饱受诟病的几点

- 在你io事件在处理的时候，可能select又循环到了这个io事件，然后又处理了一次。
- 编程较复杂，在上述服务端代码的注视处均需注意，不然可能就踩坑了。
- 沾包、半包的问题。说起来可能比较难理解，看下客户端代码和运行结果。我们的想法是客户端没输入一次，就进行一次数据传输write。

```java
package cn.europa.reactor;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SocketChannel;
import java.util.Scanner;

public class RClient {

    public static void main(String[] args) {
        try {
            SocketChannel channel = SocketChannel.open(new InetSocketAddress("127.0.0.1", 9099));
            channel.configureBlocking(false);
            while (!Thread.interrupted()) {
                Scanner scanner = new Scanner(System.in);
                System.out.println("请输入>>");
                while (scanner.hasNext()) {
                    System.out.println("请输入>>");
                    String next = scanner.next();
                    ByteBuffer buffer = ByteBuffer.allocate(100);
                    for (int i = 0; i < 1000; i++) {
                        buffer.put(next.getBytes());
                        // 涉及到了buffer的翻转
                        buffer.flip();
                        channel.write(buffer);
                        buffer.clear();
                    }
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }

    }
}

```
![image.png](https://cdn.nlark.com/yuque/0/2021/png/454950/1637291276850-3fd1d2af-2de4-410c-bc46-b5d4b68cb8a1.png#clientId=u02139155-b9d7-4&from=paste&height=153&id=u095b5b34&name=image.png&originHeight=153&originWidth=506&originalType=binary&ratio=1&rotation=0&showTitle=false&size=7931&status=done&style=none&taskId=u519fc268-a882-44fb-a780-a9ffab8bc2d&title=&width=506)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/454950/1637291290276-de54c7ec-b0e1-407c-896f-13b123e863e4.png#clientId=u02139155-b9d7-4&from=paste&height=775&id=u0e52e730&name=image.png&originHeight=775&originWidth=1199&originalType=binary&ratio=1&rotation=0&showTitle=false&size=195839&status=done&style=none&taskId=u6dc1effc-be1e-4a14-8a9c-b9fca9c1124&title=&width=1199)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/454950/1637291303941-c495d6ed-5ba0-4ed6-b206-0267465fc002.png#clientId=u02139155-b9d7-4&from=paste&height=429&id=uee42866f&name=image.png&originHeight=429&originWidth=917&originalType=binary&ratio=1&rotation=0&showTitle=false&size=103642&status=done&style=none&taskId=u7defbf9c-3c96-4805-9924-f9c561aa401&title=&width=917)

> 为了更明显，我选择了重复的词语。
> 可以看到，明明只发了几个字，结果每次读取。要么读了好几次（沾包）,要么读到了个破碎的包（半包）。
> 这是因为nio是基于缓冲区的，客户端只负责往里写。而每次服务端来捞取固定长度字节的时候，可能客户端已经写了好多字节了，也可能还没写到100个字节。




### Netty

> netty的组件比较多

- channel
- pipeline
- bootstrap
- bytebuf
- handler

等等..
![image.png](https://cdn.nlark.com/yuque/0/2021/png/454950/1637373245271-74aa7ac8-a5ac-4994-a419-b5f0c7d91b11.png#clientId=u2df69292-7ff7-4&from=paste&height=1203&id=ub05656ea&name=image.png&originHeight=2406&originWidth=5648&originalType=binary&ratio=1&rotation=0&showTitle=false&size=3286053&status=done&style=none&taskId=uc6915c58-4505-47af-a171-d184cfd4381&title=&width=2824)

> 关于netty

