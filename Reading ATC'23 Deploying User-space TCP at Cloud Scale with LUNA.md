# Reading: ATC'23 Deploying User-space TCP at Cloud Scale with LUNA

## motivation

想做一个 user-space TCP，因为在云场景下 kernel tcp 不够好。

现有的解决方案也不够好：

- RDMA：`NSDI’21, When Cloud Storage Meets RDMA`
  
  - 应用于传统设备，连接过多，网络不稳定

- 其他 user-space TCP：`NSDI’14, mTCP: A Highly Scalable User-level TCP Stack for Multicore Systems` `OSDI’14, IX: A Protected Dataplane Operating System for High Throughput and Low Latency`
  
  - 性能不够，兼容性差
  
  - 在文章后面零碎提了 mTCP 需要拷贝，延迟高；IX 虽然有零拷贝，但需要特定 linux version 和特定网卡，而且事实上需要占用整个网卡

> 对于RDMA，它说RDMA只方便在单个data center内，但多个data center不稳定且难部署。这是一个比较新的提法，暗示说 RDMA 还不够 remote。
> 
> 对于其他同类工作而言，其实提的issue不算特别强，都是兼容问题。作者在 3.4 特别提了之前文章的 Implementation quality问题，指出 IX 只支持特定网卡，VPP 在某些网卡上不支持  flow director filters。但毕竟是10年前的工作，总会有兼容跟性能问题的

## 特点

##### (1) 线程模型使用run-to-completion

提供了两种模型：

- batch-r2c：事件驱动，类似 mTCP 和 IX，利用 epoll

- inline-r2c：直接等输入队列满了，找个线程一次性拿一堆包去处理

> 处理网络包这块有两种思路，pipeline 和 r2c(run-to-completion)。以前有把r2c改进成 pipeline 的，就把解析的每个阶段分给不同核/不同硬件处理，提高效率。但后来也有 argue 说做出 pipeline 不方便扩展、有核间一致性问题、需要根据不同设备微调等等，所以做 r2c 的又变多了。
> 
> 5.2 部分翻译：
> 
> 显然，inline-r2c消除了事件排队和出队的开销，提高了缓存局部性，从而提供了更好的性能。但是，inline-r2c还需要一种新的编程模型，并强制上层应用程序使用零拷贝原始数据包式的读/写接口。此外，由于应用程序层代码必须与网络堆栈协同，因此inline-r2c仅在LibOS模型中可用。相比之下，batch-r2c在更传统的epoll-like或libev-like编程模型中工作，并且与传统的BSD-Socket-like接口兼容。实际上，我们将inline-r2c部署在面向性能的服务（如EBS）上，并出于兼容性考虑在对象存储等服务中使用batch-r2c。
> 
> r2c设计可以显著减少开销并提高性能。首先，应用程序和堆栈处理之间没有上下文切换。其次，由于网络堆栈在每次迭代中从NIC接收一批固定大小的数据包，因此允许上层应用程序及时处理它们（即无需缓冲数据包）。因此，CPU可以直接从L1和L2缓存中获取大部分数据，特别是在inline-r2c中。此外，由于缓冲数据包很少，因此DDIO不会填满最后一级缓存（LLC），进一步提高了缓存命中率。

##### （2）跨层零拷贝

- 用引用计数存对象，从DPDK驱动一直到上层app都用同一套引用计数。

- 用一个 slab 把内存分成每块 2M，这样可以在块首存元数据以支持引用计数。

- 驱动要找物理地址也可以直接从 slab 这边找，不用去查页表：

- - 只要 slab 块(2M大小)的元信息里存一下自己的物理地址，就可以快速从`(slab块号，offset)`拿到`(slab物理地址, offset)`

- 延迟释放：驱动处理完一个包不会立即释放buffer，而是等下一轮用到同一个buffer或者超时才释放

##### （3）兼容内核栈

其他 user-space TCP （指IX）需要占用整个网卡，这样整台机器就只能走它这个方案。如果有其他不兼容的应用（指非 TCP 但可以用传统内核栈跑通），就没法正常运行了。

## 测试

对比四种方法：

- 本文的方法，称为LUNA

- linux kernel TCP

- mTCP（[OSDI'14 的一篇]([mTCP: a Highly Scalable User-level TCP Stack for Multicore Systems | USENIX](https://www.usenix.org/conference/nsdi14/technical-sessions/presentation/jeong)）

- VPP（Vector packet processing，每次batch处理多个包）[详细见这里](https://s3-docs.fd.io/vpp/23.10/aboutvpp/scalar-vs-vector-packet-processing.html)

LUNA和 linux kernel TCP 开了 tso 和 lro，mTCP 只支持 lro，VPP 都不支持。

> TSO(TCP Segmentation Offload)是指把TCP包分段传输（由于包的长度限制）这个过程卸载到网卡上，LRO(Large Receive Offload)同理，是把合并TCP包的过程卸载到网卡上。

> 吞吐和延迟都有提升但没太多特点，没仔细看。
