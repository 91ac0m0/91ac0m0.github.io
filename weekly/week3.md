### NOV week3

#### srop

全称 Sigreturn Oriented Programming，用到了 sigreturn 系统调用 （rax = 15）。当程序在进行系统调用时，会把上下文环境和寄存器的数值压栈，完成系统信号处理之后会把这些数据返回给寄存器，sigreturn 就是这个作用。

在 sigreturn 之前需要把栈的内存布置好，pwntools 中的 SigreturnFrame 可以方便地完成这个工作。大致用法是：

```python
frame = SigreturnFrame()
frame.rax = rax
frame.rsp = addr
frame.rip = syscall
payload += bytes(frame)
```

除了设置存放 syscall 的参数的寄存器值，还要注意把 rip 值设为 syscall 地址，rsp 设为执行完 syscall 要 ret 的指令地址。有一个小技巧是，当布置 srop 链的时候可以调整一下每段 payload 的地址，比较方便算 rsp。

```python
payload = p64(rdi_ret_addr) + p64(15) +p64(syscall_plt) + framesigreturn(0, 0, flag_addr, 0x10, srop_addr + 0x120)
payload = payload.ljust(0x120, b'\x00')

payload += p64(rdi_ret_addr) + p64(15) +p64(syscall_plt) + framesigreturn(2, flag_addr, 0, 0, srop_addr + 0x240)
payload = payload.ljust(0x240, b'\x00')

# framesigreturn(rax, rdi, rsi, rdx, rsp)是自己定义的函数
```

#### tcache

[参考](https://blog.csdn.net/A951860555/article/details/115442780)
tcache 机制自从 glibc2.27 被引入。全称是 Thread Local Caching，它为每个线程创建一个缓存，里面包含了一些小堆块。目的是提升堆性能，但同时也舍弃了一些安全检查。

tcache 可以存放 64 个大小不同的单链表 bins，从 0x20 开始到 0x410 大小的都快都会首先存入 tcache ，每个大小 bins 中存放的堆上限为 7。先进后出，而且 prev_inuse 标记位都不会被清除，所以 tcache bin 中的 chunk 不会被合并，即使和 Top chunk 相邻。

程序第一次 malloc 时，会先 malloc 0x250 大小的堆块用于记录 tcache 的堆块信息。

#### unsortedbins

[参考](https://blog.csdn.net/qq_41202237/article/details/112589899)
unsorted 在 chunk 回到 bins 之前存放空闲 chunk。释放一个不属于 fast bin 的 chunk，并且该 chunk `不与top chunk相邻`，该 chunk 会被首先放到 Unsorted bin 中。

#### 其他

发现了一个讲的很清晰的群论[blog](https://chenliang.org/2021/02/26/group-theory/)，救我于信安数基课本。

其实也不用一定用题目给的 libc ，glibc-all-in-one 有符号表。

关于scikit-learn 写的很清楚的简单[例子](https://blog.csdn.net/qq_44971458/article/details/107092324)。

下周看看 stack smash。
