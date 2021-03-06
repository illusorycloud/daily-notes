# IO学习记录

## 常用方法

```java
package com.kct.api.test.Proxy;

import com.kct.api.test.Builder.MessageTwo;
import com.kct.api.test.Message;

import org.junit.jupiter.api.Test;
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

import java.io.File;
import java.io.IOException;
import java.io.RandomAccessFile;
import java.net.InetSocketAddress;
import java.net.ServerSocket;
import java.net.Socket;
import java.net.SocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.DatagramChannel;
import java.nio.channels.FileChannel;
import java.nio.channels.SelectableChannel;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;
import java.nio.file.FileSystem;
import java.nio.file.Files;
import java.nio.file.LinkOption;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.util.Iterator;
import java.util.Set;

import static java.nio.file.LinkOption.NOFOLLOW_LINKS;

public class NIO {
    @Test
    public void testBuffer1() {
        ByteBuffer buffer = ByteBuffer.allocate(128);
        buffer.clear();
        buffer.put("abbcdefg".getBytes());
        buffer.flip();
        System.out.println((char) buffer.get()); //a
        buffer.rewind();
        System.out.println((char) buffer.get()); //b
        buffer.mark();
        System.out.println((char) buffer.get()); //c
        buffer.reset();
        System.out.println((char) buffer.get()); //d
        System.out.println((char) buffer.get()); //e
        System.out.println((char) buffer.get()); //f
        System.out.println((char) buffer.get()); //g
        buffer.compact();
        buffer.put("hijklmn".getBytes());
    }

    @Test
    public void testChannel1() throws IOException {
        //创建随机读写文件
        RandomAccessFile file = new RandomAccessFile("D:\\test.txt", "rw");
        //获取通道
        FileChannel channel = file.getChannel();
//      channel.transferFrom(channel2,0,channel.size()); //把channel2数据传输到channel
//      channel.transferTo(0,channel.size(),channel2)    //把channel数据传输到channel2
        //创建一个用于读的buffer
        ByteBuffer readBuffer = ByteBuffer.allocate(128);
        //从channel中读取数据到buffer
        int read = channel.read(readBuffer);
        //切换到读模式
        readBuffer.flip();
        int capacity = readBuffer.capacity();
        int limit = readBuffer.limit();
        System.out.println("readBuffer capacity" + capacity);
        System.out.println("readBuffer limit" + limit);
        //从buffer中读取数据
        System.out.println((char) readBuffer.get());
        System.out.println((char) readBuffer.get());
        System.out.println((char) readBuffer.get());
        System.out.println((char) readBuffer.get());
        System.out.println((char) readBuffer.get());
        System.out.println((char) readBuffer.get());

        //创建一个写数据的buffer
        ByteBuffer writeBuffer = ByteBuffer.allocate(128);
        //清空 然后加入一些数据
        writeBuffer.clear();
        writeBuffer.put("abcdefg".getBytes());
        //从写模式切换到读模式
        writeBuffer.flip();
        // 使用循环 将buffer中的数据写入文件
        while (writeBuffer.hasRemaining()) {
            channel.write(writeBuffer);
        }
        file.close();
    }

    @Test
    public void testSocketChannel() {
        try {
            SocketChannel socketChannel = SocketChannel.open();
            socketChannel.connect(new InetSocketAddress("127.0.0.1", 33333));
            ByteBuffer writeBuffer = ByteBuffer.allocate(128);
            writeBuffer.put("this is message from client".getBytes());
            writeBuffer.flip();
            socketChannel.write(writeBuffer);
            ByteBuffer readBuffer = ByteBuffer.allocate(128);
            socketChannel.read(readBuffer);
            readBuffer.flip();
            StringBuffer sb = new StringBuffer();
            while (readBuffer.hasRemaining()) {
                sb.append((char) readBuffer.get());
            }
            System.out.println("from server:" + sb);
            socketChannel.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    @Test
    public void testServerSocketChannel() {
        try {
            ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
            //两者达到的效果是一样的，只需要使用其中之一即可。serverSocketChannel.socket().bind和serverSocketChannel.bind
            //其实，不使用ServerSocketChannel类，单纯的使用ServerSocket类和Socket类也能实现服务端和客户端通信的目的。
            // 但是主要注意的是不使用ServerSocketChannel类是无法实现IO的多路服用的。
            serverSocketChannel.socket().bind(new InetSocketAddress("127.0.0.1", 33333));
//            serverSocketChannel.bind(new InetSocketAddress("127.0.0.1", 33333));
            ServerSocket socket = serverSocketChannel.socket();
            SocketChannel socketChannel = serverSocketChannel.accept();
            ByteBuffer writeBuffer = ByteBuffer.allocate(128);
            writeBuffer.put("this is message from server".getBytes());
            writeBuffer.flip();
            socketChannel.write(writeBuffer);

            ByteBuffer readBuffer = ByteBuffer.allocate(128);
            socketChannel.read(readBuffer);
            readBuffer.flip();
            StringBuffer sb = new StringBuffer();
            while (readBuffer.hasRemaining()) {
                sb.append((char) readBuffer.get());
            }
            System.out.println("from client:" + sb);
            socketChannel.close();
            serverSocketChannel.close();

        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    //-------先打开服务端 在打开客户端 此时客服端能收到消息 但是客户端收不到 重启服务端后双方都能收到消息了
    @Test
    public void testUDPClient() {
        try {
            // 获取通道
            DatagramChannel datagramChannel = DatagramChannel.open();
            // 通过绑定来监听端口 这里不用写IP地址是因为只能接收发送到本机的消息
            // 这样 这个通道就可以获取到33332端口的消息了
            datagramChannel.bind(new InetSocketAddress(33332));
            ByteBuffer readBuffer = ByteBuffer.allocate(128);
            ByteBuffer writeBuffer = ByteBuffer.allocate(128);
            int i = 1;
            while (true) {

                writeBuffer.clear();
                writeBuffer.put("this is message from client".getBytes());
                writeBuffer.flip();
                // 这里发送消息 发送时需要指定IP和端口
                // 这里发送到本机的33333端口 服务端监听的是这个端口
                datagramChannel.send(writeBuffer, new InetSocketAddress("127.0.0.1", 33333));

                readBuffer.clear();
                // 这里接收消息 端口就是上面的33332
                datagramChannel.receive(readBuffer);
                readBuffer.flip();
                StringBuffer sb = new StringBuffer();
                while (readBuffer.hasRemaining()) {
                    sb.append((char) readBuffer.get());
                }
                System.out.println("from server: " + sb + " " + i);
                i++;

            }
//            datagramChannel.close();

        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    @Test
    public void testUDPServer() {
        try {
            DatagramChannel datagramChannel = DatagramChannel.open();
            //将通道设定为非阻塞的
            datagramChannel.configureBlocking(false);
            //绑定端口33333  可以接收发送到33333端口的数据 上面client就是发送到这个端口的
            datagramChannel.bind(new InetSocketAddress(33333));
//            Selector selector = Selector.open();
//            datagramChannel.register(selector, SelectionKey.OP_READ);
            ByteBuffer readBuffer = ByteBuffer.allocate(128);
            ByteBuffer writeBuffer = ByteBuffer.allocate(128);
//            datagramChannel.connect(new InetSocketAddress("127.0.0.1",33332));
            int i = 1;
            while (true) {
                writeBuffer.clear();
                writeBuffer.put("this is message from Server".getBytes());
                writeBuffer.flip();
                //发送数据到本机的33332端口 client监听的就是这个端口 所以可以收到数据
                datagramChannel.send(writeBuffer, new InetSocketAddress("127.0.0.1", 33332));

                //这里获取数据
                datagramChannel.receive(readBuffer);
                readBuffer.flip();
                StringBuffer sb = new StringBuffer();
                while (readBuffer.hasRemaining()) {
                    sb.append((char) readBuffer.get());
                }
                if (sb.length() != 0) {
                    System.out.println("from client: " + sb + " " + i);
                    i++;
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    @Test
    public void testSelector1() {
        try {
            ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
            ServerSocketChannel serverSocketChannel2 = ServerSocketChannel.open();
            Selector selector = Selector.open();
            serverSocketChannel.bind(new InetSocketAddress("127.0.0.1", 33333));
            SocketChannel socketChannel=SocketChannel.open();
            socketChannel.configureBlocking(false);
            //将通道设置为非阻塞的 这样才能注册到selector上
            serverSocketChannel.configureBlocking(false);
            serverSocketChannel2.configureBlocking(false);
            //将通道注册到selector上，第二个参数为关注的事件 关注多个时用|隔开
//            SelectionKey.OP_READ
//            SelectionKey.OP_CONNECT
//            SelectionKey.OP_WRITE
//            SelectionKey.OP_ACCEPT
            //一个SelectionKey键表示了一个特定的通道对象和一个特定的选择器对象之间的注册关系。
            SelectionKey selectionKey = socketChannel.register(selector, SelectionKey.OP_READ);
            while (true) {
                int select = selector.select();  //该selector上已经准备好的channel个数
                if (select == 0) {
                    return;
                }
                // 通过遍历selector上的key 来监听selector上的channel状态变化
                Set<SelectionKey> selectionKeys = selector.selectedKeys();
                Iterator<SelectionKey> iterator = selectionKeys.iterator();
                while (iterator.hasNext()) {
                    SelectionKey key = iterator.next();
                    if (key.isConnectable()) {  //channel连接了

                    } else if (key.isAcceptable()) {
                        ServerSocketChannel serverSocketChannel1 = (ServerSocketChannel) key.channel();
                        SocketChannel accept = serverSocketChannel1.accept();
                        accept.configureBlocking(false);
                        System.out.println("连接成功");
                    } else if (key.isReadable()) {

                    } else if (key.isValid()) {

                    } else if (key.isWritable()) {

                    }
                    iterator.remove();
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    @Test
    public void testPath() {
        // 根据文件路径获取Path
        Path path = Paths.get("D:\\test.ttx");
        //Path与file相互装换
        File file = path.toFile();
        Path path1 = file.toPath();

        System.out.println("getFileName: " + path.getFileName());
        System.out.println("getFileSystem: " + path.getFileSystem());
        System.out.println("getNameCount: " + path.getNameCount());
        System.out.println("getRoot: " + path.getRoot());
        System.out.println("getParent: " + path.getParent());

        Path path2 = Paths.get("."); //点 “.”表示获取当前路径
        System.out.println("getFileName: " + path2.toAbsolutePath());
        // normalize() : 返回一个路径，该路径是冗余名称元素的消除。
        // toRealPath() : 融合了toAbsolutePath()方法和normalize()方法
        System.out.println("normalize: " + path2.toAbsolutePath().normalize());
        try {
            System.out.println("toRealPath: " + path2.toRealPath());
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    @Test
    public void testFile() {
        Path path = Paths.get("D:\\test.ttx");

        boolean exists = Files.exists(path, NOFOLLOW_LINKS);
        System.out.println("文件是否存在：" + exists);
        try {
            Files.createDirectory(path);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    @Test
    public void testSpring(){
        ApplicationContext context=new ClassPathXmlApplicationContext("classpath:application.xml");
        MessageTwo bean = context.getBean(MessageTwo.class);
    }
}
```



## 1. 阻塞式IO

```java
package com.kct.api.test.Proxy.cglib;

import org.junit.jupiter.api.Test;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;
import java.nio.charset.StandardCharsets;

/**
 * 阻塞IO demo
 */
public class ServerClient {
    @Test
    public void Server() throws IOException {
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
        serverSocketChannel.socket().bind(new InetSocketAddress("127.0.0.1", 8080));

        while (true) {
            //会一直阻塞 知道有请求到来
            SocketChannel socketChannel = serverSocketChannel.accept();
            //每次请求来都开一个新的线程
            SocketHandler socketHandler = new SocketHandler(socketChannel);
            new Thread(socketHandler).start();
        }
    }


    class SocketHandler implements Runnable {
        private SocketChannel socketChannel;

        public SocketHandler(SocketChannel socketChannel) {
            this.socketChannel = socketChannel;
        }

        @Override
        public void run() {
            ByteBuffer readBuffer = ByteBuffer.allocate(1024);
            readBuffer.clear();
            try {

                int num;
                while ((num = socketChannel.read(readBuffer)) > 0) {
                    readBuffer.flip();
                    byte[] bytes = new byte[num];
                    // 取出buffer中的内容
                    readBuffer.get(bytes);
                    String result = new String(bytes, StandardCharsets.UTF_8);
                    System.out.println("收到请求： " + result);

                    ByteBuffer writeBuffer = ByteBuffer.allocate(1500);
                    writeBuffer.clear();
                    writeBuffer.put(("from client： 已经收到请求： " + result).getBytes());
                    writeBuffer.flip();
                    socketChannel.write(writeBuffer);
                    //每次读完清空readBuffer
                    readBuffer.clear();

                }
            } catch (IOException exceptione) {
                exceptione.printStackTrace();
            }

        }
    }


    @Test
    public void Client() throws IOException {
        SocketChannel socketChannel = SocketChannel.open();
        socketChannel.connect(new InetSocketAddress("127.0.0.1", 8080));
        //发送请求
        ByteBuffer writeBuffer = ByteBuffer.allocate(1024);
        writeBuffer.put("1234567890".getBytes());
        writeBuffer.flip();
        socketChannel.write(writeBuffer);

        ByteBuffer readBuffer = ByteBuffer.allocate(1024);
        int num;
        if ((num = socketChannel.read(readBuffer)) > 0) {
            // 切换到读模式
            readBuffer.flip();
            byte[] bytes = new byte[num];
            readBuffer.get(bytes);

            String result = new String(bytes, StandardCharsets.UTF_8);
            System.out.println("from server： " + result);
        }
        socketChannel.close();

    }


}

```

## 2. NIO

```java
package com.kct.api.test.Proxy.cglib;

import org.junit.jupiter.api.Test;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;
import java.nio.charset.StandardCharsets;
import java.util.Iterator;
import java.util.Set;

public class NIOServerClient {
    @Test
    public void NIOServer() throws IOException {
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
        serverSocketChannel.socket().bind(new InetSocketAddress("127.0.0.1", 8080));

        serverSocketChannel.configureBlocking(false);
        Selector selector = Selector.open();
        serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);

        while (true) {
            int select = selector.select();
            if (select <= 0) {
                continue;
            }
            //遍历
            Set<SelectionKey> selectionKeys = selector.selectedKeys();
            Iterator<SelectionKey> iterator = selectionKeys.iterator();
            while (iterator.hasNext()) {
                SelectionKey keys = iterator.next();
                iterator.remove();
                if (keys.isAcceptable()) {
                    //已经有新的连接到服务端了
                    SocketChannel socketChannel = serverSocketChannel.accept();
                    //有新连接不代表有数据了
                    //先注册到selector上，监听OP_READ
                    socketChannel.configureBlocking(false);
                    socketChannel.register(selector, SelectionKey.OP_READ);
                } else if (keys.isReadable()) {
                    //有数据可读了
                    SocketChannel channel = (SocketChannel) keys.channel();  //需要强转为SocketChannel
                    ByteBuffer readBuffer = ByteBuffer.allocate(1024);
                    int read = channel.read(readBuffer);
                    if (read > 0) {
                        //读到数据了

                        //读取到bytes数组并打印出来
                        byte[] bytes = new byte[read];
                        readBuffer.flip();
                        readBuffer.get(bytes);
                        String result = new String(bytes, StandardCharsets.UTF_8).trim();
                        System.out.println("from client: " + result);
                        //写回客户端
                        ByteBuffer writeBuffer = ByteBuffer.allocate(1024);
                        writeBuffer.put(bytes);
                        writeBuffer.flip();
                        channel.write(writeBuffer);
                    } else if (read == -1) {
                        //-1代表连接已经关闭了
                        channel.close();
                    }

                }

            }
        }
    }

    @Test
    public void NIOClient() throws IOException {
        SocketChannel socketChannel = SocketChannel.open();
        socketChannel.connect(new InetSocketAddress("127.0.0.1", 8080));
        //发送请求
        ByteBuffer writeBuffer = ByteBuffer.allocate(1024);
        writeBuffer.put("1234567890".getBytes());
        writeBuffer.flip();
        socketChannel.write(writeBuffer);

        ByteBuffer readBuffer = ByteBuffer.allocate(1024);
        int num;
        if ((num = socketChannel.read(readBuffer)) > 0) {
            // 切换到读模式
            readBuffer.flip();
            byte[] bytes = new byte[num];
            readBuffer.get(bytes);

            String result = new String(bytes, StandardCharsets.UTF_8);
            System.out.println("from server： " + result);
        }
        socketChannel.close();

    }
}
```

## 3. AIO

```java
package com.kct.api.test.Proxy.cglib;


import com.sun.xml.internal.ws.api.message.Attachment;

import org.apache.commons.lang3.CharSet;
import org.junit.jupiter.api.Test;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.net.SocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.AsynchronousFileChannel;
import java.nio.channels.AsynchronousServerSocketChannel;
import java.nio.channels.AsynchronousSocketChannel;
import java.nio.channels.CompletionHandler;
import java.nio.charset.StandardCharsets;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.Future;

import io.netty.channel.ChannelHandler;

public class AIOFile {
    @Test
    public void testAioFileChannel() throws IOException, ExecutionException, InterruptedException {
        Path path = Paths.get("D:\\test.txt");
        AsynchronousFileChannel fileChannel = AsynchronousFileChannel.open(path);
        // 一旦实例化完成，我们就可以着手准备将数据读入到 Buffer 中：
        ByteBuffer readBuffer = ByteBuffer.allocate(1024);
        readBuffer.clear();
        readBuffer.flip();
        //异步文件通道的读操作和写操作都需要提供一个文件的开始位置，文件开始位置为 0
        Future<Integer> future = fileChannel.read(readBuffer, 0);
        Integer integer = future.get();
        System.out.println(integer);
    }

    @Test
    public void testServerSocketChannel() throws IOException {
        AsynchronousServerSocketChannel socketChannel = AsynchronousServerSocketChannel.open();
        socketChannel.bind(new InetSocketAddress("127.0.0.1", 8080));
        Attachment attachment = new Attachment();
        attachment.setServer(socketChannel);
        socketChannel.accept(attachment, new CompletionHandler<AsynchronousSocketChannel, Attachment>() {
            @Override
            public void completed(AsynchronousSocketChannel result, Attachment attachment) {
                SocketAddress remoteAddress = null;
                try {
                    remoteAddress = result.getRemoteAddress();
                } catch (IOException e) {
                    e.printStackTrace();
                }
                System.out.println("收到新的连接： " + remoteAddress);

                attachment.getServer().accept(attachment, this);
                Attachment newAtt = new Attachment();
                newAtt.setServer(socketChannel);
                newAtt.setClient(result);
                newAtt.setReadMode(true);
                newAtt.setBuffer(ByteBuffer.allocate(2048));

                result.read(newAtt.getBuffer(), newAtt, new CompletionHandler<Integer, Attachment>() {
                    @Override
                    public void completed(Integer result, Attachment attachment) {
                        if (attachment.isReadMode()) {
                            //读取来自客户端的数据
                            ByteBuffer buffer = attachment.getBuffer();
                            buffer.flip();
                            byte[] bytes = new byte[buffer.limit()];
                            buffer.get(bytes);
                            String re = new String(bytes, StandardCharsets.UTF_8);
                            System.out.println("收到的数据： " + re);
                            //向客户端写数据 写数据到客户端也是异步
                            buffer.clear();
                            buffer.put("Response from Server".getBytes(StandardCharsets.UTF_8));
                            attachment.setReadMode(false);
                            buffer.flip();
                            attachment.getClient().write(buffer, attachment, this);
                        } else {
                            //到这里说明读数据将结束了 有两种选择
                            // 1. 继续等待数据
                            attachment.setReadMode(true);
                            attachment.getBuffer().clear();
                            attachment.getClient().read(attachment.getBuffer(), attachment, this);
//                            //2.断开连接
//                            try {
//                                attachment.getClient().close();
//                            } catch (IOException e) {
//                                e.printStackTrace();
//                            }

                        }
                    }

                    @Override
                    public void failed(Throwable exc, Attachment attachment) {
                        System.out.println("disconnect");

                    }
                });

            }

            @Override
            public void failed(Throwable exc, Attachment attachment) {
                System.out.println("connect failed");
            }
        });
        // 为了防止 main 线程退出
        try {
            Thread.currentThread().join();
        } catch (InterruptedException e) {
        }
    }

    class Attachment {
        private AsynchronousServerSocketChannel server;
        private AsynchronousSocketChannel client;
        private boolean isReadMode;
        private ByteBuffer buffer;
        // getter & setter

        public AsynchronousServerSocketChannel getServer() {
            return server;
        }

        public void setServer(AsynchronousServerSocketChannel server) {
            this.server = server;
        }

        public AsynchronousSocketChannel getClient() {
            return client;
        }

        public void setClient(AsynchronousSocketChannel client) {
            this.client = client;
        }

        public boolean isReadMode() {
            return isReadMode;
        }

        public void setReadMode(boolean readMode) {
            isReadMode = readMode;
        }

        public ByteBuffer getBuffer() {
            return buffer;
        }

        public void setBuffer(ByteBuffer buffer) {
            this.buffer = buffer;
        }
    }

    @Test
    public void ASocketChannelTest() throws IOException, ExecutionException, InterruptedException {
        AsynchronousSocketChannel socketChannel = AsynchronousSocketChannel.open();
        Future<Void> future = socketChannel.connect(new InetSocketAddress("127.0.0.1", 8080));
        //阻塞在这里 等待连接
        future.get();

        Attachment attachment = new Attachment();
        attachment.setClient(socketChannel);
        attachment.setBuffer(ByteBuffer.allocate(2048));
        attachment.getBuffer().put("hello server".getBytes());
        attachment.getBuffer().flip();

        socketChannel.write(attachment.getBuffer(), attachment, new CompletionHandler<Integer, Attachment>() {
            @Override
            public void completed(Integer result, Attachment attachment) {
                ByteBuffer buffer = attachment.getBuffer();
                if (attachment.isReadMode()) {
                    //读取服务端发送过来的数据
                    buffer.flip();
                    byte[] bytes = new byte[buffer.limit()];
                    buffer.get(bytes);
                    String res = new String(bytes, StandardCharsets.UTF_8);
                    System.out.println("from server: " + res);
                    //数据读取完后  接下来有两种选择
                    //1.继续向服务端发送新的数据
                    attachment.setReadMode(false);
                    attachment.getBuffer().clear();
                    attachment.getBuffer().put("new message from client".getBytes());
                    attachment.getBuffer().flip();
                    attachment.getClient().write(attachment.getBuffer(), attachment, this);
                    //  2.关闭连接
//                    try {
//                        attachment.getClient().close();
//                    } catch (IOException e) {
//                        e.printStackTrace();
//                    }

                } else {
                    //写操作完成后会进到这里
                    attachment.setReadMode(true);
                    buffer.clear();
                    attachment.getClient().read(attachment.getBuffer(), attachment, this);
                }
            }

            @Override
            public void failed(Throwable exc, Attachment attachment) {
                System.out.println("服务器无响应");
            }
        });


    }
}
```

## 4. Netty

```java
package com.kct.api.test.Proxy.cglib.netty;

import org.junit.jupiter.api.Test;

import java.net.InetSocketAddress;

import io.netty.bootstrap.Bootstrap;
import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ServerChannel;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.channel.socket.nio.NioSocketChannel;

public class test {
    private static final int PORT=33332;
    @Test
    public void test1() {
        NioEventLoopGroup group = new NioEventLoopGroup();
        try {
            ServerBootstrap bootstrap = new ServerBootstrap();
            bootstrap.group(group)
                    .channel(NioServerSocketChannel.class)
                    .localAddress(new InetSocketAddress(PORT))
                    .childHandler(new ChannelInitializer<ServerChannel>() {
                        @Override
                        protected void initChannel(ServerChannel channel) throws Exception {
                            System.out.println("connect...client: " + channel.remoteAddress());
                            channel.pipeline().addLast(new EchoServerHandler());
                        }
                    });
            // 服务器异步创建绑定
            ChannelFuture channelFuture = bootstrap.bind().sync();
            System.out.println(test.class + " started and listen on " + channelFuture.channel().localAddress());
            // 关闭服务器通道
            channelFuture.channel().closeFuture().sync();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            try {
                // 释放线程池资源
                group.shutdownGracefully().sync();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    @Test
    public void client(){
        NioEventLoopGroup group=new NioEventLoopGroup();
        try {
            Bootstrap bootstrap=new Bootstrap();
            bootstrap.group(group)
                    .channel(NioSocketChannel.class)
                    .remoteAddress(new InetSocketAddress("127.0.0.1",PORT))
                    .handler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel socketChannel) throws Exception {
                            System.out.println("connect...");
                            socketChannel.pipeline().addLast(new EchoClientHandler());
                        }
                    });
            System.out.println("created");
            //异步连接服务器
            ChannelFuture channelFuture = bootstrap.connect().sync();
            System.out.println("connect..");

            //异步等待关闭channel
            channelFuture.channel().closeFuture().sync();
            System.out.println("closed...");
        }catch (Exception e){
            e.printStackTrace();
        }finally {
            try {
                //释放线程资源
                group.shutdownGracefully().sync();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```




```
package com.kct.api.test.Proxy.cglib.netty;

import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelFutureListener;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;

public class EchoServerHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        System.out.println("server channelRead: " + msg);
        ctx.write(msg);
    }

    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
        System.out.println("server channelReadComplete ");
        // 第一种方法：写一个空的buf，并刷新写出区域。完成后关闭sock channel连接。
        ctx.writeAndFlush(Unpooled.EMPTY_BUFFER).addListener(ChannelFutureListener.CLOSE);
        //ctx.flush(); // 第二种方法：在client端关闭channel连接，这样的话，会触发两次channelReadComplete方法。
        //ctx.flush().close().sync(); // 第三种：改成这种写法也可以，但是这种写法，没有第一种方法的好。
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        System.out.println("server exceptionCaught: " + cause.getMessage());
        cause.printStackTrace();
        ctx.close();
    }
}
```

```
package com.kct.api.test.Proxy.cglib.netty;

import java.nio.charset.Charset;

import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.SimpleChannelInboundHandler;
import io.netty.util.CharsetUtil;

public class EchoClientHandler extends SimpleChannelInboundHandler<ByteBuf> {
    @Override
    protected void channelRead0(ChannelHandlerContext channelHandlerContext, ByteBuf o) throws Exception {
        System.out.println("client channelRead0");
        ByteBuf b = o.readBytes(o.readableBytes());
        System.out.println("client receive: " + b + "value is: " + b.toString(Charset.forName("utf-8")));
    }

    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        System.out.println("client channelActive");
        ctx.writeAndFlush(Unpooled.copiedBuffer("Netty rocks", CharsetUtil.UTF_8));
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        System.out.println("client exceptionCaught" + cause.getMessage());
        cause.printStackTrace();
        ctx.close();
    }
}
```

easychat

```
package com.kct.api.test.easychat;

import java.net.SocketAddress;

import io.netty.channel.Channel;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.SimpleChannelInboundHandler;
import io.netty.channel.group.ChannelGroup;
import io.netty.channel.group.DefaultChannelGroup;
import io.netty.util.concurrent.GlobalEventExecutor;

/**
 * SimpleChannelInboundHandler 实现了ChannelInboundHandlerAdapter并添加了一些功能
 */
public class SimpleChatServerHandler extends SimpleChannelInboundHandler<String> {
    /**
     * 把所有连上来的客户端channel都存在里面
     * 有人发送消息时循环遍历 给每个客户端都发一遍
     */
    public static ChannelGroup channels = new DefaultChannelGroup(GlobalEventExecutor.INSTANCE);

    /**
     * 有新客户端连接时回调
     *
     * @param ctx ChannelHandlerContext代表了ChannelPipeline和ChannelHandler之间的关联，是ChannelHandler之间信息传递的桥梁。
     * @throws Exception
     */
    @Override
    public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
        Channel channel = ctx.channel();
        SocketAddress socketAddress = channel.remoteAddress();
        channels.writeAndFlush("[SERVER] - " + socketAddress + " 进入聊天室\n");
        channels.add(channel);
    }

    /**
     * 客户端断开连接时回调
     *
     * @param ctx 同上
     * @throws Exception
     */
    @Override
    public void handlerRemoved(ChannelHandlerContext ctx) throws Exception {
        Channel channel = ctx.channel();
        SocketAddress socketAddress = channel.remoteAddress();
        channels.writeAndFlush("[SERVER] - " + socketAddress + " 退出聊天室\n");
        channels.remove(channel);
    }

    /**
     * 服务端读到客户端写入信息时回调
     * 将消息转发给其他客户端
     *
     * @param channelHandlerContext 同上
     * @param o 收到的消息
     * @throws Exception
     */
    @Override
    protected void channelRead0(ChannelHandlerContext channelHandlerContext, String o) throws Exception {
        Channel channel = channelHandlerContext.channel();
        SocketAddress socketAddress = channel.remoteAddress();
        for (Channel c : channels  //循环遍历转发消息给所有的客户端
        ) {
            if (c == channel) { //如果是自己发的消息就显示你说了什么什么
                c.writeAndFlush("you：" + o + "\n");
            } else { //如果不是就显示 XXX(地址)说了什么
                c.writeAndFlush(socketAddress + ": " + o + "\n");
            }
        }
    }

    /**
     * 服务端监听到客户端-活动-时回调
     *
     * @param ctx 同上
     * @throws Exception
     */
    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        Channel channel = ctx.channel();
        SocketAddress socketAddress = channel.remoteAddress();
        System.out.println(socketAddress + "在线\n");

    }

    /**
     * 服务端监听到客户端-不活动-时回调
     *
     * @param ctx 同上
     * @throws Exception
     */
    @Override
    public void channelInactive(ChannelHandlerContext ctx) throws Exception {
        Channel channel = ctx.channel();
        SocketAddress socketAddress = channel.remoteAddress();
        System.out.println(socketAddress + "离线\n");
    }

    /**
     * 出现异常时回调
     *
     * @param ctx 同上
     * @param cause 异常信息
     * @throws Exception
     */
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        Channel channel = ctx.channel();
        SocketAddress socketAddress = channel.remoteAddress();
        System.out.println(socketAddress + "异常\n");
        ctx.close();
        cause.printStackTrace();
    }
}
```







## charserver

```
package com.example.demo.netty;

import java.net.InetSocketAddress;
import java.security.cert.CertificateException;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.Channel;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.EventLoop;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.group.ChannelGroup;
import io.netty.channel.group.DefaultChannelGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.handler.ssl.SslContext;
import io.netty.handler.ssl.util.SelfSignedCertificate;
import io.netty.util.concurrent.ImmediateEventExecutor;

/**
 * @author Administrator
 */
public class ChatServer {
    private final ChannelGroup channels = new DefaultChannelGroup(ImmediateEventExecutor.INSTANCE);
    private final EventLoopGroup group = new NioEventLoopGroup();
    private Channel channel;

    public ChannelFuture start(InetSocketAddress address) {
        ServerBootstrap b = new ServerBootstrap();
        b.group(group)
                .channel(NioServerSocketChannel.class)
                .childHandler(createInitializer(channels));
        ChannelFuture future = b.bind(address);
        future.syncUninterruptibly();
        channel = future.channel();
        return future;
    }

    protected ChannelInitializer<Channel> createInitializer(ChannelGroup group) {        //3
        return new ChatServerInitializer(group);
    }

    SelfSignedCertificate selfSignedCertificate;

    private SecureChatServerIntializer creatSecureChatServerIntializer(ChannelGroup group) {
        try {
            selfSignedCertificate = new SelfSignedCertificate();
            return new SecureChatServerIntializer(channels, SslContext.newServerContext(selfSignedCertificate.certificate(), selfSignedCertificate.privateKey()));
        } catch (Exception e) {
            e.printStackTrace();
            return null;
        }
    }

    public void destroy() {        //4
        if (channel != null) {
            channel.close();
        }
        channels.close();
        group.shutdownGracefully();
    }

    public static void main(String[] args) throws Exception {

        int port = 8080;

        final ChatServer endpoint = new ChatServer();
        ChannelFuture future = endpoint.start(new InetSocketAddress(port));

        Runtime.getRuntime().addShutdownHook(new Thread() {
            @Override
            public void run() {
                endpoint.destroy();
            }
        });
        future.channel().closeFuture().syncUninterruptibly();
    }
}

```

```
package com.example.demo.netty;

import io.netty.channel.Channel;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelPipeline;
import io.netty.channel.group.ChannelGroup;
import io.netty.handler.codec.http.HttpObjectAggregator;
import io.netty.handler.codec.http.HttpServerCodec;
import io.netty.handler.codec.http.websocketx.WebSocketServerProtocolHandler;
import io.netty.handler.stream.ChunkedWriteHandler;

public class ChatServerInitializer extends ChannelInitializer {
    private ChannelGroup group;

    public ChatServerInitializer(ChannelGroup group) {
        this.group = group;
    }

    @Override
    protected void initChannel(Channel channel) throws Exception {
        ChannelPipeline pipeline = channel.pipeline();
        pipeline.addLast(new HttpServerCodec());
        pipeline.addLast(new HttpObjectAggregator(64 * 1024));
        pipeline.addLast(new ChunkedWriteHandler());
        pipeline.addLast(new HttpRequestHandler("/ws"));
        pipeline.addLast(new WebSocketServerProtocolHandler("/ws"));
        pipeline.addLast(new TextWebSocketFrameHandler(group));
    }
}

```

```
package com.example.demo.netty;

import java.util.List;

import io.netty.buffer.ByteBuf;
import io.netty.channel.ChannelHandlerContext;
import io.netty.handler.codec.ByteToMessageDecoder;

public class FixedLengthFrameDecoder extends ByteToMessageDecoder {
    private final int len;

    public FixedLengthFrameDecoder(int len) {
        this.len = len;
    }

    @Override
    protected void decode(ChannelHandlerContext channelHandlerContext, ByteBuf byteBuf, List<Object> list) throws Exception {
        if (byteBuf.readableBytes() >= len) {
            ByteBuf byteBuf1 = byteBuf.readBytes(len);
            list.add(byteBuf);
        }
    }
}

```

```
package com.example.demo.netty;

import java.io.File;
import java.io.RandomAccessFile;
import java.net.URI;
import java.net.URL;

import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelFutureListener;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.DefaultFileRegion;
import io.netty.channel.SimpleChannelInboundHandler;
import io.netty.handler.codec.http.DefaultFullHttpResponse;
import io.netty.handler.codec.http.FullHttpRequest;
import io.netty.handler.codec.http.FullHttpResponse;
import io.netty.handler.codec.http.HttpHeaderNames;
import io.netty.handler.codec.http.HttpHeaderValues;
import io.netty.handler.codec.http.HttpHeaders;
import io.netty.handler.codec.http.HttpResponseStatus;
import io.netty.handler.codec.http.HttpUtil;
import io.netty.handler.codec.http.HttpVersion;
import io.netty.handler.codec.http.LastHttpContent;
import io.netty.handler.ssl.SslHandler;
import io.netty.handler.stream.ChunkedNioFile;

public class HttpRequestHandler extends SimpleChannelInboundHandler<FullHttpRequest> {
    private final String wsUri;
    private static File INDEX = null;

    public HttpRequestHandler(String wsUri) {
        this.wsUri = wsUri;
    }

    static {
        URL location = HttpRequestHandler.class.getProtectionDomain().getCodeSource().getLocation();
        try {
            URI uri = location.toURI();
            String path = uri + "index.html";
            path = !path.contains("file:") ? path : path.substring(5);
            INDEX = new File(path);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }


    @Override
    protected void channelRead0(ChannelHandlerContext channelHandlerContext, FullHttpRequest fullHttpRequest) throws Exception {
        if (wsUri.equalsIgnoreCase(fullHttpRequest.uri())) {
            channelHandlerContext.fireChannelRead(fullHttpRequest.retain());
        } else {
            if (HttpUtil.is100ContinueExpected(fullHttpRequest)) {
                send100Continue(channelHandlerContext);
            }

            RandomAccessFile r = new RandomAccessFile(INDEX, "r");
            DefaultFullHttpResponse response = new DefaultFullHttpResponse(fullHttpRequest.protocolVersion(), HttpResponseStatus.OK);
            //设置响应头
            response.headers().set(HttpHeaderNames.CONTENT_TYPE, "text/html;charset=UTF-8");
            boolean keepAlive = HttpUtil.isKeepAlive(fullHttpRequest);
            if (keepAlive) {
                response.headers().set(HttpHeaderNames.CONTENT_LENGTH, r.length());
                response.headers().set(HttpHeaderNames.CONNECTION, HttpHeaderValues.KEEP_ALIVE);
            }
            channelHandlerContext.write(response);
            //判断是否有ssl 没有就使用DefaultFileRegion 否则ChunkedNioFile
            if (channelHandlerContext.pipeline().get(SslHandler.class) == null) {
                channelHandlerContext.write(new DefaultFileRegion(r.getChannel(), 0, r.length()));
            } else {
                channelHandlerContext.write(new ChunkedNioFile(r.getChannel()));
            }
            ChannelFuture channelFuture = channelHandlerContext.writeAndFlush(LastHttpContent.EMPTY_LAST_CONTENT);
            if (!keepAlive) {
                channelFuture.addListener(ChannelFutureListener.CLOSE);
            }

        }
    }

    private static void send100Continue(ChannelHandlerContext ctx) {
        FullHttpResponse response = new DefaultFullHttpResponse(HttpVersion.HTTP_1_1, HttpResponseStatus.CONTINUE);
        ctx.writeAndFlush(response);
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        cause.printStackTrace();
        ctx.close();
    }
}

```

```
package com.example.demo.netty;

import javax.net.ssl.SSLEngine;

import io.netty.channel.Channel;
import io.netty.channel.group.ChannelGroup;
import io.netty.handler.ssl.SslContext;
import io.netty.handler.ssl.SslHandler;

public class SecureChatServerIntializer extends ChatServerInitializer {
    private final SslContext sslContext;

    public SecureChatServerIntializer(ChannelGroup group, SslContext sslContext) {
        super(group);
        this.sslContext = sslContext;
    }

    @Override
    protected void initChannel(Channel channel) throws Exception {
        super.initChannel(channel);
        SSLEngine sslEngine = sslContext.newEngine(channel.alloc());
        sslEngine.setUseClientMode(false);
        channel.pipeline().addLast(new SslHandler(sslEngine));
    }
}

```

```
package com.example.demo.netty;

import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.SimpleChannelInboundHandler;
import io.netty.channel.group.ChannelGroup;
import io.netty.handler.codec.http.websocketx.TextWebSocketFrame;
import io.netty.handler.codec.http.websocketx.WebSocketServerProtocolHandler;

public class TextWebSocketFrameHandler extends SimpleChannelInboundHandler<TextWebSocketFrame> {
    private ChannelGroup group;

    public TextWebSocketFrameHandler(ChannelGroup group) {
        this.group = group;
    }

    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
        if (evt == WebSocketServerProtocolHandler.ServerHandshakeStateEvent.HANDSHAKE_COMPLETE) {
            ctx.pipeline().remove(HttpRequestHandler.class);
            group.writeAndFlush("Client " + ctx.channel() + " joined");
            group.add(ctx.channel());
        } else {
            super.userEventTriggered(ctx, evt);
        }
    }

    @Override
    protected void channelRead0(ChannelHandlerContext channelHandlerContext, TextWebSocketFrame msg) throws Exception {
        group.writeAndFlush(msg.retain());
    }
}

```