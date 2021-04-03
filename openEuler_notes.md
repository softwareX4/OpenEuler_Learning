# OpenEuler && LFS学习笔记
[TOC]
- [OpenEuler && LFS学习笔记](#openeuler----lfs----)
  * [环境](#环境)
  * [安装OpenEuler](#安装openeuler)
    + [配置静态IP](#配置静态ip)
      - [使用nmcli命令](#使用nmcli命令)
      - [使用ip命令](#使用ip命令-)
      - [通过ifcfg文件配置网络](#通过ifcfg文件配置网络)
  * [宿主机环境准备](#宿主机环境准备)
    + [宿主系统需求](#宿主系统需求)
    + [磁盘分区](#磁盘分区)
      - [创建新分区](#创建新分区)
      - [创建文件系统](#创建文件系统)
    + [软件包和补丁](#软件包和补丁)
    + [最后的准备工作](#最后的准备工作)
  * [构建LFS交叉⼯具链和临时⼯具](#构建lfs交叉⼯具链和临时⼯具)


## 环境
| 设备|版本 |
|:-:|:-:|
|主机操作系统|Win 10,64 bit|
|主机内存|8GB|
|虚拟机|VMware® Workstation 14 Pro|
|虚拟机操作系统|openEuler 20.03 LTS SP1|
|终端|MobaXterm v20.4|
## 安装OpenEuler
按照教程一步步来：[openEuler操作系统安装在vmware上](https://www.pianshen.com/article/15981091184/)
用MobaXterm连接的时候需要配置ip地址，用<code>ip addr</code>命令查看：

![](.img/ipaddr.png)

回到宿舍之后再启动虚拟机发现ip地址变了：

![](.img/ipnull.png)

说明现在ip地址是动态的，为了用ssh工具连接方便，需要改成**静态IP**。
### 配置静态IP
根据官方文档[配置网络](https://openeuler.org/zh/docs/20.03_LTS_SP1/docs/Administration/%E9%85%8D%E7%BD%AE%E7%BD%91%E7%BB%9C.html)，有三种方式：

- **使用nmcli命令**
**立即生效**且系统重启后配置也**不会丢失**。
- **使用ip命令**
立即生效但系统**重启后配置会丢失**。
- **通过ifcfg文件配置网络**
**不会立即生效**，需要在root权限下执行systemctl reload NetworkManager命令以重启网络服务后才生效。

> 注意：自己配的IP要跟主机在一个网段，否则虚拟机无法上网。。。


#### 使用nmcli命令
**立即生效**且系统重启后配置也**不会丢失**。
以下命令添加一个静态的IPv4网络连接：
```shell
nmcli connection add type ethernet con-name connection-name
 ifname interface-name ip4 address gw4 address
```
比如我要添加一个连接名为net-static的地址为192.168.0.254的连接：
```shell
nmcli con add type ethernet con-name net-static
 ifname enp3s0 ip4 192.168.0.10/24 gw4 192.168.0.254
```
![](.img/nmcliAdd.png)

第二行是新添加的net-static连接，还要使用<code>nmcli con up net-static ifname enp3s0</code>命令来激活新的连接。
查看连接详情：
```shell
nmcli -p con show net-static
```

![](.img/con-show.png)

删除一个连接使用<code>nmcli con delete [con-name]</code>

![](.img/conDel.png)

#### 使用ip命令
立即生效但系统**重启后配置会丢失**。
示例：
```c
# ip address add 192.168.0.10/24 dev enp3s0
```

#### 通过ifcfg文件配置网络
**不会立即生效**，需要在root权限下执行systemctl reload NetworkManager命令以重启网络服务后才生效。
1. 修改<code>/etc/sysconfig/network-scripts/ifcfg-ens33</code>:

![](.img/ifcfg.png)

2. 重启

同时我发现在使用nmcli命令配置IP后，在<code>/etc/sysconfig/network-scripts/</code>目录下出现了一个名为<code>ifcfg-net-static</code>的文件：

![](.img/ifcfg-netstatic.png)

查看内容：

![](.img/ifcfg-ns.png)

是和我们手动修改ifcfg文件添加的内容一样的，所以我猜想这两种方式本质上都是修改配置文件从而达到配置网络的目的。。。

## 宿主机环境准备
### 宿主系统需求
按照手册装完系统之后yum出错，很多包yum找不到对应软件包。
重新配yum源，修改yum源为华为云：
```sh
wget -O /etc/yum.repos.d/openEulerOS.repo https://repo.huaweicloud.com/repository/conf/openeuler_x86_64.repo

yum clean  all

yum makecache
```

![](.img/repo.png)



再次check：

![](.img/check0.png)

根据提示安装对应版本的软件包。

1. g++
```sh
   yum search "gcc-c++"
```

   ![](.img/yumSearch.png)

```sh
   yum install "gcc-c++.x86_64"
```
2. patch
```sh
   yum install patch
```
3. m4
```sh
   yum install m4
```
4. tar
```sh
   yum install tar
```
5. makeinfo
```sh
   yum install texinfo	
```
6. bision

由于yum找不到bision，手动wget安装bision到/usr/bin（[仓库](http://mirrors.ustc.edu.cn/gnu/bison/)）， 根据要求[下载2.7版本](https://blog.csdn.net/wch0lalala/article/details/105370314)

执行<code>sudo ./configure</code>出现“找不到命令”，<code>chmod +x  ./configure</code>增加可执行权限。

![](.img/configure.png)

make出错，根据提示信息安装缺的包：

![](.img/make.png)

```sh
yum install automake
```
又出现permission denied，chmod

![](.img/permission.png)


**在make过程中缺包就yum install，权限不够就chmod .**



装好需要的包之后，再次运行version-check。

![](.img/invalidOption.png)


这应该是由于我之前执行了yum install byacc。直接yacc --version 发现它就是指向bision的:

![](.img/yaccV.png)


<code>whereis yacc</code>，并根据手册的命令行单独分别执行:
```sh
/usr/bin/yacc --version | head -n1  
/usr/local/bin/yacc --version | head -n1
```

![](.img/yaccV2.png)


![](.img/whereisYacc.png)


发现yacc存在两个地方，在/usr/local/bin/yacc是指向bision的我们需要的yacc。

把之前装的byacc卸载
```sh
yum remove byacc
```
再次check :


![](.img/yaccNotFound.png)


出现yacc command not found.按理说安装好bision之后yacc会自动安装，这里的yacc应该只剩/usr/loacl/bin下的了。

根据手册上：

![](.img/bision.png)


直接把yacc复制到/usr/bin下：

![](.img/cp.png)


终于ok。

### 磁盘分区
#### 创建新分区
最小10GB系统区，2GB交换区。
虚拟机->设置->添加->硬盘->SCSI类型->创建新的虚拟磁盘

![](.img/disk/diskMem.png)

reboot
查看分区情况lsblk

![](.img/disk/lsblk.png)

发现新的磁盘命名为sdb.
接下来进行分区，cfdisk，并选择gpt标签。
```sh
cfdisk /dev/sdb
```

![](.img/disk/cfdisk0.png)

![](.img/disk/cfdisk1.png)


这里依次：新建->10GB->类型->Linux文件系统->写入；新建->2GB->类型->Linux swap->写入；

![](.img/disk/cfdisk3.png)

关于可选择的类型：

![](.img/disk/cfdisk2.png)

再次lsblk:

![](.img/disk/cfdisk4.png)

#### 创建文件系统
为sdb1创建ext4文件系统，并格式化sdb2为交换分区：
```sh
mkfs -v -t ext4 /dev/sdb1
mkswap /dev/sdb2
```

![](.img/disk/cfdisk5.png)

```sh
export LFS=/mnt/lfs
mkdir -pv $LFS
mount -v -t ext4 /dev/sdb1 $LFS
```
为了不用每次重启都重新挂载，修改 /etc/fstab，添加/dev/sdb1 /mnt/lfs ext4 defaults 1 1

![](.img/disk/cfdisk6.png)

启用swap
```sh
/sbin/swapon -v /dev/sdb2
```

### 软件包和补丁
创建目录并更改权限
```sh
mkdir -v $LFS/sources
chmod -v a+wt $LFS/sources
```

1. 从手册上的网址获取wget-list:
```sh
wget http://www.linuxfromscratch.org/lfs/downloads/stable/wget-list
```
获取软件包：
```
wget --input-file=wget-list --continue --directory-prefix=$LFS/sources
```

校验源码包的方法:
进入到下载源码的目录，在终端输入如下的命令：
```sh
md5sum -c md5sums
```
其中，第二个md5sums是通过如上地址下载到的。

2. 选择镜像站并下载软件包，用时34m59s
```sh
wget http://mirrors.ustc.edu.cn/lfs/lfs-packages/10.0/wget-list
wget --input-file=wget-list --continue --directory-prefix=$LFS/sources
wget http://mirrors.ustc.edu.cn/lfs/lfs-packages/10.0/check.sh
```
检查软件包
```sh
bash check.sh
```
> 用MD5校验和会出很多错，忽略掉。check没问题就好了

### 最后的准备工作
创建目录布局
```sh
mkdir -pv $LFS/{bin,etc,lib,sbin,usr,var}
case $(uname -m) in
 x86_64) mkdir -pv $LFS/lib64 ;;
esac
```
将交叉编译器和其他程序分离
```sh
mkdir -pv $LFS/tools
```
![](.img/disk/mkdir.png)

添加LFS用户
```sh
groupadd lfs
useradd -s /bin/bash -g lfs -m -k /dev/null lfs
```
将 lfs 设为 $LFS 中所有⽬录的所有者，使 lfs 对它们拥有完全访问权：
```sh
chown -v lfs $LFS/{usr,lib,var,etc,bin,sbin,tools}
case $(uname -m) in
 x86_64) chown -v lfs $LFS/lib64 ;;
esac
```
改变目录所有者
```sh
chown -v lfs $LFS/sources
```
![](.img/disk/user.png)


切换用户
```sh
su - lfs
```
配置环境；
新建⼀个除了 HOME, TERM 以及 PS1 外没有任何环境变量的 shell

```sh
cat > ~/.bash_profile << "EOF"
exec env -i HOME=$HOME TERM=$TERM PS1='\u:\w\$ ' /bin/bash
EOF
```
新的shell会读取并执行.bashrc：
```sh
cat > ~/.bashrc << "EOF"
set +h
umask 022
LFS=/mnt/lfs
LC_ALL=POSIX
LFS_TGT=$(uname -m)-lfs-linux-gnu
PATH=/usr/bin
if [ ! -L /bin ]; then PATH=/bin:$PATH; fi
PATH=$LFS/tools/bin:$PATH
export LFS LC_ALL LFS_TGT PATH
EOF
```
读取配置文件:
```sh
source ~/.bash_profile
```


## 构建LFS交叉⼯具链和临时⼯具
- 构建⼀个**交叉编译器**和与之相关的**库**
- 使⽤这个交叉⼯具链构建⼀些**⼯具**，构建⽅法保证它们和宿主系统**分离**
- 进⼊ **chroot** 环境，以进⼀步提⾼与宿主的隔离度，并构建剩余的，在构建最终的系统时必须的**⼯具**
![](.img/build/tip.png)

###  Binutils-2.35 - 第⼀遍
首先切到lfs身份，解压binutils包，进入目录。
```sh
su - lfs
tar -xvf binutils-2.35.tar.xz
cd binutils-2.35
```

编译
```sh
../configure --prefix=$LFS/tools \
 --with-sysroot=$LFS \
 --target=$LFS_TGT \
 --disable-nls \
 --disable-werror

 make

 make install
 ```

 ![](.img/build/binmake.png)
 ![](.img/build/binmakeinstall.png)
切换回包含所有源码包的目录并删除解压出来的目录。
```sh
cd ../../
rm -rf binutils-2.35
```
 ### GCC-10.2.0 - 第⼀遍
GCC 依赖于 GMP、MPFR 和 MPC 这三个包。
解压GCC，进入目录，分别解压三个包：
```sh
tar -xvf gcc-10.2.0.tar.xz
cd gcc-10.2.0

tar -xf ../mpfr-4.1.0.tar.xz
mv -v mpfr-4.1.0 mpfr
tar -xf ../gmp-6.2.0.tar.xz
mv -v gmp-6.2.0 gmp
tar -xf ../mpc-1.1.0.tar.gz
mv -v mpc-1.1.0 mpc
```

x86_64设置64位库默认目录:
```sh
case $(uname -m) in
 x86_64)
 sed -e '/m64=/s/lib64/lib/' \
 -i.orig gcc/config/i386/t-linux64
 ;;
esac
```
创建build目录
```sh
mkdir -v build
cd build
```

准备编译：
```sh
../configure \
 --target=$LFS_TGT \
 --prefix=$LFS/tools \
 --with-glibc-version=2.11 \
 --with-sysroot=$LFS \
 --with-newlib \
 --without-headers \
 --enable-initfini-array \
 --disable-nls \
 --disable-shared \
 --disable-multilib \
 --disable-decimal-float \
 --disable-threads \
 --disable-libatomic \
 --disable-libgomp \
 --disable-libquadmath \
 --disable-libssp \
 --disable-libvtv \
 --disable-libstdcxx \
 --enable-languages=c,c++
 ```

make

![](.img/build/gccmake.png)

make install

![](.img/build/gccmakeinstall.png)

创建完整版本的内部头文件：
```sh
cd ..
cat gcc/limitx.h gcc/glimits.h gcc/limity.h > \
 `dirname $($LFS_TGT-gcc -print-libgcc-file-name)`/install-tools/include/limits.h
```


### Linux-5.8.5 API 头⽂件
解压，进入目录，检查软件包中没有遗留的旧文件。
```sh
tar -xvf linux-5.8.3.tar.xz
cd linux-5.8.3
make mrproper
```
从源代码中提取用户可见的头文件。
```sh
make headers
find usr/include -name '.*' -delete
rm usr/include/Makefile
cp -rv usr/include $LFS/usr
```
删除目录。
### Glibc-2.32
解压，进入目录。
创建一个LSB兼容性符号链接
```sh
case $(uname -m) in
 i?86) ln -sfv ld-linux.so.2 $LFS/lib/ld-lsb.so.3
 ;;
 x86_64) ln -sfv ../lib/ld-linux-x86-64.so.2 $LFS/lib64
 ln -sfv ../lib/ld-linux-x86-64.so.2 $LFS/lib64/ld-lsb-x86-64.so.3
 ;;
esac

```

```sh
patch -Np1 -i ../glibc-2.32-fhs-1.patch
mkdir -v build
cd build

```

准备编译：
```sh
../configure \
 --prefix=/usr \
 --host=$LFS_TGT \
 --build=$(../scripts/config.guess) \
 --enable-kernel=3.2 \
 --with-headers=$LFS/usr/include \
 libc_cv_slibdir=/lib
 ```
 这里出错：These critical programs are missing or too old: bison
 检查 bison --version，出现bison:command not found
 检查 whereis bison，发现在/usr/bin下的bison不是需要的文件，真正的bison在/usr/local/bin下，于是复制过来，再次准备编译，成功。

 ```sh
 make
 make install
 ```


 完整性检查：
 ```sh
 echo 'int main(){}' > dummy.c
$LFS_TGT-gcc dummy.c
readelf -l a.out | grep '/ld-linux'
```

![](.img/build/check.png)

清理测试文件
```sh
rm -v dummy.c a.out
```


### GCC-10.2.0 中的 Libstdc++，第⼀遍
解压，进入目录，创建工作目录，准备编译
```sh
../libstdc++-v3/configure \
 --host=$LFS_TGT \
 --build=$(../config.guess) \
 --prefix=/usr \
 --disable-multilib \
 --disable-nls \
 --disable-libstdcxx-pch \
 --with-gxx-include-dir=/tools/$LFS_TGT/include/c++/10.2.0
 ```

 make 
 make DESTDIR=$LFS install

## 第 6 章 交叉编译临时⼯具

### M4-1.4.18
进⾏ glibc-2.28 要求的⼀些修补：
```sh
sed -i 's/IO_ftrylockfile/IO_EOF_SEEN/' lib/*.c
echo "#define _IO_IN_BACKUP 0x100" >> lib/stdio-impl.h
```
准备编译
```sh
./configure --prefix=/usr \
 --host=$LFS_TGT \
 --build=$(build-aux/config.guess)
```

make
make DESTDIR=$LFS install

### Ncurses-6.2
保证能找到gawk命令
```sh
sed -i s/mawk// configure
```
构建"tic"程序
```sh
mkdir build
pushd build
 ../configure
 make -C include
 make -C progs tic
popd
```

准备编译
```sh
./configure --prefix=/usr \
 --host=$LFS_TGT \
 --build=$(./config.guess) \
 --mandir=/usr/share/man \
 --with-manpage-format=normal \
 --with-shared \
 --without-debug \
 --without-ada \
 --without-normal \
 --enable-widec
 ```
编译安装
```sh
 make 
 make DESTDIR=$LFS TIC_PATH=$(pwd)/build/progs/tic install
echo "INPUT(-lncursesw)" > $LFS/usr/lib/libncurses.so
```
将共享库移动到/lib
```sh
mv -v $LFS/usr/lib/libncursesw.so.6* $LFS/lib
```
生成符号链接
```sh
ln -sfv ../../lib/$(readlink $LFS/usr/lib/libncursesw.so) $LFS/usr/lib/libncursesw.so
```

### Bash-5.0
准备编译
```sh
./configure --prefix=/usr \
 --build=$(support/config.guess) \
 --host=$LFS_TGT \
 --without-bash-malloc
 ```
 编译
 ```sh
 make
 ```
 出现错误

 ![](.img/build/lcurses.png)

 应该是缺少符号链接，建立一个：

```sh
 ln -sfv $(readlink $LFS/usr/lib/libncursesw.so) $LFS/usr/lib/libcurses.so
```

![](.img/build/ln.png)
成功。
安装
 ```sh
 make DESTDIR=$LFS install
 ```
 移动
```sh
mv $LFS/usr/bin/bash $LFS/bin/bash
```

 创建链接

```sh
ln -sv bash $LFS/bin/sh
```
> 下面只记录出现问题的地方，照着手册做就行


### File-5.39
解压出现错误
![](.img/build/gzip.png)
说明文件损坏了，重新下载一个。


## 进⼊ Chroot 并构建其他临时⼯具
以root身份执行
```sh
mkdir -pv $LFS/{dev,proc,sys,run}

mknod -m 600 $LFS/dev/console c 5 1
mknod -m 666 $LFS/dev/null c 1 3

mount -v --bind /dev $LFS/dev

mount -v --bind /dev/pts $LFS/dev/pts
mount -vt proc proc $LFS/proc
mount -vt sysfs sysfs $LFS/sys
mount -vt tmpfs tmpfs $LFS/run


if [ -h $LFS/dev/shm ]; then
 mkdir -pv $LFS/$(readlink $LFS/dev/shm)
fi


chroot "$LFS" /usr/bin/env -i \
 HOME=/root \
 TERM="$TERM" \
 PS1='(lfs chroot) \u:\w\$ ' \
 PATH=/bin:/usr/bin:/sbin:/usr/sbin \
 /bin/bash --login +h
```


退出chroot
```sh
exit
umount $LFS/dev{/pts,}
umount $LFS/{sys,proc,run}

```

创建一个test用户
```sh
echo "tester:x:$(ls -n $(tty) | cut -d" " -f3):101::/home/tester:/bin/bash" >> /etc/passwd
echo "tester:x:101:" >> /etc/group
install -o tester -d /home/tester
```


## 构建LFS系统
注意在chroot环境下。
安装软件包make check有时候会出错，但是install没有问题，暂且忽略。

### 清理系统
从现在起，在退出并重新进⼊ chroot 环境时，要使⽤下⾯的修改过的 chroot 命令：
```sh
mkdir -pv $LFS/{dev,proc,sys,run}

mknod -m 600 $LFS/dev/console c 5 1
mknod -m 666 $LFS/dev/null c 1 3

mount -v --bind /dev $LFS/dev

mount -v --bind /dev/pts $LFS/dev/pts
mount -vt proc proc $LFS/proc
mount -vt sysfs sysfs $LFS/sys
mount -vt tmpfs tmpfs $LFS/run


if [ -h $LFS/dev/shm ]; then
 mkdir -pv $LFS/$(readlink $LFS/dev/shm)
fi
chroot "$LFS" /usr/bin/env -i \
 HOME=/root TERM="$TERM" \
 PS1='(lfs chroot) \u:\w\$ ' \
 PATH=/bin:/usr/bin:/sbin:/usr/sbin \
 /bin/bash --login
```

### 系统配置
#### 引导linux系统
引导过程必须
- 挂载虚拟和真实⽂件系统
- 初始化设备
- 启⽤交换
- 检查⽂件系统完整性
- 挂载所有交换分区或⽂件
- 设定系统时钟
- 启⽤⽹络
- 启动系统需要的守护进程
- 并完成⽤⼾⾃定义的其他⼯作


#### SystemV
  - Unix 和 Linux 等类 Unix 系统中被⼴泛应⽤的经典引导过程。
  - 包含⼀个小程序 init，设定 login (通过 getty)并运⾏⼀个脚本。该脚本⼀般被命名为 rc，控制⼀组附加脚本的运⾏，这些附加脚本完成初始化系统需要的各项⼯作。
init 程序受到 <code>/etc/inittab</code> ⽂件的控制，被组织为用户可以选择的系统运⾏级别：
<pre lang="text">
<code>
0 — 停⽌运⾏
1 — 单⽤⼾模式
2 — 多⽤⼾模式，没有⽹络
3 — 完整的多⽤⼾模式
4 — ⽤⼾⾃定义模式
5 — 拥有显⽰管理器的完整多⽤⼾模式
6 — 重启系统
</code>
</pre>
通常的默认运⾏级别是 3 或 5。
- 优点
  - 完备的，已经被详细理解的系统。
  - 容易定制。
- 缺点
  - 引导速度较慢。⼀个中等速度的基本 LFS 系统从第⼀个内核消息开始，到出现登录提⽰符为⽌，需要8-12 秒的引导时间，之后还需要约 2 秒启动⽹络连接。
  - 串⾏执⾏引导任务，这与前⼀项缺点相关。引导过程中的延迟 (如⽂件系统检查) 会拖延整个引导过程。
  - 不⽀持控制组 (cgroups)、每用户公平共享调度等⾼级特性。
  - 添加脚本时，需要⼿动决定它在引导过程中的次序。

#### 设备和模块管理
- 传统的 Linux 系统
通常使⽤**静态**地创建设备，即在 /dev 下创建⼤量设备节点 (有时有数千个节点)，⽆论对应的硬件设备**是否真的存在**。
⼀般通过 MAKEDEV 脚本完成这⼀⼯作，它包含以相关的主设备号和次设备号，为世界上可能存在的每个设备建⽴节点的⼤量 mknod 命令。
- 使⽤ Udev
只有那些**被内核检测到的设备**才会获得为它们创建的设备节点。由于这些设备节点在每次引导系统时都会重新创建，它们被储存在 devtmpfs ⽂件系统中 (⼀个虚拟⽂件系统，完全驻留在系统内存)。

##### Sysfs
sysfs 的⼯作是将系统硬件配置信息导出给用户空间进程，用户空间可⻅的配置描述。
- 编译到**内核**中的驱动程序在它们的对象被内核检测到时，直接将它们注册到 sysfs (内部的 devtmpfs)。
- 被编译为**模块**的驱动程序，注册过程在模块**加载**时进⾏。
只要 sysfs ⽂件系统被挂载好 (位于 /sys)，用户空间程序即可使⽤驱动程序注册在 sysfs 中的数据，Udev就能够使⽤这些数据对设备进⾏处理 (包括修改设备节点)。


####  设备管理
```sh
bash /lib/udev/init-net-rules.sh
cat /etc/udev/rules.d/70-persistent-net.rules
```
没有70-persistent-net.rules这个文件。。

![](.img/build/70.png)


#### 一般网络配置
配置域名服务（用Google公用DNS服务IP地址）
```sh
cat > /etc/resolv.conf << "EOF"
 # Begin /etc/resolv.conf
 nameserver  8.8.8.8
 nameserver 8.8.4.4
 # End /etc/resolv.conf
 EOF
```

配置主机名
```sh
echo "lfs" > /etc/hostname
```

⾃定义 /etc/hosts ⽂件

```sh
cat > /etc/hosts << "EOF"
# Begin /etc/hosts
127.0.0.1 localhost.localdomain localhost
192.168.1.1 localhost.localdomain lfs
::1 localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
# End /etc/hosts
EOF
```

#### System V 引导脚本使⽤与配置

配置sysvinit
```sh
cat > /etc/inittab << "EOF"
# Begin /etc/inittab
id:3:initdefault:
si::sysinit:/etc/rc.d/init.d/rc S
l0:0:wait:/etc/rc.d/init.d/rc 0
l1:S1:wait:/etc/rc.d/init.d/rc 1
l2:2:wait:/etc/rc.d/init.d/rc 2
l3:3:wait:/etc/rc.d/init.d/rc 3
l4:4:wait:/etc/rc.d/init.d/rc 4
l5:5:wait:/etc/rc.d/init.d/rc 5
l6:6:wait:/etc/rc.d/init.d/rc 6
ca:12345:ctrlaltdel:/sbin/shutdown -t1 -a -r now
su:S016:once:/sbin/sulogin
1:2345:respawn:/sbin/agetty --noclear tty1 9600
2:2345:respawn:/sbin/agetty tty2 9600
3:2345:respawn:/sbin/agetty tty3 9600
4:2345:respawn:/sbin/agetty tty4 9600
5:2345:respawn:/sbin/agetty tty5 9600
6:2345:respawn:/sbin/agetty tty6 9600
# End /etc/inittab
EOF
```

#### 配置系统时钟
查看是否为UTC
```sh
hwclock --localtime --show
```

![](.img/build/utc.png)

```sh
cat > /etc/sysconfig/clock << "EOF"
# Begin /etc/sysconfig/clock
UTC=1
# Set this to any options you might need to give to hwclock,
# such as machine hardware clock type for Alphas.
CLOCKPARAMS=
# End /etc/sysconfig/clock
EOF
```

测试locale
```sh
LC_ALL=zh_CN.gb18030 locale charmap
LC_ALL=zh_CN.gb18030 locale language
LC_ALL=zh_CN.gb18030 locale int_curr_symbol
LC_ALL=zh_CN.gb18030 locale  int_prefix

```

![](.img/build/locale.png)

创建/etc/profile文件
```sh
cat > /etc/profile << "EOF"
# Begin /etc/profile
export LANG=zh_CN.gb18030
# End /etc/profile
EOF
```
创建/etc/inputrc文件：与手册相同
创建 /etc/shells ⽂件：与手册相同

### 使lfs系统可引导
查看分区情况

![](.img/build/lsblk.png)

```sh
cat > /etc/fstab << "EOF"
/dev/sdb1 / ext4 defaults 1 1
/dev/sdb2 swap swap pri=1 0 0
proc /proc proc nosuid,noexec,nodev 0 0
sysfs /sys sysfs nosuid,noexec,nodev 0 0
devpts /dev/pts devpts gid=5,mode=620 0 0
tmpfs /run tmpfs defaults 0 0
devtmpfs /dev devtmpfs mode=0755,nosuid 0 0
# End /etc/fstab
EOF

```




