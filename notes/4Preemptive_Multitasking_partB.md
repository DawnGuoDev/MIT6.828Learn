## Part B: Copy-on-Write Fork

Unix提供`fork()`system call作为主要的进程创建原语。`fork()`system call 复制调用进程（the parent）的地址空间，然后去创建一个新的进程（the child）。xv6实现的`fork()`是将parent  pages的所有数据复制到child的pages，这跟我们上面的实现的`dumbfork()`很像，这种复制是整个`fork()`操作中消耗最大的。但是，在调用`fork()`之后在子进程中往往会立即调用`exec()`函数，`exec()`这个函数会将会把child的内存替换成一个新的程序（shell经常就是这么干的）。这样子一来花在复制上的时间就被大量浪费了，因为子进程在调用`exec()`之前将会使用很少一部分的内存。

出于这种原因，Unix的后续利用虚拟地址硬件的优势，允许parent和child共享映射到他们各自的地址空间的内存，直到父子进程中有一个准备修改。**这种技术就叫做copy-on-write**。为了实现这个，在`fork()`中kernel将地址空间的映射从父进程复制到子进程（复制的不是映射页的内容），同时将现在的共享页标记为read-only。当其中一个进程想要对其中一个共享页执行写操作，那么将会发生page fault。这时unix kernel意识到这个页是一个“virtual”copy或者“copy on write”copy，所以对于这个faulting process来说，它将会创建一个新的、私有的、可写的、复制的页。这种优化使得子进程中对于`fork()`之后`exec()`这种方式的开销变得少了：在`exce()`之前child将可能只需要复制一个页（stack的当前页）。

### User-level page fault handling



#### Setting the Page Fault Handler

首先是注册一个

为了处理自己的page faults，一个user environment将会使用`sys_env_set_pgfault_upcall` system call向JOS kernel 注册一个 page fault handler entrypoint。同时我们也在Env结构体中增加了一个新成员`env_pgfault_upcall`，这个成员变量用来记录这些信息。

#### Normal and Exception Stacks in User Environments

主要是介绍Normal和Exception stack

#### Invoking the User Page Fault Handler

