## Introduction

在这个Lab中我们将会实现`spawn`，这是一个library call可以加载和运行磁盘上的可执行文件。之后我们将会扩展kernel和library operating system在console上运行shell。这些功能都需要一个文件系统，所以这个Lab将会引入一个简单的读写文件系统。

### Getting Started

在mergel到新的Lab5的代码之后，再次运行Lab4中pingpong、primes和forktree等几个测试例子。不过在运行测试例子之前先注释掉`kern/init.c`中的`ENV_CREATE(fs_fs) `，因为`fs/fs.c`会尝试IO操作，但是这个IO操作JOS还没有实现，同时注释掉`lib/exit.c`中的`close_all()`，因为这个函数将会调用将要在这个lab中实现的子程序。全都注释完之后，我们运行测试例子，如果Lab4代码没有包含任何bug的话，这几个测试例子是可以成功运行的。

```
// pingpong
SMP: CPU 0 found 1 CPU(s)
enabled interrupts: 1 2 4
send 0 from 1000 to 1001
1001 got 0 from 1000
1000 got 1 from 1001
1001 got 2 from 1000
1000 got 3 from 1001
1001 got 4 from 1000
1000 got 5 from 1001
1001 got 6 from 1000
1000 got 7 from 1001
1001 got 8 from 1000
1000 got 9 from 1001
1001 got 10 from 1000
No runnable environments in the system!
Welcome to the JOS kernel monitor!
Type 'help' for a list of commands.
K> 
```

```
......
CPU 0: 7951 CPU 0: 7963 CPU 0: 7993 CPU 0: 8009 CPU 0: 8011 CPU 0: 8017 CPU 0: 8039 CPU 0: 8053 CPU 0: 8059 CPU 0: 8069 CPU 0: 8081 CPU 0: 8087 CPU 0: 8089 CPU 0: 8093 CPU 0: 8101 CPU 0: 8111 CPU 0: 8117 CPU 0: 8123 CPU 0: 8147 [000013ff] user panic in <unknown> at lib/fork.c:133: sys_exofork: out of environments
Welcome to the JOS kernel monitor!
Type 'help' for a list of commands.
TRAP frame at 0xf0287f84 from CPU 0
  edi  0x00000000
  esi  0x0080153a
  ebp  0xeebfdf40
  oesp 0xefffffdc
  ebx  0xeebfdf54
  edx  0xeebfddf8
  ecx  0x00000001
  eax  0x00000001
  es   0x----0023
  ds   0x----0023
  trap 0x00000003 Breakpoint
  err  0x00000000
  eip  0x00800194
  cs   0x----001b
  flag 0x00000282
  esp  0xeebfdf38
  ss   0x----0023
K> 
```

> 需要注意的是，这个例子跟在Lab4运行的输出结果有点不一样，是因为在Lab5中`kern/env.c`的`env_free()`中将
>
> ```c
> // cprintf("[%08x] free env %08x\n", curenv ? curenv->env_id : 0, e->env_id);
> ```
>
> 这么一句话注销掉了，所以不会像Lab4那样输出了

```
// forktree test case
SMP: CPU 0 found 1 CPU(s)
enabled interrupts: 1 2 4
1000: I am ''
1001: I am '0'
1002: I am '1'
1003: I am '10'
2000: I am '00'
2002: I am '000'
1004: I am '11'
1006: I am '01'
1007: I am '001'
2001: I am '010'
3002: I am '110'
3000: I am '111'
1005: I am '100'
4002: I am '101'
1008: I am '011'
No runnable environments in the system!
Welcome to the JOS kernel monitor!
Type 'help' for a list of commands.
K> 
```

测试完之后记得取消上述的注释！！！

## File system preliminaries

这个文件系统相比Unix中的文件系统是简单了许多的，但是它也提供了基础的功能：创建、读取、写入、删除按层次目录结构组织的文件。

目前我们只开发只有一个用户的操作系统，这个系统对捕获bugs提供了足够多的保护，但是无法保护多个相互怀疑的用户。因此我们的文件系统不支持Unix中文件所属者或者权限等概念，同时我们的文件系统也不支持大部分Unix文件系统中的硬链接、符号链接、时间戳或者特殊设备文件等。

### On-Disk File System Structure

大部分Unix文件系统将磁盘空间分成两个主要类型的区域：inode regions和data regions。data region又被分成很多大的data blocks（通常是8KB或者更多），文件系统将文件数据和目录meta-data存储在这些data blocks中。Unix文件系统给文件系统中的每一个文件分配一个inode，一个文件的inode存储着文件关键的meta-data比如文件的`stat`属性和指向data blocks的指针。目录项包含了文件的名字和inode的指针，假如文件系统中很多个目录项都指向一个文件的inode，那么这个文件是hard-linked。

但是我们的文件系统不支持hard-link，所以我们不需要这种间接的等级，因此可以做一个简化：我们的文件系统不使用inodes，相反我们会在描述文件的目录项中简单存储一个文件（或者子目录）所有的meta-data。文件和目录逻辑上都是由一系列data blocks组成，但是这些data blocks是分散在磁盘上的，就像一个environment的虚拟地址空间的页就是分散在物理内存上的。文件系统environment隐藏了block分布的细节，提供了对文件任意偏移量的位置读写一系列字节的接口。文件系统environment内部处理所有的目录的修改，作为执行文件创建或删除操作的一部分。我们的文件系统允许user environment直接读取目录的meta-data，这也就意味着user environments可以自己执行目录扫描操作（比如`ls`程序）而不是不得不依赖于额外的对文件系统的特殊的调用。这种目录扫描的方式的缺点以及现代Unix阻止它的原因是它使得应用程序依赖于目录meta-data的格式，使得修改文件系统内部布局变得困难，因为还需要改变应用程序或者至少重新编译应用。

#### Sectors and Blocks

大部分磁盘不能按照字节来读取，而是按sector的单位来执行读写。在JOS中，每一个sector是512字节的。文件系统实际上分配和使用disk是按照block的来的。我们需要注意sector和block这两个术语之间的区别：sector的大小是磁盘硬件的性能，然而block的大小是操作系统使用disk的一方面（个人理解sector是硬件上的，block是软件上），并且文件系统的block 大小必须是磁盘上sector大小的整数倍。

Unix xv6文件系统的block大小跟sector是一样的都是512字节，但是大部分现代文件系统使用一个更大的block 大小，因为存储空间变得相当廉价了并且以更大的粒度来存储管理是更加有效的。

#### Superblocks

文件系统通常把存储在disk“esy to find”的位置（比如在disk的最开始或者末端）的某些磁盘blocks拿来存储文件系统所有的meta-data描述符，比如block size、disk size、任何用于找到root目录的meta-data、上一次加载文件系统的时间、上一次检查文件系统是否有错误的时间等等。这些特殊的块就叫做*superblocks*.

我们的文件系统将只有一个superblock，这个superblock将总是在disk的block 1的位置，它的layout是由`inc/fs.h`中的`struct Super`定义的。block 0通常用来存储boot loader和分区表，所以文件系统通常不使用第一块。很多真正的文件系统维持着很多superblocks，这些superblocks被复制到几个间隔很宽的磁盘区域，所以即使这些区域中的一个坏了或者disk在那个区域发生了media error，那么仍然可以找到其他几个可以使用的superblocks，然后去访问文件系统。

#### File Meta-data

文件系统中描述文件的meta-data的layout由`inc/fs.h`的`struct File`定义。这些meta-data包括了文件的名字、大小、类型（普遍文件还是目录）、包含文件内容的blocks的指针。正如上面我们所提到的，我们没有inode，所以这些meta-data是被存储在disk上的目录项中的。同时跟大多数真正的文件系统不同的是，为了简单起见对在disk和memory中的文件meta-data，使用同一个`struct File`来表示。

`struct File`中的`f_direct`数组，用来存储文件前10（NDIRECT）个block的block number，这些blocks被叫做文件的direct block。对于大小为10*4096 = 40KB 的文件来说，这意味着该文件所有block的block number将直接适合`struct File`。但是对于任何大于40KB的文件来说，我们需要有一个空间来存储剩下的文件block number，因此我们分配了额外的一个block叫做indirect block来存储4096/4 = 1024个额外的block number。因为最终我们的文件系统允许文件大小达到1034blocks。为了支持更大的文件，真正的文件系统通常支持double indirect blocks或者triple-indirect blocks。

![](./image/Lab5_0.png)

#### Directories versus Regular Files

文件系统中的`File`结构体既可以表示一个普通文件也可以表示一个目录，使用`File`结构体中的`type`成员变量来区别这两种类型的文件。文件系统用完全相同的方式来管理普通文件和目录文件上，除了以下这点：文件系统不解释与普遍文件相关联的data block的内容，然而它会将目录文件的内容解析为一系列描述文件和子目录的`File`结构体。

supberblock包含了一个`File`结构体（`struct Super`中`root`成员变量），这个`File`结构体存储着文件系统中根目录的meta-data。这个目录文件的内容是一系列`Fille`结构体，这些结构体描述的是根目录中的文件和子目录。这些子目录可能又会包含更多的`File`结构体来表示sub-subdirectories，以此类推。

## The File System

在这个Lab中，不需要去实现整个文件系统，但是需要去实现文件系统关键的部分。比如我们需要去实现将块读取到block cache中并且将他们flush回磁盘、分配磁盘blocks、映射文件的offset到磁盘blocks、实现IPC接口中的读写和打开。因为我们将不会实现整个文件系统，所以我们需要要对提供的代码和各种各样的文件系统的接口很熟悉。

### Disk Access



### The Block Cache

### The Block Bitmap

### File Operations

### The file system interface

到这一步，在file system environment本身中有了必要的函数（这些函数fs environment本身可以访问），但是这些函数需要让其他想要使用file system的environment访问。由于其他environment不能直接调用file system environment中的函数，所以我们通过构建在JOS IPC机制基础上的*remote procedure call*（**RPC**）来访问file system environment中的函数。下面是对file system server调用（read操作）的一个示意图

```
      Regular env           FS env
   +---------------+   +---------------+
   |      read     |   |   file_read   |
   |   (lib/fd.c)  |   |   (fs/fs.c)   |
...|.......|.......|...|.......^.......|...............
   |       v       |   |       |       | RPC mechanism
   |  devfile_read |   |  serve_read   |
   |  (lib/file.c) |   |  (fs/serv.c)  |
   |       |       |   |       ^       |
   |       v       |   |       |       |
   |     fsipc     |   |     serve     |
   |  (lib/file.c) |   |  (fs/serv.c)  |
   |       |       |   |       ^       |
   |       v       |   |       |       |
   |   ipc_send    |   |   ipc_recv    |
   |       |       |   |       ^       |
   +-------|-------+   +-------|-------+
           |                   |
           +-------------------+
```

上图中，虚线下面的是从普通的environment中获取read请求到文件系统中处理的简化版机制。

- 首先，`read`可以使用任何文件描述符（file descriptor），并且会分派到合适的设备读取函数；
- 当分派到`devfile_read`情况时（有好多种device type，比如pipes），`devfile_read`实现对磁盘上文件的特殊的`read`。这个函数以及在`lib/file.c`中的其他`devfile_*`函数，都实现了文件系统操作的客户端，并且工作方式大致相同，都是将参数绑定在一个请求结构体中，然后调用`fsipc`来发送IPC请求，同时会解包和返回结果；
- `fsipc`函数简单处理发送给server请求的共同细节以及获取回复；
- FS server相关的代码在`fs/serv.c`中，`serve`函数中，通过循环来不断获取来自IPC的请求，之后将请求分派到合适的处理函数，然后通过IPC把结果返回。在read的例子中，server将请求分派到`serve_read`函数
- `serve_read`函数将会对读请求的结构体进行解包，最后调用`file_read()`函数来完全执行文件的读取工作；

回顾JOS的IPC通信机制，一个environment会发送一个32-bit数字以及一个可选的共享页。对于发送一个从client到server的请求来说，我们使用32bit的数字作为请求类型（FS server RPCs是被编号了的，就像syscalls被编号了一样），并且将请求参数存储到IPC共享page上的`union Fsipc`中。在客户端，总是共享在`fsipcbuf`处的页；在服务端，我们将进来的请求页映射到`fsreg(0x0ffff000)`上。

服务端通过IPC将回复（response）返回，我们使用32bit数字作为函数的返回码。对于大部分RPCs来说都会返回，并且针对不同类型的RPCs会有不同的返回

- `FSREQ_READ` 和`FSREQ_STAT` 类型的还会返回数据，这些数据是被简单的写到客户端请求时传来的页上。由于客户端和FS server共享这个页，所以FS server没有必要发送这个页作为回复。
- `FSREQ_OPEN`类型的话，将会和客户端共享一个新的“Fd page”，这样我们将会很快回到文件描述page。



## Spawning Processes



### Sharing library state across fork and spawn

Unix的文件描述符是一个通用的概念，除包含文件文件外它还包含了pipes、console IO等等。在JOS中，每一个device类型都有一个相关联的`struct Dev`，这种结构体中有指向读写等操作函数的指针。

```c
struct Dev {
  int dev_id;
  const char *dev_name;
  ssize_t (*dev_read)(struct Fd *fd, void *buf, size_t len);
  ssize_t (*dev_write)(struct Fd *fd, const void *buf, size_t len);
  int (*dev_close)(struct Fd *fd);
  int (*dev_stat)(struct Fd *fd, struct Stat *stat);
  int (*dev_trunc)(struct Fd *fd, off_t length);
};
```

`lib/fd.c`在上述的基础之上实现了通用的Unix-like的文件描述符接口。每一个`struct Fd`表明它的device type，`lib/fd.c`中的大部分函数会调用`struct Dev`中合适的函数。`lib/fd.c`同时也在每一个应用程序environment的地址空间中维护着文件描述符表（file descriptor table），文件描述符表从虚拟地址FDTABLE处开始。这个区域为每一个文件描述符（一共有MAXFD文件描述符，当前MAXFD是32）保留着一个页的大小，这些文件描述可以被应用程序同时打开。在任何时候，一个特殊的文件描述符表的页只有在相关联的文件描述符被使用时才会被映射。每一个文件描述符在FILEDATA开始的区域也还有可选的"data page"，如果选了的话，那么device将会使用它选择的。

我们希望在`fork`和`spawn`之间共享file descriptor state，但是FD state 是被保存在用户地址空间。在`fork`中，内存将会被标记为copy-on-write，所以state将会被复制而不是共享，这意味着environments将无法在自己没有打开的文件中进行查找，并且经过`fork`之后pipes将不会工作了。在`spawn`中，内存将不会被复制，而是留在后面，那么实际上，spawn出来的environment将没有打开的文件描述符。我们将会改变`fork`来知道“library opearting system”正在使用内存的哪些区域，这些区域应该是共享的。我们将会在page table entries中设置一个没有使用过的位来表示这些区域而不是通过硬编码（hard-code）一系列的区域来当做这些区域（就像在`fork`中使用`PTE_COW`一样）。

我们在`inc/lib.h`中定义了一个新的`PTE_SHARE`，这是在Intel 和 AMD手册中用于标记“available for software use”三位中的一位。下面我们这样子规定，假如一个page table entry的这一位被设置了，那么在`fork`和`spawn`中，这个PTE将会从parent直接复制到child。需要注意的是这跟标记它为copy-on-write是不同的，因为我们想要确保这个页共享更新的。





> 为什么tate将会被复制而不是共享，这意味着environments将无法在自己没有打开的文件中进行查找，并且经过`fork`之后pipes将不会工作了？
>
> 

## The keyboard interface

## The Shell

