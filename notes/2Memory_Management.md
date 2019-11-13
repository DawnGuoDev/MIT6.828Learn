## 0. 前言

由前面的启动那章我们已经大致了解到，BIOS最后会将启动代码Boot Loader加载到0x7C00处，接着 Boot Loader 代码在执行的后面会将内核代码的ELF文件头读取到0x1000开始的4KB内存中，然后根据ELF文件头将内核代码加载到0x100000开始的地方。之后先执行entry.S文件中的内容，在`kern/entry.S`中的最后调用`i386_init()`函数执行内核初始化操作，其中就包括初始化内存分配函数`mem_init()`函数，而这个函数是在`kern/pmap.c`中实现的，所以内存分配的任务只要是在`kern/pmap.c`文件中。

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

x86保护模式下内存管理架构是：分段和分页的（页转化的）。通过这种机制，可以将一个虚拟地址转化为物理地址。x86中的一个**虚拟地址**包括了一个段选择子（segment selector）和一个段内偏移（segment offset ）( a *virtual address* consists of a segment selector and an offset within the segment)。虚拟地址通过段地址转化机制之后就成了**线性地址**（Linear Address），线性地址再经过CPU的MMU部件的页式转化就得到了物理地址（Physical Address）（这个是在分页之后的，假如没有启动分页的话，得到的线性地址就是物理地址）。整个转化如下图所示： 

![](./image/virtual_linear_physical_translation0.jpg)

上述过程中更加详细的转化图如下所示：

![](./image/virtual_linear_physical_translation1.jpg)



### 段机制

对于段机制转化来说，GDT/LDT表（全局段描述符表/本地段描述符表）是存放关于某个运行在内存中的程序的分段信息的，比如某个程序的代码段是在哪里开始的，有多大；这个程序的数据段又是从哪开始的，有多大；这两个表是同类型的表，每一个表项都包括三个字段：

- Base：32位，代表这个程序的这个段的基地址

- Limit：20位，代表这个程序的这个段的大小

- Flags：12位，代表这个程序的这个段的访问权限

当给出selector和offset之后，首先根据selector在GDT/LDT表中找到这个段的段表项位置（selector代表这个段对应的段表项在GDT/LDT表中的位置），之后根据该段表项的Flags字段来判断是否可以访问这个段的内容，如果可以访问则把Base字段的内容取出，直接与offset相加，就得到了线性地址（linear address）。

> 需要注意的是我们编写的C语言程序中的指针的值其实是虚拟地址中段内偏移部分的值，这点在后面的编码中是相当重要的。

### 页机制

在接下去是页表机制的，开启分页之后，当处理碰到一个线性地址后，它的MMU部件会把这个地址分成3部分，分别是页目录索引（Directory）、页表索引（Table）和页内偏移（offset），这3个部分把原来32位的线性地址分成了10+10+12的3个片段。

再说明这3个片段的作用之前，先来看一下页目表和页表，页目录项的结构和页表项的结构基本一致，**页目表项的前20位为页表的物理页索引，用来定位页表的物理地址**；**页表项的前20位为页的物理索引，表示页的物理地址**。那么页目录和页表的低12位是用于一些标识和权限控制的，各个位含义如下：

- P —— Present，判断对应物理页面是否存在，存在为1，否则为0；

- W -Write，该位用来判断对所指向的物理页面是否可写，1可写，0不可写；
- U —— User，该位用来定义页面的访问者应该具备的权限。为1是User权限即可，如果为 0，表示需要高特权级才能访问；
- WT —— 1=Write-through，0=Write-back；
- CD —— Cache Disabled，1为禁用缓存，0不禁用；
- A —— Accessed，最近是否被访问；
- D —— Dirty，是否被修改；
- PS——Page Size（0=4KB，1=4MB）
- PAT——Page table attribute index
- G——Global Page
- AVL —— Available，可以被系统程序所使用；
- 0 —— 保留位；

说完线性地址、页目录和页表的结构之后，讲解一下地址转换的流程。在启用分页之前，页目录所在的物理页面的首地址需要存放在CR3寄存器中，当CPU进行页式地址转换时会从**CR3中取得页目录物理地址**，然后根据线性地址的高10位也就是页目录索引片段，取得页目录项，从取得的页目录项中取出高20位，这高20位即为页表物理页的首地址，再通过线性地址的中间10位即页表索引片段，取得页表项，页表项的高20位即为物理页起始地址，将该物理页起始地址与12位的页内偏移量进行拼接就得到了真正的物理地址。



> 1.根据页目录索引为10位，所以页目录项最多为1024（2^10），相当于有1024个页表；又因为页表索引为10位，所以页表项最多为1024（2^10），即一个页表可以表示1024个页。那么一个页目录项最多可以表示的页的个数为1M个，假如一个页的大小为4KB的话那么可以表示4GB的地址空间，因为页表项为4字节，所以页表大小为4KB，那么其实页表本身就是页。而通常我们说每个用户进程虚拟地址空间为4GB，其实就是每个进程都有一个页目录，进程运行时将页目录地址加载到CR3寄存器中，从而每个进程最大可以用4GB内存。但是在JOS中，为了简单起见，只用了一个页目录表，整个系统的线性地址空间4GB是被内核和用户程序所共用的。
>
> 2.分页管理中，页目录以及页表都存放在内存中，而由于CPU和内存速度的不匹配，这样子地址翻译时势必会降低系统的效率。为了提高地址翻译的速度，x86处理器引入了地址翻译缓存TLB（旁路转换缓冲）来缓存最近翻译过的地址，当然缓存之后也会引入缓存和内存中页表内容不一致的问题，可以通过重载CR3使整个TLB内容失效或者通过invlpg指令。
>
> 3.为什么CR3里面存着的是物理地址？
>
> 假如CR3里面存着的是线性地址，因为内容是真实存放在物理内存上的，所以我们还需要去找这个线性对应的物理地址，就可以得到物理地址，但是也是多次一举。所以我们可以这么想，因为我们需要得到最终的物理地址，所以在这个过程用到的都是物理地址的话，才能得到最终的物理地址。

### JOS中的初步实现

对于JOS操作系统来说，分段最早可以追溯到boot loader执行期间，因为在那时将CPU的运行模式从实模式切换到了保护模式，相应的我们也就可以看到分段的使用，可以看到在`boot/boot.S`中已经引入了GDT这么一个全局描述符表，这个表把所有的段的基址设置为0，界限设置为0xffffffff，这个其实相当于禁止了分段转化功能，因此这段选择子没有实际上的作用了，相当于虚拟地址的偏移量就是线性地址了。再接下来我们仔细看一下，GDT表的段表项的大小是8字节，那么找到某个段表项的位置，其实就是GDT表的起始地址+8*段表项索引。因为GDT表的第一项是要留出来的，所以GDT表的第一项（索引为0）为`SEG_NULL`；第二项为代码段`SEG(STA_X|STA_R, 0x0, 0xffffffff) # code seg`，我们可以看到代码段的段选择器被设置为了8`PROT_MODE_CSEG, 0x8 # kernel code segment selector`，这个正好是代码段的位置；同理数据段的段选择器被设置成了16，`PROT_MODE_DSEG, 0x10 # kernel data segment selector`，这也正好是数据段的位置。

```assembly
# file:boot/boot.S
.set PROT_MODE_CSEG, 0x8         # kernel code segment selector                                 
.set PROT_MODE_DSEG, 0x10        # kernel data segment selector                            
.set CR0_PE_ON,      0x1         # protected mode enable flag

# Switch from real to protected mode, using a bootstrap GDT
# and segment translation that makes virtual addresses 
# identical to their physical addresses, so that the 
# effective memory map does not change during the switch.
lgdt    gdtdesc
movl    %cr0, %eax
orl     $CR0_PE_ON, %eax
movl    %eax, %cr0

# Jump to next instruction, but in 32-bit code segment.
# Switches processor into 32-bit mode.
ljmp    $PROT_MODE_CSEG, $protcseg

.code32                     # Assemble for 32-bit mode
protcseg:
    # Set up the protected-mode data segment registers
    movw    $PROT_MODE_DSEG, %ax    # Our data segment selector
    movw    %ax, %ds                # -> DS: Data Segment
    movw    %ax, %es                # -> ES: Extra Segment
    movw    %ax, %fs                # -> FS
    movw    %ax, %gs                # -> GS
    movw    %ax, %ss                # -> SS: Stack Segment

# Bootstrap GDT
.p2align 2                                # force 4 byte alignment
gdt:
  SEG_NULL                              # null seg
  SEG(STA_X|STA_R, 0x0, 0xffffffff)     # code seg
  SEG(STA_W, 0x0, 0xffffffff)           # data seg

gdtdesc:
  .word   0x17                            # sizeof(gdt) - 1
  .long   gdt                             # address gdt
```

在启动分段机制到启动页表地址转化之前，这个期间得到的线性地址就是物理地址。那么启动页表地址转换机制最早是出现在`kern/entry.S`中。

```assembly
# file:kern/entry.S
# Load the physical address of entry_pgdir into cr3.  entry_pgdir
# is defined in entrypgdir.c.
movl    $(RELOC(entry_pgdir)), %eax
movl    %eax, %cr3
# Turn on paging.
movl    %cr0, %eax
orl     $(CR0_PE|CR0_PG|CR0_WP), %eax
movl    %eax, %cr0
```

在这段代码中，引入了`entry_pgdir`这个变量，这个变量指的其实就是我们上面所提到的页目录，定义在`kern/entrypgdir.c`中。

```c
// 下面可能会用到的宏定义，定义在inc/mmu.h中
// Page directory and page table constants.
#define PGSIZE          4096            // bytes mapped by a page
#define NPDENTRIES      1024            // page directory entries per page directory
#define NPTENTRIES      1024            // page table entries per page table
#define PDXSHIFT        22              // offset of PDX in a linear address

// 下面可能会用到的宏定义，定义在inc/memlayout.h中
// All physical memory mapped at this address
#define KERNBASE        0xF0000000

// 下面是简单列表的实现
pde_t entry_pgdir[NPDENTRIES] = {
        // Map VA's [0, 4MB) to PA's [0, 4MB)
        [0]
                = ((uintptr_t)entry_pgtable - KERNBASE) + PTE_P,
        // Map VA's [KERNBASE, KERNBASE+4MB) to PA's [0, 4MB)
        [KERNBASE>>PDXSHIFT]
                = ((uintptr_t)entry_pgtable - KERNBASE) + PTE_P + PTE_W
};

// Entry 0 of the page table maps to physical page 0, entry 1 to
// physical page 1, etc.
__attribute__((__aligned__(PGSIZE)))
pte_t entry_pgtable[NPTENTRIES] = {
        0x000000 | PTE_P | PTE_W,
        0x001000 | PTE_P | PTE_W,
        0x002000 | PTE_P | PTE_W,
        0x003000 | PTE_P | PTE_W,
        0x004000 | PTE_P | PTE_W,
        0x005000 | PTE_P | PTE_W,
        0x006000 | PTE_P | PTE_W,
        0x007000 | PTE_P | PTE_W,
        0x008000 | PTE_P | PTE_W,
        0x009000 | PTE_P | PTE_W,
        0x00a000 | PTE_P | PTE_W,
            	  .....
        0x3f9000 | PTE_P | PTE_W,
        0x3fa000 | PTE_P | PTE_W,
        0x3fb000 | PTE_P | PTE_W,
        0x3fc000 | PTE_P | PTE_W,
        0x3fd000 | PTE_P | PTE_W,
        0x3fe000 | PTE_P | PTE_W,
        0x3ff000 | PTE_P | PTE_W,
};
```

通过上述代码我们引入了一个简单的页表，可以使得内核程序可以跑在它所要求的链接地址上（虚拟地址）0xf0100000，当然内核程序真实的是跑在0x00100000上。但是这个页表只能映射4MB的内存，我们希望把这种映射扩展到头256MB的物理内存中，同时把256MB的物理内存映射到虚拟内存为0xf0000000开始的地方，或同时映射到其他的虚拟空间中。那么我们就来看看下面的实现。









