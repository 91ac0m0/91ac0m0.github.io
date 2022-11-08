---
title: ubantu16.04 pwn环境搭建
---

​		配环境花了整整两天时间，我感受到我的修为得到了极大的提升，境界得到新的程度的开拓，特此写一篇博客。



## 虚拟机安装

ubantu镜像下载[地址](https://mirrors.tuna.tsinghua.edu.cn/ubuntu-releases/16.04/)

## 配置

###  vmtools（如果联网会自动安装）

  ```
  sudo apt-get update
  sudo apt-get install open-vm-tools-desktop -y
  sudo reboot
  ```
### python，git，pip等工具

#### python3.6

参考博客[python3.6安装](https://blog.csdn.net/weixin_48125776/article/details/122475540)，这里先不用按照文中的安装pip，因为会下载pip8比较老

更改了默认py3版本之后，也把顺便默认的python从2.7改成3.6，参考博客[更改默认py的版本](https://blog.csdn.net/White_Idiot/article/details/78240298)

#### pip21

安装pip21.3.1[安装pip](https://blog.csdn.net/weixin_46039239/article/details/114694348)文中的命令为python3.7，需要使用如下命令。

```
curl https://bootstrap.pypa.io/pip/3.6/get-pip.py -o get-pip.py
```
更换pip源,这里换成清华源
```
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
```
####  git

git clone命令会把下载内容放到我们使用git clone时候的路径下面，所以可以建一个文件夹专门存放git下载的文件

```
sudo apt-get install git
```

### gdb重新编译和pwndbg的安装

由于gdb调用py不是调用python这个可执行文件，而是链接到gdb的libpython.so，版本不能更改只能重新编译gdb。我的ubantu上gdb的py版本是3.5，能够支持的pip8无法下载很多模块（py3.5不能被pip21支持），所以这里选择下载py3.6重新编译gdb。

- 查看gdb的python版本(如果是3.6就不用重新编译了)

```
(gdb) py
>import sys
>print(sys.version)
>end
3.4.0 (default, Apr 11 2014, 13:08:40)
[GCC 4.8.2]
```

- 卸载原来的gdb
```
sudo apt remove gdb
```

- 从官网下载gdb压缩包 [下载](https://mirrors.ustc.edu.cn/gnu/gdb/)
- 安装

解压：tar -zxf gdb-7.11.1.tar.gz

进入目录：cd gdb-7.11.1

```
mkdir build
cd    build
../configure  （此处不用添加参数，会用默认版本的python）
make        （时间稍久）
sudo apt-get install texinfo
sudo make install
```

此时在terminal输入gdb能出现gdb界面了。

- pwndbg下载

```
git clone https://github.com/pwndbg/pwndbg 
cd pwndbg 
（修改 requirements.txt）
./setup.sh
```

修改的requirements如下，有些版本pip21无法下载，所以改一下版本号

```
attrs==21.4.0
capstone==4.0.2
enum34==1.1.10
future==0.18.2
iniconfig==1.1.1
isort==5.10.1
packaging==21.3
pbr==5.9.0
pluggy==1.0.0
psutil==5.9.1
py==1.11.0
pycparser==2.21
pyelftools==0.28
Pygments==2.12.0
pyparsing==3.0.7
pytest==7.0.1
python-ptrace==0.9.8
ROPGadget==6.8
six==1.16.0
testresources==2.0.1
tomli==1.2.3
unicorn==2.0.0
```

### 其他安装

- sublime

```
cd /usr/lib/python3/dist-packages
sudo cp apt_pkg.cpython-35m-x86_64-linux-gnu.so apt_pkg.cpython-36m-x86_64-linux-gnu.so
sudo add-apt-repository ppa:webupd8team/sublime-text-3
sudo apt-get update
sudo apt-get install sublime-text-installer
```



## ps：

我也尝试过不要重新编译gdb，改一改pwndbg的代码说不定也可以用，但是卡在psutil这个模块上面了，就没有试下去，以下是我的尝试**！！！！没有成功！！！！**

- 更改setup.sh

```
# Find the Python version used by GDB.
-PYVER=$(gdb -batch -q --nx -ex 'pi import platform; print(".".join(platform.python_version_tuple()[:2]))')
+PYVER=3.6
PYTHON+=$(gdb -batch -q --nx -ex 'pi import sys; print(sys.executable)')
PYTHON+="${PYVER}"

# Find the Python site-packages that we need to use so that
# GDB can find the files once we've installed them.
if linux && [ -z "$INSTALLFLAGS" ]; then
    -SITE_PACKAGES=$(gdb -batch -q --nx -ex 'pi import site; print(site.getsitepackages()[0])')
    +SITE_PACKAGES=/usr/local/lib/python3.6/dist-packages
    INSTALLFLAGS="--target ${SITE_PACKAGES}"
```

有些模块版本安装不了，需要更改一下requirements.txt，修改内容如上。

成功安装，启动gdb报错如下，据说强制更新psutil有用，但是我没有搞出来。
