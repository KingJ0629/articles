这篇文章是对上一篇文章的一些内容补充和详细分析 [上一篇](https://github.com/KingJ0629/articles/tree/main)

### 文件描述符

linux系统中的文件描述符是什么？在回答这个问题前先来看一下linux系统中进程是什么？</br>
在linux系统中进程实际上就是一个结构体，而且线程和进程使用的是同一个结构体，其部分源码如下：
```C++
struct task_struct {
	// 进程状态
	long			  state;
	// 虚拟内存结构体
	struct mm_struct  *mm;
	// 进程号
	pid_t			  pid;
	// 指向父进程的指针
	struct task_struct __rcu  *parent;
	// 子进程列表
	struct list_head		children;
	// 存放文件系统信息的指针
	struct fs_struct		*fs;
	// 一个数组，包含该进程打开的文件指针
	struct files_struct		*files;
};
```

可以看到在结构体中有一个files字段，它记录着该进程打开的文件指针，而我们所说的文件描述符实际上就是这个files数组的索引，他们的关系如下图所示：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/086325e6ee324a8fbc1c433f5fd7c934~tplv-k3u1fbpfcp-watermark.image)

为了画图方便，这里将fd1和fd2都写成了1，实际上每个进程被创建时，files的前三位被填入默认值，分别指向标准输入流、标准输出流、标准错误流。所以进程的文件描述符默认情况下 0 是输入，1 是输出，2 是错误。
从图中可以看出fd1 和fd2 其实并没有直接的关系，那么进程2是如何通过进程1的fd1 来生成一个同fd1 指向同一个 文件呢？
回想一下我们是怎么把fd1 转成fd2 的，是通过Binder#transact 方法实现的，因此我们来看一下Binder 的源码是如何做的

```C++
//Binder.c
static void binder_transaction(struct binder_proc *proc,
			       struct binder_thread *thread,
			       struct binder_transaction_data *tr, int reply) {

    switch(fp->type) {
            case BINDER_TYPE_FD: {
			int target_fd;
			struct file *file;

			// 通过进程1的fp->handle获取到真正的文件，在内核中是唯一的fd指向它
			file = fget(fp->handle);
			// 获取目标进程中未使用的文件描述符fd
			target_fd = task_get_unused_fd_flags(target_proc, O_CLOEXEC);
			// 将目标进程的文件描述符fd和该file进行配对，这样目标进程就能通过target_fd找到file
			task_fd_install(target_proc, target_fd, file);

		 } break;
   }
}
```

看了源码我们发现原理非常简单，其实就是通过内核中的Binder帮我们进行转换的，因为内核是有所有用户进程信息，所以它可以轻松的做到这一点。
还有一点需要说明的是，在上图中的file1,file2,file3并不一定是存在磁盘上的物理文件，也有可能是抽象的文件（虚拟文件），而本篇文章说的匿名共享内存 实际上就是映射到一个虚拟的文件，至于这块的内容可以看一下Linux的tmpfs文件系统 。


### 虚拟内存
虚拟内存————vm 不是一块实际存在的内存，为了缓解物理内存不够用的问题，往往从磁盘空间中开辟一块空间作为物理内存的扩展，这块开辟的磁盘空间被称为交换区——swap。

但是我们知道，CPU 是不能直接访问磁盘空间的，所有的数据都必须先从磁盘空间读入物理内存之后才能被 CPU 访问。所以 swap 的工作方式并不是简单地当物理内存用完之后直接从交换 区读取数据，而是有一个换出（swap out）换入（swap in）的过程。

具体来说，根据局部访问原理，我们总是认为物理内存中存储的总是最近最常访问的数据，所以当我们要访问数据时，大多数时候可以直接从物理内存中访问到。但是，当访问数据不在物理内存中时，内核需要将数据从磁盘载入物理内存（换入），同时将最近最不常使用的数据从内存中暂存到磁盘交换区以留出足够的空间来存储换入的数据（换出）。

这样，即使我们需要运行的程序和数据量超过了物理内存的上限，但是我们仍然感觉内存空间比实际物理内存要大，而这块更大的内存并不是因为实际物理内存有这么大，而是通过动态的换入换出过程虚拟出来的。由于实际不是一块真实的内存，所以将其称为虚拟空间更为合适，这个虚拟空间的大小为：

```
vm = 物理内存 + swap
```

对于进程来说，它能看到的是虚拟空间，所以虚拟空间需要进行统一编址。一个进程在访问一个虚拟内存页的时候，实际是要去访问其对应的物理内存页，所以虚拟地址和物理地址之间需要一张映射表去根据虚拟地址查找到对应的物理地址，这个工作由内存管理单元————MMU（一般集成在 CPU 中）来完成。如果 MMU 映射到的物理页面存在，则直接访问，否则触发缺页中断，需要启动磁盘执行换入换出操作，重新建立该页的映射关系。

![](https://raw.githubusercontent.com/huanzhiyazi/articles/master/%E6%8A%80%E6%9C%AF/android/Android%E5%8C%BF%E5%90%8D%E5%85%B1%E4%BA%AB%E5%86%85%E5%AD%98%EF%BC%88ashmem%EF%BC%89%E5%8E%9F%E7%90%86/images/vm_map.png)

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

### 缺页中断机制

mmap 过程只是简单的将虚拟空间和文件指定区域建立了对应关系，这在逻辑上表示以后可以像读写虚拟内存 vma 一样读写文件中的指定区域。但是实际上，还并没有为文件上该块区域分配相应的物理内存。那么如何对映射的文件区域进行访问呢？这就要用到前面提到的 MMU 转义和缺页中断机制了：

![](https://raw.githubusercontent.com/huanzhiyazi/articles/master/%E6%8A%80%E6%9C%AF/android/Android%E5%8C%BF%E5%90%8D%E5%85%B1%E4%BA%AB%E5%86%85%E5%AD%98%EF%BC%88ashmem%EF%BC%89%E5%8E%9F%E7%90%86/images/mmap_no_page.png)

如图所示，mmap 会返回虚拟空间的起始地址 addr，这实际也是分配的 vma 链表中的第一个 vma 块的起始地址，即 vma->vm_start。对于调用进程而言，可以直接往 addr 指向的虚拟空间进行读写，而不用通过 read/write 系统调用来读写映射的文件了。

- 首先，向 addr 指向的虚拟空间读写数据。
- 由 MMU 将 addr 转义成具体的物理地址，但是因为没有为文件分配物理空间，无法找到对应的物理地址，发生缺页，调用缺页中断程序。
- 缺页中断先试图从 swap 中寻找对应的页面，如果找到，说明文件被载入过且因为其它页的换入操作被换出过了，于是直接从 swap 中换入缺失的页面到物理内存。
- 若 swap 中也没有对应的页面，说明文件从未被载入过，于是从磁盘空间载入指定文件的区域到物理内存。
- 返回物理内存地址，实现虚拟页到物理页的映射关系。

以上，缺页发生后无论是从个 swap 中载入缺页还是从文件中载入缺页，物理内存都可能已满，这时缺页机制会执行换出操作，将其它最近最不常使用的页面换出到 swap 区。


### 匿名共享文件的原理

![共享文件原理](https://raw.githubusercontent.com/huanzhiyazi/articles/master/%E6%8A%80%E6%9C%AF/android/Android%E5%8C%BF%E5%90%8D%E5%85%B1%E4%BA%AB%E5%86%85%E5%AD%98%EF%BC%88ashmem%EF%BC%89%E5%8E%9F%E7%90%86/images/ashmem_transfer_fd.png)

看了上面相关的一些概念，我们在回过头来看匿名共享内存的原理，就会理解得更加透彻。几个过程我们大致归纳如下：
- 服务端进程 通过 `mFD = native_open(name, length)` 在tmpfs中申请一块 ashmem共享内存 asma(asma 是一个 ashmem_area 结构体，用于描述 ashmem共享内存)，这里mFD就是赋值给图中的sfd
- 服务端进程 通过 `mAddress = native_mmap(mFD, length, modeToProt(mode))`映射到服务端进程中的虚拟空间中，`mAddress`就是这个空间地址的起始地址，有了这个起始地址，服务端进程就能在内存中读写文件
- Binder通信在内核态的时候，是有所有进程的信息的，在拿到服务端进程的sfd后，找到sfd对应的tmpfs中的文件，然后把客户端进程中生成的一个空cfd指向这个文件，就相当于达到了一次转换的过程
- 客户端进程 通过 `mAddress = native_mmap(mFD, length, modeToProt(mode))`，其中的mFD是客户端自己的cfd，完成了映射关系，得到了自己进程中的`mAddress`，然后就可以在自己进程中操作内存，从而完成了整个文件共享的实现，但此时各个进程操作的都是内存，直到完成通信后，tmpfs中的内存才会被写入到硬盘文件中。

### 问题
###### 一、Android为什么设计一个匿名共享内存，共享内存不能满足需求吗？</br>
首先我们来思考一下共享内存和Android匿名共享内存最大的区别，那就是共享内存往往映射的是一个硬盘中真实存在的文件，而Android的匿名共享内存映射的一个虚拟文件。这说明Android又想使用共享内存进行跨进程通信，又不想留下文件，同时也不想被其它的进程不小心打开了自己进程的文件，因此使用匿名共享内存的好处就是：

- 不用担心共享内存映射的文件被其它进程打开导致数据异常。
- 不会在硬盘中生成文件，使用匿名共享内存的方式主要是为了通信，而且通信是很频繁的，不希望因为通信而生成很多的文件，或者留下文件。

###### 二、为什么叫匿名共享内存？明明通过iotc设置了名字的？
这个问题在我看来是我之前对匿名这个词有些误解，其实匿名并不是没有名字，而是无法通过这些明面上的信息找到实际的对象，就像马甲一样。匿名共享内存也正是如此，虽然我们设置了名字，但是另外的进程通过同样的名字创建匿名共享内存却并不指向同一个内存了，虽然名字相同，但是背后的人却已经换了。这同时也回答上个问题，为什么匿名共享内存不用担心被其它进程映射进行数据读写（除非经过自己的同意，也就是通过binder传递了文件描述符给另一个进程），只有通过第一次分配内存时候返回的fd才能找到对应的内存块，而客户端的fd是通过Binder分享过去的，其他情况调用native_open都是申请到的新的内存，所有说只有经过自己同意，而对于其他没有授权的进程来说，这块内存就是块访问不到的匿名内存。



### 推荐阅读

[1] [LINUX下tmpfs介绍及使用](https://blog.csdn.net/haibusuanyun/article/details/17199617)</br>
[2] [Android Binder 分析——匿名共享内存（Ashmem）](http://light3moon.com/2015/01/28/Android%20Binder%20%E5%88%86%E6%9E%90%E2%80%94%E2%80%94%E5%8C%BF%E5%90%8D%E5%85%B1%E4%BA%AB%E5%86%85%E5%AD%98[Ashmem]/)</br>
[3] [Android匿名共享内存（Ashmem）](https://juejin.cn/post/6906444643089514510)</br>
[4] [Android匿名共享内存（ashmem）原理](https://github.com/huanzhiyazi/articles/issues/27)</br>
