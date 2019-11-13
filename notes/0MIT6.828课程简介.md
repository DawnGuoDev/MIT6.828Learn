## 课程简介

Lab1主要讲的主要是PC启动的相关内容，如BIOS、boot loader，以及内核程序的第一个文件；

Lab2主要讲的主要是PC的内存管理，包括了物理内存分配和虚拟内存；

Lab3会讲到给qemu打补丁的事

## xv6与JOS

注意xv6与jos是不同的！！！

### xv6

xv6是一个已经给你写好的类UNIX内核，只是它相对来说会比UNIX简洁，因为这是一个用于教学的UNIX内核。homework中涉及到的就是xv6相关的，对应的书籍资料是xv6-books、xv6-resouce.

### JOS

JOS个人理解是我们根据Lab的内容来，自己一点点去实现的内核，个人认为JOS也是类UNIX内核，虽然在Lab3实现的environment的时候，Lab中提到environment是跟process不一样的，因为没有实现JOS没有实现一些UNIX的接口，但是这两种在概念上是类似的。而JOS的参考主要是Lab中的内容讲述以及i386参考手册。

Lab3中就会涉及到i386中手册学习。

## 材料汇总

1.  https://pdos.csail.mit.edu/6.828/2018/reference.html 

   这个链接里面包含了整个课程学习中都会用到的一些资料；

2.  https://pdos.csail.mit.edu/6.828/2018/xv6/book-rev11.pdf 

   这个PDF是xv6这个操作系统对应的书籍；

3.   https://pdos.csail.mit.edu/6.828/2018/xv6/xv6-rev11.pdf 

    这个PDF是xv6的源代码

4.   https://pdos.csail.mit.edu/6.828/2018/readings/i386/toc.htm 

    这个是Intel 80386的参考文档

5. 汇编语言的资料

6. Intel 80386 Reference Programmer's Manual

    https://pdos.csail.mit.edu/6.828/2018/readings/i386/toc.htm 



## 学习Tips

1. 在学习这个课程之前，我没学过汇编啥的，所以在学习Lab1的时候，处处碰壁，感觉难度上天了，所以个人建议对汇编语言先了解一些，比较辛苦的是，Lab1中也会要求你先熟悉一下汇编语言，也有给出相应的资料；

2. 对C语言编译的过程需要知道一下，如下所示（该图来自课程官方的slides）：

   ![](./image/0_0.jpg)

3. 对于Lab中有些不懂的问题，个人建议是先记录下来，因为你在后面的Labs的时候会发现这些问题都会迎刃而解；

4. 

## 参考的博客

1. https://www.cnblogs.com/fatsheep9146/p/5060292.html 

   这位大佬的博客只有前面几个Lab，但是每一个都是超详细的，我刚开始做的时候就是看这位大佬的。

2. https://github.com/shishujuan/mit6.828-2017/tree/master/docs 

   这位大佬的github是将每个Lab分成两部分，一部分是理论讲解，另一部分是exercise。

3.  https://www.jianshu.com/p/782cb80c7690 

   这位大佬做的实验记录，个人感觉更多是他自己梳理过了的，并不是完全按照Lab中的顺序来的，在总结上我觉得超级nice。

4.  https://blog.csdn.net/bysui/article/category/6232831 

   这位大佬的博客，主要是后面的几个Lab有参考。