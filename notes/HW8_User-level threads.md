在这个homework中，我们通过实现线程之间的上下文切换的代码来完成一个简单的user-level线程包。

## 实验准备

1. 下载 [uthread.c](https://pdos.csail.mit.edu/6.828/2018/homework/uthread.c) and [uthread_switch.S](https://pdos.csail.mit.edu/6.828/2018/homework/uthread_switch.S)这两个文件放到xv6的目录中（Hint:`uthread_switch.S `是以S结尾的而不是s结尾的）

2. 在Makefile文件中的`_forktest`规则后面添加上下面内容（Hint：每一行开始的空白的地方是tab不是space）

   ```makefile
   _uthread: uthread.o uthread_switch.o
   	$(LD) $(LDFLAGS) -N -e main -Ttext 0 -o _uthread uthread.o uthread_switch.o $(ULIB)
   	$(OBJDUMP) -S _uthread > uthread.asm
   ```

3. 在Makefile文件中`UPROGS`后面添加上`_uthread`，（UPROGS定义user programs的列表）

> 完成上述步骤之后，使用`make qemu`运行xv6之后，在xv6 shell中运行`uthread`，xv6 kernel将会打印错误信息，表示`uthread`这个程序发生了page fault

## 实验文件的初步了解

我们来看一下`uthread.c`这个文件，`uthread.c`它自己定义了一个“线程”结构体类型：thread结构体有一个自定义的stack，一个stack pointer（指向了thread stack），以及线程的状态标志符。

```c
struct thread {
  int        sp;                /* saved stack pointer */
  char stack[STACK_SIZE];       /* the thread's stack */
  int        state;             /* FREE, RUNNING, RUNNABLE */
};
```

同时全局变量`current_thread`和`next_thread`都会指向这个结构体，那么C编译器对`struct thread`这个结构体的布局是按照下面这种方式来的。不太严谨的说法是，有点类似栈的味道，越是上面的结构体成员是越是放在下面。

```c
    --------------------
    | 4 bytes for state|
    --------------------
    | stack size bytes |
    | for stack        |
    --------------------
    | 4 bytes for sp   |
    --------------------  <--- current_thread
         ......

         ......
    --------------------
    | 4 bytes for state|
    --------------------
    | stack size bytes |
    | for stack        |
    --------------------
    | 4 bytes for sp   |
    --------------------  <--- next_thread
```

之后来瞅一瞅`main()`函数，`main()`函数首先是调用`thread_init()`函数，该函数是把main函数开始也当成了一个线程（这个线程是上述定义的结构体），之后创建了两个线程，创建的过程会对每个线程的stack进行初始化。最后调用`thread_schedule()`函数。

```c
int
main(int argc, char *argv[])
{
  thread_init();
  thread_create(mythread);
  thread_create(mythread);
  thread_schedule();
  return 0;
}
```

下面我们来看一下`thread_schedule()`函数，这个函数主要是调度另一个线程来运行，其中` thread_switch()`函数是我们这个homework中在`uthread_switch.S`这个文件中要实现的。它的作用就是去运行另一个线程。

```c
static void
thread_schedule(void)
{
  thread_p t;

  /* Find another runnable thread. */
  next_thread = 0;
  for (t = all_thread; t < all_thread + MAX_THREAD; t++) {
    if (t->state == RUNNABLE && t != current_thread) {
      next_thread = t;
      break;
    }
  }

  if (t >= all_thread + MAX_THREAD && current_thread->state == RUNNABLE) {
    /* The current thread is the only runnable thread; run it. */
    next_thread = current_thread;
  }

  if (next_thread == 0) {
    printf(2, "thread_schedule: no runnable threads\n");
    exit();
  }

  if (current_thread != next_thread) {         /* switch threads?  */
    next_thread->state = RUNNING;
    thread_switch();
  } else
    next_thread = 0;
}
```

同时我们可以看到两个线程执行的函数，在每次执行完`printf()`之后，就会调用` thread_schedule()`函数，这个函数最终调用的也是`thread_schedule()`函数。

```c
static void
mythread(void)
{
  int i;
  printf(1, "my thread running\n");
  for (i = 0; i < 100; i++) {
    printf(1, "my thread 0x%x\n", (int) current_thread);
    thread_yield();
  }
  printf(1, "my thread: exit\n");
  current_thread->state = FREE;
  thread_schedule();
}
```

所以整个实验的效果实现的效果是两个线程来回切换，每一个线程打印完"my thread ..."之后就会切换到另一个线程执行，最终效果类似下面这样的。

```
...
cpu0: starting
sb: size 1000 nblocks 941 ninodes 200 nlog 30 logstart 2 inodestart 32 bmap start 58
init: starting sh
$ uthread
my thread running
my thread 0x2D68
my thread running
my thread 0x4D70
my thread 0x2D68
my thread 0x4D70
my thread 0x2D68
my thread 0x4D70
my thread 0x2D68
...
```

## 实验过程

从上面的分析可知我们要实现的其实就是 `thread_switch.S`这个文件的内容。这个uthread_switch主要是保存当前线程的状态到`current_thread`所指的结构体中，之后恢复`next_thread`的状态，然后让`current_thread`指向`next_thread`所指的，这样当uthread_switch返回之后，next_thread就是current_thread，状态是running。对于保存和恢复所有八个x86寄存器的值，我们可以使用汇编指令`pushal`和`popal`。

同时我们可以看到在`thread_create()`函数中，有模仿8个寄存器被压入线程新栈的操作`t->sp -= 32`，那么这段空间存放的就是寄存器的值。因为创建的线程的是会被调度的，那么被调度的时候，是需要把保存的寄存器的值全都恢复。

```c
void
thread_create(void (*func)())
{ 
  thread_p t;
  
  for (t = all_thread; t < all_thread + MAX_THREAD; t++) {
    if (t->state == FREE) break;
  }
  t->sp = (int) (t->stack + STACK_SIZE);   // set sp to the top of the stack
  t->sp -= 4;                              // space for return address
  * (int *) (t->sp) = (int)func;           // push return address on stack
  t->sp -= 32;                             // space for registers that thread_switch expects
  t->state = RUNNABLE;
}
```

那么综上所述，其实该程序一共有三个线程，一个是main线程，另外两个是创建的线程，但是main线程用的栈其实是进程的user stack，而创建的线程用的是结构体中指定的栈。

下面是整个`thread_switch.S`的实现，根据上述要求，我们首先是需要保存当前寄存器的值，之后保存esp的值，然后恢复esp的值，并把所保存的寄存器的值全都恢复。

> Hint:假如你想对current_thread所指的sp成员进行写操作，那么可以使用下面这些汇编语言
>
> ```assembly
> movl current_thread, %eax
> movl %esp, (%eax)
> ```
>
> 通过上述操作之后，我们就将%esp寄存器的值保存在了 `current_thread->sp`中了，这个主要是因为sp相当结构体的偏移量是0（可以看上述讲到的结构体在内存中的布局）。

```assembly
	text
/* Switch from current_thread to next_thread. Make next_thread
 * the current_thread, and set next_thread to 0.
 * Use eax as a temporary register; it is caller saved.
 */
  .globl thread_switch
thread_switch:
  /* YOUR CODE HERE */
  // save the current thead state
  pushal
  movl  current_thread, %eax
  movl  %esp, (%eax)

  // restore next_thread
  movl  next_thread, %eax
  movl  %eax, current_thread
  movl  (%eax), %esp
  popal

  movl $0, next_thread

  ret       /* pop return address from stack */
```

我们按照整个程序执行来讲述，首先运行的是main这个线程，首先是把main这个线程的所有状态保存下来（使用`pushal`指令），之后是把当前%esp寄存器的值保存到main线程结构体的sp成员变量中。之后我们将`current_thread`指向了`next_thread`，并把`next_thread`中保存的sp的值恢复到了%esp寄存器中。之后弹出所有已保存的寄存器的值。记得把next_thread置0，最后返回，那么执行的指令就是线程函数的第一条指令了。

下面我们使用gdb来测试实验的结果，在两个窗口输入`make qemu-nox-gdb`和`make gdb`，然后在gdb窗口依次输入`symbol-file _uthread`、` b thread_switch`(打断点)和`c`。如下所示：

```
(gdb) symbol-file _uthread
Load new symbol table from "_uthread"? (y or n) y
Reading symbols from _uthread...done.
(gdb) b thread_switch
Breakpoint 1 at 0x268: file uthread_switch.S, line 11.
(gdb) c
Continuing.

```

上述操作，Continuing会一直等待，这是因为使用`c`命令之后，xv6系统会一直运行到shell，然后等待我们输入，所以在shell窗口输入`uthread`，gdb窗口输入如下所示（可以看到`pushal`哦）

```
(gdb) c
Continuing.

[  1b: 268]    0x418 <stat+8>:  push   $0x0

Breakpoint 1, thread_switch () at uthread_switch.S:11
11        pushal
(gdb) 
Continuing.
[  1b: 268]    0x418 <stat+8>:  push   $0x0

Breakpoint 1, thread_switch () at uthread_switch.S:11
11        pushal
```

下面我们输入`p/x next_thread->sp`、`x/9x next_thread->sp`来查看相关内容

```
(gdb) p/x next_thread->sp
$1 = 0x6d10
(gdb) x/9x next_thread->sp
0x6d10 <all_thread+24560>:      0x00000000      0x00000000      0x00000000      0x00000000
0x6d20 <all_thread+24576>:      0x00000000      0x00000000      0x00000000      0x00000000
0x6d30 <all_thread+24592>:      0x00000170
```

0x00000170是线程执行函数的地址，栈顶的元素是0，可以看到有8个0，这个相当8个寄存器的值。



