在这个作业中，我们将要探索如何使用pthread函数库中提供的condition 变量来实现barrier。barrier是程序中的一个点，在这个点上任何的线程都必须等待直到其他所有线程都到达了这个点。condition 变量是一种序列协调技术，类似于xv6的sleep和wakeup。

> **同步屏障**(**Barrier**)是并行计算中的一种同步方法。对于一群进程或线程，程序中的一个同步屏障意味着任何线程/进程执行到此后必须等待，直到所有线程/进程都到达此点才可继续执行下文。

## 实验准备

1. 下载 [barrier.c](https://pdos.csail.mit.edu/6.828/2018/homework/barrier.c) 文件到你的Linux机器上（不用xv6目录中）

## 实验初步了解

假如这个时候，对文件进行编译的话并运行的话，是会报错的，如下所示。

```bash
$ gcc -g -O2 -pthread barrier.c
$ ./a.out 2
Assertion failed: (i == t), function thread, file barrier.c, line 55.
```

`./a.out 2`中的2表示在barrier中同步运行的thread数。其实barrier.c文件跟我们之前的 [lock primitives](https://pdos.csail.mit.edu/6.828/2018/homework/lock.html) 这个作业里面用到的源文件很像。根据用户的输入创建相应的线程数，每一个线程都是一个循环，每一次循环都会调用`barrier()`函数，然后sleep一个随机的时间。

```c
static void *
thread(void *xa)
{
  long n = (long) xa;
  long delay;
  int i;

  for (i = 0; i < 20000; i++) {
    int t = bstate.round;
    assert (i == t);
    barrier();
    usleep(random() % 100);
  }
}
```

上述会报错的主要原因是`barrier()`函数，比如我们创建了两个线程A,B，假设A线程已经到达barrier，但是它在其他线程到达之前就离开了。那么假设线程B开始执行了，在`assert(i==t)`的时候，i==0，但是t==1，那么就报错了。

```c
static void 
barrier()
{
  bstate.round++;
}
```

所以我们需要的做的就是，当一个线程进入`barrier()`函数之后，它需要阻塞，直到所有的线程都进入了`barrier()`函数，这样子就不会报错了。

## 实验过程

在实现上述要求的时候，除了需要之前的 [lock](https://pdos.csail.mit.edu/6.828/2018/homework/lock.html) 这个作业中的函数、变量等

```c
pthread_mutex_t lock;     // declare a lock
pthread_mutex_init(&lock, NULL);   // initialize the lock
pthread_mutex_lock(&lock);  // acquire lock
pthread_mutex_unlock(&lock);  // release lock
```

同时也会使用下面这些pthread原语

```c
pthread_cond_wait(&cond, &mutex);  // go to sleep on cond, releasing lock mutex
pthread_cond_broadcast(&cond);     // wake up every thread sleeping on cond
```

> `pthread_cond_wait`在被调用的时候将会释放`mutex`，但是在返回之前又会重新获得`mutex`，这个函数的作用是sleep on cond；
>
> `pthread_cond_broadcast`wake up每一个sleep on cond的线程；

我们已经定义了`struct barrier`，并且使用`barrier_init()`函数对这个结构体的一些变量进行了初始化。同时需要注意以下两个问题:

-  `bstate.round` 记录当前的轮数，当每一轮开始的时候我们应该增加`bstate.round`
- 你必须处理这样一种情况：当一个线程在其他线程退出barrier之前绕着循环运行，特别是在从这一轮到下一轮重复使用`bstate.nthread`的时候。请确保一个线程离开barrier并且绕着循环运行不会增加bstate.ntheread，当前一个线程仍然在使用它的时候。

最终我们实现的`barrier()`函数如下所示：

```c
static void
barrier()
{ 
  pthread_mutex_lock(&bstate.barrier_mutex);
  bstate.nthread ++;
  if(nthread > bstate.nthread){
    pthread_cond_wait(&bstate.barrier_cond, &bstate.barrier_mutex);
  }else{
    bstate.round ++; 
    bstate.nthread = 0;
    pthread_cond_broadcast(&bstate.barrier_cond);
  }
  pthread_mutex_unlock(&bstate.barrier_mutex);
}
```

> 根据`pthread_cond_wait`和`pthread_cond_broadcast`，实现的大致思路是当一个线程进入barrier之后（表示这个线程到达了barrier），那么`bstate.nthread`需要+1，之后我们需要判断`bstate.nthread`和`nthread`，假如没有达到`nthread`，那么该线程需要等待也就是调用`pthread_cond_wait`，假如`bstate.nthread`达到了`nthread`，那么表示可以进行下一轮了，所以它设置好下一轮并且使用`pthread_cond_broadcast`唤醒其他sleep on cond的线程。那么为什么需要在开始的获得lock呢？简答来说的话在多线程涉及对全局变量进行操作的时候最好要有lock机制，并且考虑`pthread_cond_wait`函数中释放和获得lock，那么也需要lock。假如没有lock，假设A线程先进入了barrier，并运行到了`pthread_cond_wait`之前，之后CPU开始执行B线程，B线程进入进入barrier之后，将会继续运行（因为`bstate.nthread`==nthread），wake up了其他线程，然后开始了下一轮barrier，但是此时A线程还在前一轮barrier。那不就嗝屁了嘛。
>
> 个人觉得上述需要注意的第二问题是以下这种情况（请确保一个线程离开barrier并且绕着循环运行不会增加bstate.ntheread，当前一个线程仍然在使用它的时候）：
>
> ```
> static void barrier()
> { 
>   bstate.nthread ++;
>   if(nthread > bstate.nthread){
>     pthread_cond_wait(&bstate.barrier_cond, &bstate.barrier_mutex);
>   }else{
>   	pthread_cond_broadcast(&bstate.barrier_cond);
>     bstate.round ++; 
>     bstate.nthread = 0;
>   }
> }
> ```
>
> 这种情况下，假如A线程先是阻塞，B线程之后唤醒了A线程，结果因为没有lock，在B操作还需要使用`bstate.nthread`的时候，A线程又会对`bstate.nthread`进行++。这种情况自然是不可取的，这种情况同样在开头获得lock和末尾释放lock可以解决。

最终实验效果如下所示

```
root@share-virtual-machine:~/mit6.828/homework/hw9_barrier# ./a.out 2
OK; passed
root@share-virtual-machine:~/mit6.828/homework/hw9_barrier# ./a.out 4
OK; passed
root@share-virtual-machine:~/mit6.828/homework/hw9_barrier# ./a.out 6
OK; passed
```



