# Memory Management

## 待更加巩固的地方

1. 两级页表转换过程中，权限位需要更加了解；
2. EIP寄存器
3. 对于虚拟地址和逻辑地址的理解，虚拟地址空间是相对于物理地址来说的，也就是我们虚拟出来这么一块地址空间，这块地址空间是不存在的，上面的地址就叫做虚拟地址。那么逻辑地址是什么呢？个人觉得逻辑地址在某种程度上等同于虚拟地址，但是逻辑地址更多的是在编程的时候，用到的，逻辑地址经过推导就得到了虚拟地址。

## 0. 前言

由前面的启动那章我们已经大致了解到，BIOS最后会将启动代码Boot Loader加载到0x7C00处，接着 Boot Loader 代码执行的后面会将内核代码的ELF文件头读取到0x1000开始的4KB内存中，然后根据ELF文件头将内核代码加载到0x100000开始的地方。之后先执行entry.S文件中的内容，在`kern/entry.S`中



在entry.S文件的最后调用`i386_init()`函数执行内核初始化操作，其中就包括初始化内存分配函数`mem_init()`函数，而这个函数是在`kern/pmap.c`中实现的，所以内存分配的任务只要是在`kern/pmap.c`文件中。

```c
void
i386_init(void)
{
        extern char edata[], end[];

        // Before doing anything else, complete the ELF loading process.
        // Clear the uninitialized global data (BSS) section of our program.
        // This ensures that all static/global variables start out zero.
        memset(edata, 0, end - edata);

        // Initialize the console.
        // Can't call cprintf until after we do this!
        cons_init();

        cprintf("6828 decimal is %o octal!\n", 6828);

        // Lab 2 memory management initialization functions
        mem_init();

        // Drop into the kernel monitor.
        while (1)
                monitor(NULL);
}
```



## 1. 保护模式下的地址转换

x86保护模式下内存管理架构是：分段和分页的（页转化的）。通过这种机制，可以将一个虚拟地址转化为物理地址。x86中的一个**虚拟地址**包括了一个段选择子（segment selector）和一个段内偏移（segment offset ）( a *virtual address* consists of a segment selector and an offset within the segment)。虚拟地址通过段地址转化机制之后就成了**线性地址**（Linear Address），线性地址再经过CPU的MMU部件的页式转化就得到了物理地址（Physical Address）。整个转化如下图所示： 

![](./image/virtual_linear_physical_translation0.jpg)

上述过程中更加详细的转化图如下所示：

![](./image/virtual_linear_physical_translation1.jpg)



### 段机制

对于段机制转化来说，GDT/LDT表（全局段描述符表/本地段描述符表）是存放关于某个运行在内存中的程序的分段信息的，比如某个程序的代码段是在哪里开始的，有多大；这个程序的数据段又是从哪开始的，有多大；这两个表是同类型的表，每一个表项都包括三个字段：

- Base：32位，代表这个程序的这个段的基地址

- Limit：20位，代表这个程序的这个段的大小

- Flags：12位，代表这个程序的这个段的访问权限

当给出selector和offset之后，首先根据selector在GDT/LDT表中找到这个段的段表项位置（selector代表这个段对应的段表项在GDT/LDT表中的索引），之后根据该段表项的Flags字段来判断是否可以访问这个段的内容，如果可以访问则把Base字段的内容取出，直接与offset相加，就得到了线性地址（linear address）。

---

其中段表项也就是所谓的段描述符，它是由编译器、链接器、加载程序或者操作系统生成的。上图中的段表项示例图是整合之后的，实际上常见的两种段表项格式如下所示：

![](./image/lab2_dt0.jpg)

**对于Base字段而言**，处理器会将三个Base片段（fragment）整合到一块，形成一个32bit的值；

**对于Limit字段而言**，处理器会整合Limit区域的两个部分，形成一个20bit的值。同时处理器解析这个字段有两种方式，方式的选择由`granularity `位决定：

- 当该位没有被设置的时候，单位为1byte，定义limit的上限为1MB
- 当该位被设置了的时候，单位为4KB，定义limit的上限为4GB

**Granularity bit（G）**：决定limit区单位的大小，具体如上所阐述的那样；

**TYPE**：区别不同种类的描述符

**DPL**：用作保护机制

**Segement-Present bit（P）**：

- 当该位设置为0的时候，那么这个描述符在地址转换中是无效。当这个描述符对应的段选择器被加载到段寄存器的时候（a selector for the descriptor is loaded into a segment register.），处理器将会产生一个异常，同时段描述符变为如下格式：

  ![](./image/lab2_dt1.jpg)

- 当该位设置为1的时候，那么操作系统可以自由访问。

同时操作系统只有在以下两种情况的时候才会清除该位：

- 当段所生成的线性地址空间没有被叶机制所映射时；
- 当该段没有出现在内存中时

**Accessed bit（A）**：当这个段是可以被访问的时候，那么该位为1。比如：一个段描述符的段选择器（selector）被加载到段寄存器的时候或者这个段选择器被一条测试语句的时候。实现虚拟地址的操作系统，在段层面通过检测和清除该位来监控段的使用。所以说创建和维护描述符是系统软件的责任。

---

段描述符是存储在描述符表中，有GDT/LDT这两种表，每张表由8字节长的描述符条目组成的数组组成，表的长度是可变的，可能包含的数量会达到8192（2^13）个描述符。但是无论如何，GDT第一条条目（index=0）是不会被处理器所使用的。同时处理器通过GDTR（GDT寄存器）或LDTR（LDT寄存器）来定位GDT或LDT在内存中的位置，这两个寄存器会存放表开始的线性地址和段的限制。

![](./image/lab2_GDT_LDT.jpg)

---

段选择器通过指定段描述符表并在表中标注出描述符的索引来找到一个描述符。段选择器可以作为指针变量中的一个字段对应用程序可见的，但是选择器的值通常由链接器或者链接加载器所指定。

![](./image/lab2_selector.jpg)

**Index**：在段描述符表中选择描述符。处理器只需将这个索引值乘以8(描述符的长度)，并将结果添加到描述符表的基本地址中，以便访问表中适当的段描述符；

**Table Indicator**： 代表哪种描述符表，0是GDT，1是LDT；

**Requested Privilege Level**：被保护机制所使用。

---

段寄存器（Segment Registers）：

80386将描述符信息存储在段寄存器中，因此避免每次访问内存时都需要访问描述符表。每一个段寄存器有一个可见的部分也有一个不可见的部分，如下图所示：

![](./image/lab2_sg.jpg)

这些可见的部分可以被程序所操作，就像他们是一个16位的寄存器一样，这些不可见的部分是被处理器所操作。处理器自动从描述符表中提取基本地址、限制、类型和其他信息，并将它们加载到段寄存器的不可见部分。当指令需要操作段中的数据时，因为段中的选择器已经被段寄存器了，一些相关信息也被加载到了段寄存器，所以处理器可以将指令中提供的段相对偏移量直接添加到段的基址中，而不需要额外的开销。

### 页机制

在接下去是对页表机制的，开启分页之后，当处理碰到一个线性地址后，它的MMU部件会把这个地址分成3部分，分别是页目录索引（Directory）、页表索引（Table）和页内偏移（offset），这3个部分把原来32位的线性地址分成了10+10+12的3个片段。CPU首先从CR3寄存器中获得页目录物理地址，之后根据线性地址的高10位获得页目录项，由页目录项所存储的高20位找到页表首地址，之后取出中间10位取得页表项，由页表项所存储的高20位获得物理页的起始地址，将该地址+12位页内偏移就得到了真正的物理地址。

然而对于JOS操作系统来说，因为`boot/boot.S`中引入了GDT这么一个全局描述符表，这个表把所有的段的基址设置为0，界限设置为0xffffffff，相当于禁止了分段转化功能，因此这段选择子没有实际上的作用了，相当于虚拟地址的偏移量就是线性地址了。

```assembly
.p2align 2                                # force 4 byte alignment
gdt:
  SEG_NULL                              # null seg
  SEG(STA_X|STA_R, 0x0, 0xffffffff)     # code seg
  SEG(STA_W, 0x0, 0xffffffff)           # data seg
```



>  其中每个页的大小为4KB（因为页内偏移为12位）。

回顾lab1的part3，引入了一个简单的页表，使得内核程序可以跑在它所要求的链接地址上（虚拟地址）0xf0100000上，当然内核程序真实的跑在0x00100000上。但是这个页表只能映射4MB的内存，但是在JOS操作系统，我们希望把这种映射扩展到头256MB的物理内存中，同时把256MB的物理内存映射到虚拟内存为0xf0000000开始处，同时映射到其他的虚拟空间中去。



需要注意的是我们编写的C语言程序中的指针的值其实是虚拟地址中段内偏移部分的值。

页目录最多包含1K的页表

页表本身就是页，而页的大小是4KB的，因此它本身包括了1024个页表项（page specifiers）。

## 2. 物理内存的管理

这块主要实现的是物理内存的管理，在内存管理中，操作系统必须要追踪记录哪些内存区域是空闲，哪些是被占用的，JOS内核是以页（page）为最小粒度来进行管理的，页大小为4KB，它使用MMU去映射和保护每一块被分配出去的内存。在上文中`i386_init`函数我们已经看到了`mem_init`函数的调用，下面来看一下`kern/pmap.c`中的`mem_init`函数。在这个函数中，首先是调用了`i386_detect_memory()`这个函数，这个函数的功能就是检测现在系统中有多少可用的内存空间。

```c
static void
i386_detect_memory(void)
{
        size_t basemem, extmem, ext16mem, totalmem;

        // Use CMOS calls to measure available base & extended memory.
        // (CMOS calls return results in kilobytes.)
        basemem = nvram_read(NVRAM_BASELO);
        extmem = nvram_read(NVRAM_EXTLO);
        ext16mem = nvram_read(NVRAM_EXT16LO) * 64;

        // Calculate the number of physical pages available in both base
        // and extended memory.
        if (ext16mem)
                totalmem = 16 * 1024 + ext16mem;
        else if (extmem) 
                totalmem = 1 * 1024 + extmem;
        else
                totalmem = basemem;

        npages = totalmem / (PGSIZE / 1024); 
        npages_basemem = basemem / (PGSIZE / 1024);

        cprintf("Physical memory: %uK available, base = %uK, extended = %uK\n",
                totalmem, basemem, totalmem - basemem);
}
```

最开始的时候说过，0xA0000\~0x100000，这部分是IO hole，是不可用的；那么剩下的，0x00000\~0xA0000，这部分叫做basemem，是可用的；IO hole后面那段0x100000~0x，这部分则叫做extmem，也是可以用的，是最重要的内存区域。这个函数一些变量的意思如下：

```
#define PGSIZE      4096   // 页大小为4096B，即4KB
npages:整个内存的页数
npages_basemem:basemem段的页数
```

获得了可使用的物理内存空间之后，之后要为页目表分配一块内存空间，大小为一个`PGSIZE`即一个页的大小，这个页目表在上文已经提到是分页转化机制中一个非常重要的表：

```c
// create initial page directory.
kern_pgdir = (pde_t *) boot_alloc(PGSIZE);
memset(kern_pgdir, 0, PGSIZE);
```

而`boot_alloc`是我们要实现的一个函数，这个函数的核心就是维护一个静态变量`nextfree`，这个变量代表下一个可以使用的空闲内存空间的虚拟地址，所以每次想要分配n个字节的内存时，我们都需要修改这个变量的值。

```c
static void *
boot_alloc(uint32_t n)
{
        static char *nextfree;  // virtual address of next byte of free memory
        char *result;

        // Initialize nextfree if this is the first time.
        // 'end' is a magic symbol automatically generated by the linker,
        // which points to the end of the kernel's bss segment:
        // the first virtual address that the linker did *not* assign
        // to any kernel code or global variables.
        // 个人理解：假如是第一次分配出来的内存的话，
        // 这个分配出来的内存是不给任何内核代码或者全局变量使用的，
        // 所以还需要在分配
        if (!nextfree) {
                extern char end[];
                nextfree = ROUNDUP((char *) end, PGSIZE);
        }

        // Allocate a chunk large enough to hold 'n' bytes, then update
        // nextfree.  Make sure nextfree is kept aligned
        // to a multiple of PGSIZE.
        //
        // LAB 2: Your code here.
        cprintf("boot_alloc, nextfree:%x\n", nextfree);
        result = nextfree;
        if(n != 0){
                nextfree = ROUNDUP(nextfree+n, PGSIZE);
                //if((uint32_t)nextfree - KERNBASE > (npages * PGSIZE)){
        }


        return result;
}
```



`page2pp(page)` 我们可以很轻松的得到该 PageInfo 结构对应的物理内存的起始位置；

## 3.内核内存空间

JOS将32位线性地址空间分为两部分，用户进程在低地址区域，内核占据了高区域。这个是分界线是被定义在`inc/memlayout.c`中的`ULIM`，大约有256MB的虚拟地址空间分配给内核。

![这边是有一张图的](./image/JOS_mem_layout.jpg)

### 权限

我们在页表中使用permission位，以确保用户的代码只能访问用户部分的地址空间，否则当用户中的代码有bug时，它将会修改kernel的数据，造成严重的事故或者轻微事故。而且用户的代码可能会偷取其他环境中私有的数据，注意`PTE_W`同时影响用户和内核代码。

用户环境下，将没有任何权限去访问在ULIM上面的内存，然而kernel却可以去读或者写这块内存。对于地址空间[UTOP,ULIM)，kernel和用户环境有着相同的权限，他们只能读不能写，这段内存是将内核中只读的数据结构展示给用户。对于地址在UTOP下面的，用户可以使用，可以修改这部分地址空间的内容。

### 初始化内核地址空间

### 地址布局空间的重新选择

进程的虚拟地址空间的布局不是只有我们讨论的这种唯一的情况，我们也可以把内核映射到低地址处，但是JOS之所以现在这么做，是为了保证x86的向后兼容性。甚至有可能，我们可以设计出一种布局方式，这种布局下，不用为内核自己留下任何固定的部分，但是可以让用户等级的程序无条件的使用4GB的虚拟空间，但是这种布局下，我们也要完全保护好内核不被其他进程访问，进程之间也不能随便访问。

## 附：问题解答

- 问题1：

x是虚拟地址，主要掌握一点C语言中的指针都是虚拟地址。

- 问题2：就是说页目录中，分别对应的是什么？

因为我们根据linear address中10、10、12的分法，最高的10位是指在页目表中的索引，所以对于KERNBASE来说，它的虚拟地址为0xf0000000，取最高10位为0x3c0，所以这个目录索引0x3c0对应的是KERNBASE.

- 问题3：我们把kernel和用户环境放在相同的地方，为什么用户的程序不能写或者读内核的内存？什么样特殊的机制保护内核的地址。

正常的操作系统通常采用两个部件来完成对内核地址的保护，一个是通过段地址机制来实现，但是JOS中分段功能并没有实现（因为全都设为0了）。二就是通过分页机制来实现，通过把页表项中的Supervisor/User位置0，那么用户态的代码就不能访问这个内存中的这个页了。

- 问题四：这个操作系统最大支持的内存大小是多大？

因为这个操作系统，分配了4MB的空间来存放所有物理页的PageInfo结构体信息，每个结构体的大小为8B，所以能存放512K个的PageInfo结构体，所以一共可以出现512K个物理页，每个物理页大小为4KB，所以支持的最大内存为2GB。

- 问题五：如何达到了最大数量的物理内存，那么所需要的额外开支是多大？

首先是需要存放所有的PageInfo，需要4MB，需要存放页目表表，kern_pgdir，4KB，之后是存放页表512*4KB = 2MB，所以额外需要6MB+4KB

- 问题六：`entry.S`文件中，当分页机制开启的时候，寄存器EIP的值是一个很小的值，在哪个位置代码才开始运行在高于KERNBASE的虚拟地址空间中的？当程序位于开启分页之后到运行在KERNBASE之上这之间的时候，EIP的值是很小的值，怎么保证可以把这个值转化为真实物理地址？

在`entry.S`文件中有一个指令`jmp *%eax`，这个指令要完成跳转，就会重新设置EIP的值，把它设置为寄存器eax中的值，而这个值是大于KERNBASE的，所以就完成了EIP从小的值到大于KERNBASE的值的转换。

## 附：段描述符



## 附：分页、分段、段页式存储管理方式（本科书本）

分页存储管理将进程的逻辑地址空间分成若干个页，并为各页加以编号；同时把内存的物理地址空间分成若干个块，同时也为它们加以编号。在为进程分配内存时，以块为单位，将进程的若干个页分别装入到多个可以不相邻接的物理块中。

页表：在分页系统中，允许将进程的各个页分散地存储在内存的任一物理块中，为保证进程仍然能够正确地运行，即能在内存中找到每个页面所对应的物理块，系统又为每个进程建立了一张页面影响表，简称页表。进程地址空间内的所有页（0~n），依次在页表中有一页表项，其中记录了相应页在内存中对应的物理块号。**页表的作用是实现从页号到物理块号的地址映射。**

【此处需要一张图】

 