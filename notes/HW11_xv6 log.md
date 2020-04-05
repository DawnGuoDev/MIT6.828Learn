这个作业将会从两方面来探索xv6 log（也就是日志文件系统）。一方面，你将会人为的创建一个 crash 来阐述为什么需要 log。另一方面，你将会解除掉 xv6 logging system 中一个低效的问题。

## Creating a Problem

xv6 log 的目的是为了让文件系统中对磁盘更新的操作都是原子化的。举个例子，文件创建涉及到像 directory 中添加一个新的 entry 已经将这个新的文件的 inode 标记为 in-use。在没有 log 的情况下如果一个 crash 发生在添加新 entry 之前，但是在标记之后，那么将会导致文件系统在 reboot 之后处于一个不准确的状态。

下面的步骤将破坏 logging code ，导致一个文件被部分创建。

- first，用下面这段代码替换掉log.c中的 `commit()`函数

  ```c
  #include "mmu.h"
  #include "proc.h"
  void
  commit(void)
  {
    int pid = myproc()->pid;
    if (log.lh.n > 0) {
      write_log();
      write_head();
      if(pid > 1)            // AAA
        log.lh.block[0] = 0; // BBB
      install_trans();
      if(pid > 1)            // AAA
        panic("commit mimicking crash"); // CCC
      log.lh.n = 0; 
      write_head();
    }
  }
  ```

  BBB 这一行将会导致 log 中的 first block 被写入 zero，而不是它应该被写的内容。在文件的创建过程中，log 中的 first block 是新文件的 inode，应该被更新为 non-zero 类型的值。那么这样，BBB 这一行导致带有更新的 inode  的 block 被写为 0，接连导致 on-disk inode 仍然被标记为 unallocated。CCC这一行产生一个 crash。因为 init 过程也会创建文件，但是这个创建是在shell 开始之前，而我们想展示的是通过 shell 创建时出错，所以AAA 这一行让 init 过程不会产生buggy。

- second，使用下面这段代码来代替 log.c 中的 `recover_from_log()`函数

  ```c
  static void
  recover_from_log(void)
  {
    read_head();      
    cprintf("recovery: n=%d but ignoring\n", log.lh.n);
    // install_trans();
    log.lh.n = 0;
    // write_head();
  }
  ```

  这些修改将会阻止 log 的恢复，log 的恢复是指将会修复由 `commit()` 改变带来的 damage。

- finally，将 Makefile 文件中的 QEMUEXTRA 定义中 `-snapshot`选项移掉。这样子 disk image 才可以看见有所改变。

- 之后就是一系列的测试。先移掉 fs.img 然后运行 xv6，之后通过 xv6 的 shell 创建一个文件，如下所示，你将会看见来自 `commit()` 函数的panic。这看起来像是由于没有 logging system，在创建文件的过程发生了 crash。

  ![](./image/HW10_1.png)

  现在 re-start xv6，使用相同的 fs.img，通过 cat 查看文件的内容。你将会看			到如下内容。

  ![](./image/HW10_2.png)


## Solving the Problem

现在对 `recover_from_log()` 中的内容进行修改

```c
static void
recover_from_log(void)
{
  read_head();
  cprintf("recovery: n=%d\n", log.lh.n);
  install_trans();
  log.lh.n = 0;
  write_head();
}
```

之后运行 xv6 使用同样的 fs.img ，然后重新 cat 创建的文件。这次将会没有 crash。

> 为什么这次文件系统可以正常工作了？以及为什么即使你使用`echo hi > a`，这个文件还是空的？
>
> 个人认为，将 `recover_from_log()` 内容进行修改之后，那么恢复功能就相当于可以使用了，那么也就相当于logging system 可以正常使用了。即整个创建文件是原子性的了，那么 crash 发生在创建文件中间，那么 logging system 会让整个文件创建的操作都将被取消掉，所以读取文件时将会是空的。那么假如没有恢复功能或者说 logging system，文件创建过程中发生 crash ，那么创建过程中有一部分内容还是会被执行的，比如部分内容被写入。

**最后请记得将 `commit()`中的修改移掉，这样子 logging 才能正常工作，之后记得移掉 fs.img。**

## Streamlining Commit

假设文件系统想要更新 block 33 这块中的 inode。文件系统首先会调用 `bp=bread(block 33)` 将内容读取到 buffer 区，然后更新数据。 `commit()` 中的 `write_log()` 将会把数据拷贝到磁盘日志中的一个 block 上，比如 block 3。在 commit 的 `install_trans()` 中将会从 log 中读取 block 3（也就是 block 33 的内容），将内容复制到内存缓冲区中 block 33的位置，之后再把缓存区内容写入 disk 的 block 33。

但是，在 `install_trans()` 中，修改后的 block 33仍然保存在 buffer cache 中，那么就没有必要让 `install_trans()` 从 log 中读取 block 33，然后又复制到内存缓冲区中 block 33 的位置。那么下面则修改 `install_trans()` 函数，在 `commit()` 调用 `install_trans()` 的时候没有必要再从 log 中读取内容和复制内容。最终修改为如下所示

```c
static void
install_trans(void)
{
  int tail;

  for (tail = 0; tail < log.lh.n; tail++) {
    // struct buf *lbuf = bread(log.dev, log.start+tail+1); // read log block
    struct buf *dbuf = bread(log.dev, log.lh.block[tail]); // read dst
    // memmove(dbuf->data, lbuf->data, BSIZE);  // copy block to dst
    bwrite(dbuf);  // write dst to disk
    // brelse(lbuf);
    brelse(dbuf);
  }
}
```

