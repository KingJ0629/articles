接着上一篇的分析：[链接](https://github.com/KingJ0629/articles/blob/main/%E8%B7%A8%E8%BF%9B%E7%A8%8B%E6%96%87%E4%BB%B6%E5%85%B1%E4%BA%AB%E6%96%B9%E6%A1%88.md)

### tmpfs介绍

tmpfs是一种虚拟内存文件系统，而不是块设备。是基于内存的文件系统，创建时不需要使用mkfs等初始化</br>
它最大的特点就是它的存储空间在VM(virtual memory)，VM是由linux内核里面的vm子系统管理的。</br>
linux下面VM的大小由RM(Real Memory)和swap组成,RM的大小就是物理内存的大小，而Swap的大小是由自己决定的。
Swap是通过硬盘虚拟出来的内存空间，因此它的读写速度相对RM(Real Memory）要慢许多，当一个进程申请一定数量的内存时，如内核的vm子系统发现没有足够的RM时，就会把RM里面的一些不常用的数据交换到Swap里面，如果需要重新使用这些数据再把它们从Swap交换到RM里面。如果有足够大的物理内存，可以不划分Swap分区。</br>
VM由RM+Swap两部分组成，因此tmpfs最大的存储空间可达（The size of RM + The size of Swap）。 但是对于tmpfs本身而言，它并不知道自己使用的空间是RM还是Swap，这一切都是由内核的vm子系统管理的。</br>
tmpfs默认的大小是RM的一半，假如你的物理内存是1024M，那么tmpfs默认的大小就是512M，一般情况下，是配置的小于物理内存大小的。</br>
tmpfs配置的大小并不会真正的占用这块内存，如果/dev/shm/下没有任何文件，它占用的内存实际上就是0字节；如果它最大为1G，里头放有100M文件，那剩余的900M仍然可为其它应用程序所使用，但它所占用的100M内存，是不会被系统回收重新划分的。</br>
当删除tmpfs中文件，tmpfs 文件系统驱动程序会动态地减小文件系统并释放 VM 资源。</br>

#### 特点
- 临时性：由于tmpfs是构建在内存中的，所以存放在tmpfs中的所有数据在卸载或断电后都会丢失；
- 快速读写能力：内存的访问速度要远快于磁盘I/O操作，即使使用了swap，性能仍然非常卓越；
- 动态收缩：tmpfs一开始使用很小的空间，但随着文件的复制和创建，tmpfs文件系统会分配更多的内存，并按照需求动态地增加文件系统的空间。而且，当tmpfs中的文件被删除时，tmpfs文件系统会动态地减小文件并释放内存资源。

#### 用途
Linux中可以把一些程序的临时文件放置在tmpfs中，利用tmpfs比硬盘速度快的特点提升系统性能。</br>



![共享文件原理](https://raw.githubusercontent.com/huanzhiyazi/articles/master/%E6%8A%80%E6%9C%AF/android/Android%E5%8C%BF%E5%90%8D%E5%85%B1%E4%BA%AB%E5%86%85%E5%AD%98%EF%BC%88ashmem%EF%BC%89%E5%8E%9F%E7%90%86/images/ashmem_transfer_fd.png)




[1] [LINUX下tmpfs介绍及使用](https://blog.csdn.net/haibusuanyun/article/details/17199617)</br>
[2] [Android Binder 分析——匿名共享内存（Ashmem）](http://light3moon.com/2015/01/28/Android%20Binder%20%E5%88%86%E6%9E%90%E2%80%94%E2%80%94%E5%8C%BF%E5%90%8D%E5%85%B1%E4%BA%AB%E5%86%85%E5%AD%98[Ashmem]/)
