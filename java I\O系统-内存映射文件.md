__引言__

在前面的博文中关于RandomAccessFile类对超大文件的处理方式进行了学习，随机访问是独立的，支持读写的，而且它最大的特征是可以访问文件中任意一个位置。在本篇博文中，认识一个基于NIO的对于大文件的处理方式：内存映射文件。解释它的运作原理，同时给出对应demo。笔者目前整理的一些blog针对面试都是超高频出现的。大家可以点击链接：http://blog.csdn.net/u012403290

内核空间和用户空间
内核空间： 
在物理内存中，一般会割出一部分空间作为内核空间。这个空间在IO系统中主要的功能是对硬盘上的数据进行缓存，其目的是减少与硬盘的直接读写操作。为什么要这么操作呢？因为读取硬盘的速度与读取内存的速度不是一个级别的，我们尽量不去频繁的读写硬盘上的文件。比如说在硬盘上存在一个文件，我java需要进行读取，这个时候并不是直接从硬盘上把数据读取出来，而是先把数据读取到内核空间，然后其实java是从内核空间获取的数据。避免了过多的与硬盘的IO操作。 
用户空间： 
软件运行所占用的空间，比如说我们jvm运行所占用的就是用户空间。

硬盘，物理内存，运行内存
在解释内存映射之前，我们先解释一下对于磁盘，物理内存，运行内存的区别。硬盘：它具有一定的容量，比如说F盘就是一个磁盘；物理内存：我们可以称它为电脑的内存，其实就是内存条的容量；运行内存：主要是软件运行所占用的内存空间。软件运行结束会释放这部分空间。比如说java虚拟机运行时所占的内存就叫运行内存。 
所以其实运行内存是上面所说的用户空间。也属于物理内存的一部分。 
这里写图片描述
![io](/pic/内存分布.png 'io')
IO操作流程
一般来说，文件都是存储在硬盘中的，所以一般IO对文件的读取都是：①先把文件从硬盘中读取到内核空间；②从内核空间把文件读取到用户空间。在用户空间中，其实也是我们java虚拟机中，我们就可以对它进行操作。

内存映射
内存映射是是由一个文件到一块内存的映射，内存映射文件与虚拟内存有些类似，通过内存映射文件可以保留一个地址空间的区域，同时将物理存储器提交给此区域，只是内存文件映射的物理存储器来自一个已经存在于磁盘上的文件，而非系统的页文件（系统页文件就是指内核空间对文件的一次拷贝），而且在对该文件进行操作之前必须首先对文件进行映射，就如同将整个文件从磁盘加载到内存。

通俗来说就是这个操作不再经过系统内核进行中转，而是直接由用户空间与磁盘交互。但是需要说明的是映射与数据拷贝并不是一码事。映射是指把硬盘中的一部分数据与逻辑内存进行关联起来，后续的拷贝工作不是在映射这个步骤进行的。

在RandomAccessFile中知道，用随机访问的方式来处理文件的时候，我们可以用position指针指向我们要操作的文件部分。这样一来，就可以实现独立获取文件局部数据进行处理。但是两者的处理方式有很大的区别： 
①经典的IO操作，会先把数据读取到内核空间，再把文件读取到用户空间，进行两次文件拷贝。流程如下： 
这里写图片描述
![io](pic/传统IO实现图.png 'io')
②对于NIO来说，会直接产生一个内存映射，不经过内核读取操作： 
这里写图片描述
![io](pic/NIO内存映射实现图.png 'io')
需要说明的是，此时并没有拷贝数据到内存中去，而是当进程代码第一次引用这段代码内的用户地址时，触发了缺页异常，这时候系统会根据映射关系直接将文件的相关部分数据拷贝到进程的用户私有空间中去，这样就比传统的IO操作少了从内核空间拷贝到用户空间这样一步操作。

对于大多数操作系统而言，与通过普通的 read 和 write 方法读取或写入数千字节的数据相比，将文件映射到内存中开销更大。从性能的观点来看，通常将相对较大的文件映射到内存中才是值得的

MappedByteBuffer
在NIO中，第二个能直接与通道交互的缓冲器叫MappedByteBuffer，它是继承ByteBuffer而来，它具有ByteBuffer的所有方法。 
它主要是接收通道产生内存映射之后获得的文件。在NIO中，通道的map方法就是系统映射操作，具体请看下面这段通道(FileChannels)的map方法源码：
```java
    public abstract MappedByteBuffer map(MapMode mode,long position, long size)throws IOException;
```
这个方法主要是把硬盘的文件直接映射到内存中，它的MapMode分为3种模式：

1、PRIVATE：专用（ 写入时拷贝）映射模式 
2、READ_ONLY：只读映射模式，试图修改得到的缓冲区将导致抛出 ReadOnlyBufferException 
3、READ_WRITE：读/写映射模式

一般的，都采用读写模式。Position是一个指针，在Buffer类文章中的意义，表示要处理的其实位置。size表示要处理的大小，这个大小从position位置开始算起。

一个简单的内存映射文件demo
下面这个例子就是实现了内存映射文件来获取大文件的局部数据
```java
package com.brickworkers.io;

import java.io.IOException;
import java.io.RandomAccessFile;
import java.nio.MappedByteBuffer;
import java.nio.channels.FileChannel;

/**
 * 
 * @author Brickworker
 * Date:2017年5月19日下午4:49:43 
 * 关于类MappedTest.java的描述：内存映射文件测试
 * Copyright (c) 2017, brcikworker All Rights Reserved.
 */
public class MappedTest {

    public static void main(String[] args) throws IOException {
        //存在一个比较大的文件，是我自己复制黏贴的中间有一段是brickworker。
        //先建立一个通道，获取FileChannels
        FileChannel fc = new RandomAccessFile("F:/java/io/bw.txt", "rw").getChannel();
        //用mappedByteBuffer映射最后一段文字
        MappedByteBuffer mb = fc.map(FileChannel.MapMode.READ_ONLY, 1 , 11);
        while(mb.hasRemaining()){
            System.out.print((char)mb.get());
        }

    }

}
```
传统IO与内存映射文件的比较
在本例中，通过比较传统IO的文件处理方式与内存映射文件的处理方式的各自耗时，得出各自的性能情况。看看是否符合前面所描述的那样:
```java
package com.brickworkers.io;

import java.io.File;
import java.io.FileInputStream;
import java.io.FileNotFoundException;
import java.io.FileOutputStream;
import java.io.IOException;
import java.io.RandomAccessFile;
import java.nio.MappedByteBuffer;
import java.nio.channels.FileChannel;

import net.sf.cglib.asm.MethodAdapter;

/**
 * 
 * @author Brickworker
 * Date:2017年5月19日下午4:49:43 
 * 关于类MappedTest.java的描述：内存映射文件测试
 * Copyright (c) 2017, brcikworker All Rights Reserved.
 */
public class MappedTest {

    //拷贝目标路径
    final static String from ="F:/tools/YNote.exe";
    //拷贝结果路径
    final static  String to = "F:/java/io/YNote.exe";

    {
        //静态方法避免出现文件不存在
        File file = new File(to);
        if(!file.exists()){
            try {
                file.createNewFile();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    //用模板类创建一个模板方法
    private abstract static class Method{
        private String name;
        public Method(String name) {
            this.name = name;
        }
        public void run() throws IOException{
            long startTime = System.currentTimeMillis();
            test();
            System.out.println(this.name+"耗时:"+(System.currentTimeMillis() - startTime));
        }

        public abstract void test()throws IOException;
    }

    private static Method[] methods = {
            //用RandomAccessFile进行拷贝
            new Method("传统IO方式"){

                @Override
                public void test() throws IOException {
                    try {
                        //建立一个输出流
                        FileInputStream fis = new FileInputStream(new File(from));
                        //建立一个输出流
                        FileOutputStream fos = new FileOutputStream(new File(to));

                        byte [] buff = new byte [512]; 
                        //从一个流读取，写入另外一个流
                        int len = 0;
                        while((len = fis.read(buff)) != -1){
                            fos.write(buff, 0, len);
                        }
                        fis.close();
                        fos.close();
                    } catch (FileNotFoundException e) {
                        // TODO Auto-generated catch block
                        e.printStackTrace();
                    }

                }},

            new Method("内存映射方式"){

                @Override
                public void test() throws IOException {
                    try {
                        //创建输入输出通道
                        FileChannel fci = new FileInputStream(new File(from)).getChannel();
                        FileChannel fco = new FileOutputStream(new File(to)).getChannel();
                        //把对象映射到逻辑地址
                        MappedByteBuffer mb = fci.map(FileChannel.MapMode.READ_ONLY, 0, fci.size());
                        //把对象写出到目标通道
                        fco.write(mb);
                        fci.close();
                        fco.close();
                    } catch (Exception e) {
                        e.printStackTrace();
                    }

                }}
    };

    public static void main(String[] args) throws IOException {
        for(Method method : methods){
            method.run();
        }
    }

}
```
在上面的代码中，用静态代码块保证了输出的存在。同时用模板方法比较两者之间的运行效率。经过多次测试，本人的测试结果如下： 
这里写图片描述
![io](pic/IO_耗时对比.png 'io')
大家可以试一试，并且可以尝试把带有缓冲功能的传统IO，随机访问的RandomAccessFile都可以进行尝试。

希望对你有所帮助
