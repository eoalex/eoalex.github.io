---
title: 'Kafka入门之十二:Kafka的高性能之道'
tags:
  - Kafka
id: 1502
categories:
  - 云计算及虚拟化
  - 运维
date: 2016-12-30 10:03:24
---
## 1. 简介
在 LinkedIn的Kafka的系统上，每天有超过 8000 亿条消息被发送，相当于超过 175 兆兆字节（terabytes）数据，另外，每天还会消耗掉 650 兆兆字节（terabytes）数据的消息，为什么Kafka有这样的能力去处理这么多产生的数据和消耗掉的数据? 下面我们就来分析一下Kafka的高性能之道。
## 2. 高性能之道
### 2.1 高效使用磁盘
首先kafka的消息是不断追加到文件中的，因此数据只增加不更新。也没有记录级别的数据删除，只会整个segment删除。上述这个特性使kafka可以充分利用磁盘的顺序读写性能，顺序读写不需要硬盘磁头的寻道时间，只需很少的扇区旋转时间，所以速度远快于随机读写。另外kafka重度依赖底层操作系统提供的PageCache功能。当上层有写操作时，操作系统只是将数据写入PageCache，同时标记Page属性为Dirty。当读操作发生时，先从PageCache中查找，如果发生缺页才进行磁盘调度，最终返回需要的数据。
### 2.2 使用零拷贝
在Linux kernel2.2 之后出现了一种叫做"零拷贝(zero-copy)"系统调用机制，就是跳过“用户缓冲区”的拷贝，建立一个磁盘空间和内存的直接映射，数据不再复制到“用户态缓冲区”。传统模式下数据从文件传输到网络需要4次数据拷贝，4次上下文切换和2次系统调用，通过NIO的transferTo/transferFrom调用操作系统的sendfile实现了零拷贝。总共发生2次内核数据拷贝，2次上下文切换和1次系统调用，消除了CPU数据拷贝。这样系统上下文切换减少为2次，可以提升一倍的性能。
### 2.3 数据批处理和压缩
Producer和Consumer均支持批量处理数据，从而减少了网络传输的开销。比如每满100条消息才发送一次，或者每5秒发送一次。另外Producer可将数据压缩后发送给broker，从而减少网络传输代价。目前支持Snappy, Gzip和LZ4压缩。Producer压缩之后，在Consumer端需要解压，虽然增加了CPU的工作，但在对大数据处理上，瓶颈在网络上而不是CPU。
### 2.4 使用Partition技术
通过Partition实现了并行处理和水平扩展，Partition是Kafka(包括Kafka Stream)并行处理的最小单位，不同Partition可处于不同的Broker(节点)，可以充分利用多机资源。同一Broker(节点)上的不同Partition可置于不同的Directory，如果节点上 有多个Disk Drive，可将不同的Drive对应不同的Directory，从而使Kafka充分利用多Disk Drive的磁盘优势。
### 2.5 使用ISR
ISR实现了可用性和一致性的动态平衡。ISR可容忍更多的节点失败，ISR如果要容忍f个节点失败，至少只需要f+1个节点。一旦Leader crash后，ISR中的任何replica皆可竞选成为Leader，如果所有replica都crash，可选择让第一个recover的replica或者第一个在ISR中的replica成为leader。 
### 2.6 zerocopy实验
这里我们使用NIO的transferTo/transferFrom做文件数据传输性能测试，同时使用read／write方式做文件数据传输测试，并比较二者的差异。
* 使用FileChannel.transferFrom()
* 使用FileChannel.transferTo()
* 使用非直接模式ByteBuffer的read／write
* 使用直接模式 ByteBuffer的read／write
测试文件大小：600+ MB
![2016-12-24_10-44-45](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/12/2016-12-24_10-44-45.png)

测试的缓冲区大小：4KB
机器配置：MacBook Pro i7 2.2GHz  Mem 16GB  SSD 256 GB

测试结果如下.
![2016-12-24_10-53-32](http://orufryv17.bkt.clouddn.com/wp-content/uploads/2016/12/2016-12-24_10-53-32.png)
结论：
* transferFrom和transferTo 数据传输性能差不多。transferFrom性能稍优
* 使用直接模式和非直接模式 read/write 数据传输性能差不多。直接模式性能稍优
* transferFrom／To与read/write 性能高一倍以上。

代码如下：

```java
package zerocopy;

import java.io.IOException;  
import java.nio.ByteBuffer;  
import java.nio.channels.FileChannel;  
import java.nio.file.Files;  
import java.nio.file.Path;  
import java.nio.file.Paths;  
import java.nio.file.StandardOpenOption;  
import java.util.EnumSet;  

public class ZeroCopyTest {  

     private final static Path copy_from = Paths.get("/tmp/test/from/Security.mp4");  
     private final static Path copy_to = Paths.get("/tmp/test/to/Security.mp4");  
     private static long startTime, elapsedTime;  
     private static int bufferSizeKB = 4;
     private static int bufferSize = bufferSizeKB * 1024;  

    public static void main(String[] args) throws Exception {  

        transferfrom();            
        transferTo();            
        nonDirectBuffer();          
        directBuffer();        

    }  

    public static void transferfrom() {  

         try (FileChannel fileChannel_from = (FileChannel.open(copy_from,     
                              EnumSet.of(StandardOpenOption.READ)));  
              FileChannel fileChannel_to = (FileChannel.open(copy_to,    
                              EnumSet.of(StandardOpenOption.CREATE_NEW, StandardOpenOption.WRITE)))) {  

              startTime = System.currentTimeMillis();  
              fileChannel_to.transferFrom(fileChannel_from, 0L, (int) fileChannel_from.size());  
              elapsedTime = System.currentTimeMillis() - startTime;  
              System.out.println("transferFrom Time is " + elapsedTime + " ms");  
         }catch (IOException ex) {  
           System.err.println(ex);  
         }  
         deleteCopied(copy_to);      
    }  

    public static void transferTo() throws Exception{  

        try (FileChannel fileChannel_from = (FileChannel.open(copy_from,    
                              EnumSet.of(StandardOpenOption.READ)));  
              FileChannel fileChannel_to = (FileChannel.open(copy_to,    
                              EnumSet.of(StandardOpenOption.CREATE_NEW, StandardOpenOption.WRITE)))) {  

              startTime = System.currentTimeMillis();  

              fileChannel_from.transferTo(0L, fileChannel_from.size(), fileChannel_to);  

              elapsedTime = System.currentTimeMillis() - startTime;  
              System.out.println("transferTo Time is " + elapsedTime + " ms");  
        }catch (IOException ex) {  
           System.err.println(ex);  
        }  
        deleteCopied(copy_to);  

    }  

    public static void nonDirectBuffer(){  

         try (  
                FileChannel fileChannel_from = FileChannel.open(copy_from,    
                         EnumSet.of(StandardOpenOption.READ));  
                FileChannel fileChannel_to = FileChannel.open(copy_to,    
                         EnumSet.of(StandardOpenOption.CREATE_NEW, StandardOpenOption.WRITE));){   

              startTime = System.currentTimeMillis();  
              ByteBuffer bytebuffer = ByteBuffer.allocate(bufferSize);  
              while ((fileChannel_from.read(bytebuffer)) > 0) {  
               bytebuffer.flip();  
               fileChannel_to.write(bytebuffer);  
               bytebuffer.clear();  
              }  

              elapsedTime = System.currentTimeMillis() - startTime;  
              System.out.println("nonDirectBuffer read/write Time is " + elapsedTime  + " ms");  
         }catch (IOException ex) {  
              System.err.println(ex);  
         }  
         deleteCopied(copy_to);  
    }  

    public static void directBuffer(){  
         try (  
             FileChannel fileChannel_to = FileChannel.open(copy_to,    
                     EnumSet.of(StandardOpenOption.CREATE_NEW, StandardOpenOption.WRITE));  
             FileChannel fileChannel_from = (FileChannel.open(copy_from,    
                                 EnumSet.of(StandardOpenOption.READ)));) {  

         startTime = System.currentTimeMillis();  
         ByteBuffer bytebuffer = ByteBuffer.allocateDirect(bufferSize);  
         while ((fileChannel_from.read(bytebuffer)) > 0) {  
              bytebuffer.flip();  
              fileChannel_to.write(bytebuffer);  
              bytebuffer.clear();  
         }  

         elapsedTime = System.currentTimeMillis() - startTime;  
         System.out.println("directBuffer read/write Time is " + elapsedTime + " ms");  
        }catch (IOException ex) {  
        System.err.println(ex);  
        }  
        deleteCopied(copy_to);  
    }  

    public static void deleteCopied(Path path){  
          try {  
              Files.deleteIfExists(path);  
          }catch (IOException ex) {  
            System.err.println(ex);  
          }  
    }  
}
```

参考：[通过零拷贝实现有效数据传输](https://www.ibm.com/developerworks/cn/java/j-zerocopy/)