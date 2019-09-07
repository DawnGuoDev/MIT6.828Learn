## Introduction

在实现完Lab5之后，我们有了一个文件系统，然而没有network stack的OS是没有灵魂的OS。所以在这个Lab中，我们准备写一个network interface card的驱动。这网卡是基于Intel 82540EM芯片，也被称为E1000。network card driver 可以让我们的OS连上因特网。在Lab6的，已经提供了一个network stack和network server。相关新的代码在`net/`目录和`kern/`中。下面来说说相关的实现事项：

- 实现网卡驱动之外，并且创建一个system call使得可以访问这个驱动；
- 实现缺掉的network server code ，使得在network stack和驱动之间可以传输数据包；
- 完成web server，将所有的事情捆绑在一起。使用新的web server将可以从文件系统中提供文件；

大部分内核设备驱动的代码需要我们自己从头开始写，同时这个Lab比之前的Lab提供了更少的指导：没有一个文件框架、没有已经固定的system call 接口并且很多设计的决策都是由自己决定的。所以，建议你开始做练习之前先阅读所有的作业。

### 相关资源

1. [ QEMU's documentation ](https://qemu.weilnetz.de/doc/qemu-doc.html#Using-the-user-mode-network-stack)
2. [lwIP - A Lightweight TCP/IP stack - Summary](https://savannah.nongnu.org/projects/lwip/)



## QEMU's virtual network

我们将使用qemu 的用户模式network stack因为它不需要任何管理的权限即可运行。Makefile文件已经被修改，可以使用qemu的用户模式network stack和虚拟的E1000网卡。

默认的，qemu提供了一个虚拟的路由器，该路由器运行在IP 10.0.2.2上，同时给JOS分配了一个IP地址10.0.2.15。为了让事情简单，我们在`net/ns.h`中将这些默认的参数硬编码进network server。

```c
#define IP "10.0.2.15"
#define MASK "255.255.255.0"
#define DEFAULT "10.0.2.2"
```

在QEMU内部虚拟网络之外，JOS的10.0.2.15的地址将毫无意义，然而qemu的虚拟网络允许JOS连上因特网，这也就是说QEMU充当了NAT的角色，所以我们不能直接连接到运行在JOS内部的serves，即便从运行qemu的宿主机上都不能直接连接到。为了解决这个，我们配置QEMU，在宿主机的某些端口上运行一个server，这个server简单连接到JOS上的一些端口，并在宿主机和虚拟网络之间传输数据。

你将会运行JOS的servers在ports 7（echo）和80（http）。要了解qemu将这些端口转发到你宿主机的哪些端口，可以使用`make which-ports`.

```bash
$ make which-ports
Local port 25001 forwards to JOS port 7 (echo server)
Local port 25002 forwards to JOS port 80 (web server)
```

makefile同时提供了` make nc-7`和`make nc-80`，这两个命令可以让你在你的终端中直接跟运行在这些端口上的服务通信（但是这些只会连接到运行的qemu实例，所以必须单独启动qemu，也就是得让qemu运行着）。

> 在这块你可能不好理解，假如你接触过VMware 虚拟机的话，你会发现Vmware有两种方式连上网络一种是桥接，一种是NAT。NAT的意思就是说，内部网络（如局域网）一些主机已经分配到IP地址了，但是这些是仅仅可以在内部网络使用的（如192.168.200.2），无法跟外网通信，这是来一台NAT路由器，NAT路由器有两个IP地址，一个是内部网路的地址（如192.168.200.254），一个是外网的地址（218.197.10.2)。那么内部网络想与外网通信时都会使用这个外网的IP地址来跟外界通信，而外网发给这个网络的地址都是发送到218.197.10.2。NAT的全称是网络地址转换，也就相当把内网的地址转换成了一个外网的地址，这样子可以使用少量公有IP地址代表较多的私有IP地址了。
>
> ![](./image/Lab6_0.jpg)

### Packet Inspection（查看数据包）

同时makefile 也配置了qemu的network stack将所有的进来或者出去的packets记录到qemu.pcap中。要获得捕获的packets的hex/ASCII的形式可以使用tcpdump比如：

```bash
tcpdump -XXnr qemu.pcap
```

当然，你还可以使用wireshark以图形的方式来审查这些pcap文件。Wireshark还知道如何解码和检查数百个网络协议。

### Debugging the E1000（E1000的调试）

我们使用的E1000是仿真硬件，也就是说E1000是以软件运行的，那么仿真的E1000可以用一种用户可读的格式来报告它内部的状态和任何遭遇的问题。E1000将会产生大量的debug输出，所以我们必须使能一些特定的log通道。一些比较有用的通道如下：

| Flag      | Meaning                                            |
| --------- | -------------------------------------------------- |
| tx        | Log packet transmit operations                     |
| txerr     | Log transmit ring errors                           |
| rx        | Log changes to RCTL                                |
| rxfilter  | Log filtering of incoming packets                  |
| rxerr     | Log receive ring errors                            |
| unknown   | Log reads and writes of unknown registers          |
| eeprom    | Log reads from the EEPROM                          |
| interrupt | Log interrupts and changes to interrupt registers. |

为了使能`tx`和`txerr`，使用如下命令。需要注意的是`E1000_DEBUG` 标志只有在QEMU 6.828版本中有效。

```bash
make E1000_DEBUG=tx,txerr ....
```

可以使用软件仿真硬件来进一步调试，如果你被卡住了，并且无法理解为什么E1000没有回复你想要的，你可以看一下`hw/net/e1000.c`中qemu E1000的实现。

## The Network Server

从头开始（from scratch）写一个network stack是一项很艰难的工作。所以我们将会使用IwIP，一个轻量级开源的TCP/IP协议套件，IwIP里面就包含了一个network stack。在这个作业中，对于我们来说，IwIP就是一个黑盒，它实现了一个BSD 的socket接口并且有一个packet输入端口和packet output端口。

network server是四个environments的组合：

- core network server environment（包括了socket call分派和IwIP）
- input environment
- output environment
- timer environment

下面的这张图展示了这四个不同的environments和它们的关系，同时展示了包含设备驱动程序在内的完整的系统。在这个Lab中将要实现的图中绿色突出的部分。

![](./image/Lab6_1.png)

### The Core Network Server Environment

core network server environment由socket call dispatcher和IwIP组成。socket call dispatcher工作原理同file server一样。user environment使用stubs（在`lib/nsipc.c`）发送IPC messages给core network environment。如果查看`lib/nsipc.c`你将会发现，我们找到core network server的方法跟找到file server的方法是一样：`i386_init`使用`ENV_TYPE_NS`创建了NS environment，那么使用`ENV_TYPE_NS`类型可以找到NS environment。从而实现与NS environment的IPC通信。

```c
// core network server
static int
nsipc(unsigned type)
{
  static envid_t nsenv;
  if (nsenv == 0)
    nsenv = ipc_find_env(ENV_TYPE_NS);

  static_assert(sizeof(nsipcbuf) == PGSIZE);

  if (debug)
    cprintf("[%08x] nsipc %d\n", thisenv->env_id, type);

  ipc_send(nsenv, type, &nsipcbuf, PTE_P|PTE_W|PTE_U);
  return ipc_recv(NULL, NULL, NULL);
}

// file server
static int
fsipc(unsigned type, void *dstva)
{
  static envid_t fsenv;
  if (fsenv == 0)
    fsenv = ipc_find_env(ENV_TYPE_FS);
    
  static_assert(sizeof(fsipcbuf) == PGSIZE);

  if (debug)
    cprintf("[%08x] fsipc %d %08x\n", thisenv->env_id, type, *(uint32_t *)&fsipcbuf);
    
  ipc_send(fsenv, type, &fsipcbuf, PTE_P | PTE_W | PTE_U);
  return ipc_recv(NULL, dstva, NULL);
} 
```

对于每一个user environment IPC，在networks server中的dispatcher调用合适的BSD socket interface 函数来代表相应的用户。

常规的user environment不会直接调用`lib/nsipc.c`中 `nsipc_*`等函数（如`nsipc_close`、`nsipc_send`等），相反他们会调用的是`lib/sockets.c`中的函数，这个文件中的函数提供了基于**文件描述符的sockets API**。因此user environment通过文件描述符来指向sockets，就像使用文件描述符指向在disk上的file一样。

```c
struct Dev devsock =
{
  .dev_id = 's',
  .dev_name = "sock",
  .dev_read = devsock_read,
  .dev_write =  devsock_write,
  .dev_close =  devsock_close,
  .dev_stat = devsock_stat,
};
```

有一些操作是socket特有的（比如`connect`、`accept`等），但是`read`、`write`和`close`等操作使用`lib/fd.c`中正常文件描述符设备分派代码。跟file server内部为所有打开的文件维持不同的ID一样，IwIP为所有打开的sockets分配不同的ID。在file server和network server中，我们使用存储在`struct Fd`中的信息把per-environment的文件描述符映射到不同的ID空间。

尽管file server和network server中的IPC dispatchers可能表现的差不多，但是他们也有一个根本的区别。





### The Output Environment

### The Input Environment

### The Timer Environment

## 个人小目标实现

驱动实现这个网卡驱动，顺便看一下IDE DISK的实现。