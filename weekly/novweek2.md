### NOV week2

这周反正就是继续看题吧，更确切的是在 ddl 夹缝中求摸鱼，摸鱼中忙里偷闲看题。

#### 原来题目给的 libc 是有用的

libc.so.6 中有很多东西可以利用，但是可是本地和远程的 libc 版本可能不一样，导致本地的地址到远程对不上。可以用 patchELF 把本地要动态链接的 libc 换掉，参考： [更换目标程序libc](https://mambainveins.com/2021/08/21/2021/2021-08-21-pwn_patchelf/)。这样就可以使用 libc.so.6 中的函数、gadgets、字符串、在对应的 libc 下调试......

可以用 ldd 命令 (本质是个脚本) 查看文件依赖的共享库。(`readelf -d ezheap` 打印 .dynamic 段也可以)。

```bash
giacomo@ubuntu:~/Desktop/ctf/hnctf/ezheap$ ldd ./ezheap
    linux-vdso.so.1 (0x00007ffe66f48000)
    ./libc-2.23.so (0x00007f37b0d58000)
    /home/giacomo/Desktop/tools/glibc-all-in-one/libs/2.23-0ubuntu11.3_amd64/ld-2.23.so => /lib64/ld-linux-x86-64.so.2 (0x00007f37b112b000)
```

我被这个命令里的 `=>` 困扰了很久，才知道这个箭头不是指向的意思，而是：

> 在 ldd 命令打印的结果中，“=>”左边的表示该程序需要连接的共享库之 so 名称，右边表示由 Linux 的共享库系统找到的对应的共享库在文件系统中的具体位置

#### 粗略理解延迟绑定

上周在调 plt 是干什么的，没看懂到底怎么跳转的。

c 代码长这样：

```c
#include <stdio.h>
int main()
{ 
    printf("1st");
    printf("2nd");
     return 0; 
 }
```

在第一处 printf 下断点，就可以看见，连续 push 了 `0` 和 `__globle_offset_table` 之后，程序跳转到 `_dl_runtime_resolve_fxsave`。

![](https://s2.loli.net/2022/11/14/HRJkP6FwWb9qUeV.png)

在`_dl_runtime_resovle`函数中，`_dl_fixup()`函数用于解析导入函数的真实地址，并改写GOT。（这里尝试看了一下源码，但是结构体太复杂了 orz，于是先这样吧知道个原理就好，细节...再说吧。）

简单理解就是`_dl_fixup(table, idx)`，`table` 里面装着先对地址，`idx` 是函数在 table 中相应序号，这个例子里面 idx 为 0。根据 idx 在 table 里面找相对地址，再加上基址，填到 got 表里面。
