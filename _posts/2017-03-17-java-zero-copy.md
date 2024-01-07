---
layout: post
title: 通过zero copy优化系统之间的数据传输
date: 2017-03-17
categories: java
tags: [java]
description: 使用zero copy技术，减少系统在网络之间传输数据时不必要的数据拷贝和上下文切换
---

> 原文地址：https://www.ibm.com/developerworks/library/j-zerocopy/

### 通过zero copy技术实现高效的数据传输

在很多网页应用中，服务端有大量的静态内容，比如文件图片等，这些文件保存在磁盘中。在通过网络访问的时候，从磁盘中读取，然后原样写入到响应Socket中。在现代的计算机体系中，整个行为只需要少量的CPU参与，但是还是有一些低效，原读取因在于从磁盘到网络的过程：内核(操作系统内核)从磁盘中读取数据，并将数据通过用户空间缓冲区（kernel-user boundary）传输到应用中，应用再通过用户空间缓冲区将数据写入到Socket。实际上，应用服务器在整个从磁盘到socket的传输过程中只是一个低效的中间件，而实际上没有完成任何额外的工作。

>kernel-user boundary以及kernel mode和user mode的区别：[Understanding User and Kernel Mode](https://blog.codinghorror.com/understanding-user-and-kernel-mode/)

每次数据通过用户空间，都需要被复制，这个过程会消耗CPU和内存资源。幸运的是，你可以通过zero copy的技术来避免这些复制过程。应用使用zero copy使内核之间将数据从磁盘中拷贝到socket中。而不需要通过应用本省。zero copy极大提高了整个应用的性能，并减少从内核态到用户态的上下文切换次数。

Java使用`java.nio.channels.FileChannel`的`transferTo`和`transferFrom`来实现*NIX系统上的zero copy支持。你可以使用trasnferTo()方法将字节直接从调用方(fromChannel)传输到另一个作为写入目标的Channel(toChannel)，数据中间不需要再经过应用。下面以文件传输为例，分别使用传统方法和使用zero copy技术来展示两者之间的差异。

### 数据传输：使用传统方式

考虑这样的应用场景：通过网络在不同的程序之间传输数据（在Web服务器和FTP、Mail服务器这种场景是非常常见的）。这个场景的核心分成两个步骤：（1）读取文件；（2）通过Socket发送数据

去除异常处理和流关闭后发送方的代码如下：

````java
	Socket socket = null;
	DataOutputStream output = null;
	FileInputStream inputStream = null;
	socket = new Socket(server, port);
	String fname = "XXX.txt";
	inputStream = new FileInputStream(fname);
	output = new DataOutputStream(socket.getOutputStream());
	byte[] buffer = new byte[4096];
	long read = 0;
	while ((read = inputStream.read(buffer)) >= 0) {
		output.write(buffer);
	}
	//finally deal with io close.
````

可以概括为

````java
	File.read(fileDesc, buf, len);
	Socket.send(socket, buf, len);
````

整个流程非常简单，但是在程序和操作系统内部，却需要在用户态和内核态之间完成4次上下文切换，文件对应的数据则需要在整个过程中被拷贝4次。下图展示了整个文件从磁盘到socket过程中内部数据的移动。

Figure 1. 传统的数据拷贝途径

![figure-1](/assets/img/zerocopy/figure1.gif)

Figure 2. CPU上下文切换过程

![figure-2](/assets/img/zerocopy/figure2.gif)

具体步骤如下

1. `read()`方法调用导致上下文从用户态切换到内核态(Figure-2)。内部`sys_read()`（或者等价方法）被发送从文件中读取数据。直接内存读取引擎（[DMA](https://en.wikipedia.org/wiki/Direct_memory_access)）执行第一次拷贝(Figure-1)，将文件内容从磁盘中读取，并存储到内核地址空间(内核内存)缓存。
2. 请求的数据被从read buffer拷贝到application buffer，`read()`方法返回触发第二次上下文切换——从内核态切换到用户态，现在数据存储造用户地址空间（用户内存）缓存。
3. Socket的`send()`方法触发第三次上下文切换，从用户态到内核态；以及第三次数据拷贝，再次将数据从用户地址空间拷贝到内核地址空间缓存。此时，数据被放入一个和目标socket关联的内核地址空间缓存。
4. `send()`调用完成，出发第四次上下文切换，最后数据会从内核缓存中通过DMA引擎传输到socket协议引擎（网卡）

使用内核缓存作为中间层，而不是直接将数据放到用户缓存(application buffer)，看起来是较为低效的，但是内核缓存作为中间层被引入到处理流程中却是为了提高性能。在读取端使用中间缓存，当应用请求的数据量小于内核缓存容量的时候，内核缓存会作为“预读缓存(readable cache)”，这会显著改善性能表现；而在写入端，中间缓存允许写入完全异步化。（可以参考 [Page Cache](https://en.wikipedia.org/wiki/Page_cache)）

但是这种优化并不总会生效，甚至可能成为性能瓶颈。例如请求的数据量大于内核缓存大小，数据会多次在磁盘、内核缓存、用户缓存之间拷贝后才送达目的地。

zero copy就是通过减少无谓的数据拷贝来提高性能。

### 数据传输：zero copy模式

如果重新审视一下前面的例子，你会发现第二次和第三次数据拷贝，即内核缓存和用户缓存之间的数据拷贝不是必须的。应用除了将数据做一次缓存并传输到socket buffer并没有做其他的事情，相反数据可以直接从read buffer传输到socket buffer而不需要经过application buffer。使用`transferTo()`方法可以让你做到这一点。

````java
	public void transferTo(long position, long count, WritableByteChannel target);
````

`transferTo`方法将数据从文件通道(FileChannel)转移到指定的写入通道(WritableByteChannel）。在内部实现以来底层的操作系统对zero copy的支持；在大多数的*NIX系统，`transferTo()`会转换为`sendfile()`系统调用，它会将数据从一个文件描述转移到另一个文件描述（在C++中为指针）。

````cpp
	#include <sys/socket.h>
	ssize_t sendfile(int out_fd, int in_fd, off_t *offset, size_t count);
````

之前的`file.read()`和`socket.sned()`调用会被替换为单个的`transferTo()`调用

````java
    FileChannel fc = new FileInputStream(fname).getChannel();
	// transferTo(position, count, writableChannel);
    long curnset = fc.transferTo(0, fsize, sc);
````

Figure-3 展示了transferTo方法调用的数据拷贝过程

![figure-3](/assets/img/zerocopy/figure3.gif)

Figure-4 展示了transferTo方法调用的CPU上下文切换

![figure-4](/assets/img/zerocopy/figure4.gif)

具体步骤如下

1. `transferTo`方法使DMA引擎将文件内容拷贝到read buffer，然后数据在kernel内部拷贝到输出socket关联的缓存
2. 第三次拷贝是DMA引擎将数据从内核内存的socket buffer中拷贝到传输引擎(网卡)

在这一步，我们把上下文切换次数从4次减少到2次，数据拷贝次数从4次减少到3次（只有一次需要CPU参与），但是这还没有达到我们zero copy的目标。如果底层的网卡支持gather operations，我们可以进一步减少数据在内核中的复制。在Linux kernels 2.4以及之后的版本，socket buffer能够支持这个需求。使用这种方法，不仅可以减少上下文切换，还可以消除需要CPU参与的数据拷贝。在用户端代码不需要进行调整，但是内部实现已经发生了改变。

1. 调用`transferTo()`方法触发DMA引擎将文件内容拷贝到内核缓存中
2. 数据不会被拷贝到socket buffer，取而代之的是数据在内存缓存中的位置和长度描述符会被添加到socket buffer中。DMA引擎直接将数据从内核缓存中直接传输到协议引擎中。因此整个数据传输的过程都不需要CPU参与了

Figure-5 展示了在调用`transferTo`方法在支持gather opertaitons的环境下的表现

![figure-5](/assets/img/zerocopy/figure5.gif)

完整的代码如下（原文中代码存在一些小的问题）

````java
public class TraditionalClient {

    public static void main(String[] args) {
        int port = 2000;
        String server = "localhost";
        Socket socket = null;
        DataOutputStream output = null;
        FileInputStream inputStream = null;
        int ERROR = 1;
        // connect to server
        try {
            socket = new Socket(server, port);
            System.out.println("Connected with server " +
                    socket.getInetAddress() +
                    ":" + socket.getPort());
        } catch (IOException e) {
            e.printStackTrace();
            System.exit(ERROR);
        }

        try {
            String fname = "C:/cnarea20160320.sql";
            inputStream = new FileInputStream(fname);
            output = new DataOutputStream(socket.getOutputStream());
            long start = System.currentTimeMillis();
            byte[] b = new byte[4096];
            long read, total = 0;
            while ((read = inputStream.read(b)) >= 0) {
                total = total + read;
                output.write(b);
            }
            System.out.println("bytes send--" + total + " and totaltime--" + (System.currentTimeMillis() - start));
        } catch (IOException e) {
            e.printStackTrace();
        }
        try {
            assert output != null;
            output.close();
            socket.close();
            inputStream.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}

````

````java
public class TraditionalServer {

    public static void main(String args[]) {

        int port = 2000;
        ServerSocket server_socket;
        DataInputStream input;

        try {

            server_socket = new ServerSocket(port);
            System.out.println("Server waiting for client on port " +
                    server_socket.getLocalPort());

            while (true) {
                Socket socket = server_socket.accept();
                System.out.println("New connection accepted " +
                        socket.getInetAddress() +
                        ":" + socket.getPort());
                input = new DataInputStream(socket.getInputStream());
                try {
                    byte[] byteArray = new byte[4096];
                    while (true) {
                        int nread = input.read(byteArray, 0, 4096);
                        if (0 == nread)
                            break;
                    }
                } catch (IOException e) {
                    System.out.println(e);
                }
                try {
                    socket.close();
                    System.out.println("Connection closed by client");
                } catch (IOException e) {
                    System.out.println(e);
                }
            }
        } catch (IOException e) {
            System.out.println(e);
        }
    }
}
````

````java
public class TransferToClient {

    public static void main(String[] args) throws IOException, InterruptedException {
        TransferToClient sfc = new TransferToClient();
        sfc.testSendFile();
    }

    public void testSendFile() throws IOException, InterruptedException {
        String host = "localhost";
        int port = 9027;
        SocketAddress sad = new InetSocketAddress(host, port);
        SocketChannel sc = SocketChannel.open();
        sc.connect(sad);
        sc.configureBlocking(true);
        String fName = "C:/text.txt";
        File file = new File(fName);
        long fSize = file.length();
        FileChannel fc = new FileInputStream(fName).getChannel();
        long start = System.currentTimeMillis();
        long currentSet = 0;
        currentSet = fc.transferTo(0, fSize, sc);
        System.out.println("total bytes transferred--" + currentSet + " and time taken in MS--" + (System.currentTimeMillis() - start));
        sc.close();
    }

}
````

````java

public class TransferToServer {
    ServerSocketChannel listener = null;
    protected void mySetup() {
        InetSocketAddress listenAddr = new InetSocketAddress(9027);

        try {
            listener = ServerSocketChannel.open();
            ServerSocket ss = listener.socket();
            ss.setReuseAddress(true);
            ss.bind(listenAddr);
            System.out.println("Listening on port : " + listenAddr.toString());
        } catch (IOException e) {
            System.out.println("Failed to bind, is port : " + listenAddr.toString()
                    + " already in use ? Error Msg : " + e.getMessage());
            e.printStackTrace();
        }

    }

    public static void main(String[] args) {
        TransferToServer dns = new TransferToServer();
        dns.mySetup();
        dns.readData();
    }

    private void readData() {
        ByteBuffer dst = ByteBuffer.allocate(4096);
        try {
            while (true) {
                SocketChannel conn = listener.accept();
                System.out.println("Accepted : " + conn);
                conn.configureBlocking(true);
                int nread = 0;
                while (nread != -1) {
                    try {
                        nread = conn.read(dst);
                    } catch (IOException e) {
                        e.printStackTrace();
                        nread = -1;
                    }
                    dst.rewind();
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}

````

测试结果，使用zero copy大概会带来65%的效率提升（同等大小的文件下）。

文章的最后PrashanthKS提出需要排除NIO和OIO在网络连接上带来的影响因子，实际上在单个连接测试的时候，主要的OIO和NIO的开销是一致的的，由于`Selector`的对象和线程切换的原因，在单个连接的处理上，NIO可能相比OIO还会有一些性能上的损耗。在多并发的应用场景下，NIO因为其线程模型的提升才会带来优势。

### 总结

使用中间缓存层，可以最大化利用将数据集中之后利用批处理带来的效率提升，但是对于文件传输等问题上，使用缓冲只是一个无用的转发，对于整个传输过程甚至是起到了退化的作用。针对性的优化是去除多余的拷贝，通过这个过程也减少了上下文之间的切换操作。

> 对于系统上下文切换，可以看作是CPU工作效率得一个参考指标，过多得上下文切换可能会降低系统的运行效率，通过 vmstat 和 pidstat -w 可以进行响应的监控辅助排除问题。

现代计算机底层系统对于效率的优化和提升已经做了很多工作，了解和掌握这部分内容对于构建高性能的应用系统也至关重要。



