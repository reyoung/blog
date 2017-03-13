# Proot, 非root用户下的chroot

在一些封闭的linux系统中，我们经常可能遇到没有root权限，但是需要安装软件包。或者需要将一个二进制包分发到**任何**的Linux系统上。如果该系统有docker的安装，那么显然我们使用docker会更好的完成这件事情。但是，国内很多地方的linux环境比较老，譬如kernel不能升到比较高的版本，可能还停留在2.6等系统上。亦或是有一些特殊的设备不支持比较新的linux kernel导致无法安装docker。这时候，我们需要一个更方便的方式去发布一个二进制包。

其实，我并不是很理解为什么CentOS会在运维界这么火，感觉单纯从技术的角度来看，CentOS似乎并不是一个持续维护与更新的系统。特别是CentOS 6这个版本已经在官方停止维护了。但显然在国内还很有市场。这就跟Windows XP或者IE6一样，我觉得实在是需要升级的系统了。特别是随着CoreOS之类的纯容器化操作系统的出现和流行，运维同事的工作应该可以大大的减少。技术进步的速度是不会因为人们的抱残守缺而减慢的，相反，抱残守缺的技术人员才会被时代淘汰。

所以，其实这篇文章估计在两三年后便会一点用都没有，我也希望这篇文章在两三年后一点用都没有。闲言少叙，下面说一下这个事情的来龙去脉。

## 动态库简介

[syscall](http://man7.org/linux/man-pages/man2/syscalls.2.html)是linux的系统内核和用户程序的最基本接口。用户所使用的用户接口，譬如 `printf`等函数内部，实际最终都会调用到`syscall`函数上。不同的Linux Kernel所提供`syscall`的功能不尽相同，又或是对于同一个`syscall`函数的实现有着不同的优化。

那么，既然Linux的所有用户程序，实际上都是调用到`syscall`来和kernel做交互，并且这个接口是基本**稳定**的，为什么linux下经常还会有[Dependency hell](https://en.wikipedia.org/wiki/Dependency_hell)呢？

这是因为，常见的Linux系统基本上都是鼓励动态库和动态链接的，然后是用一个更**聪明**的包管理器，去管理软件之间的依赖关系。譬如:

* 大部分的Linux系统的[libc](https://www.gnu.org/software/libc/)都是有一个全局的动态连接库，大家都去引用这个全局的libc.
* 软件包之间维护一个依赖关系，每个软件包之间都依赖于另一个软件包的特定版本。

例如，在linux下，查询openssl调用的动态库，便有如下这些。

```
root@5f3ed64ed3f1 ~# ldd (which openssl)
    linux-vdso.so.1 =>  (0x00007fff7b3fe000)
    libssl.so.1.0.0 => /lib/x86_64-linux-gnu/libssl.so.1.0.0 (0x00007fe3ed9a8000)
    libcrypto.so.1.0.0 => /lib/x86_64-linux-gnu/libcrypto.so.1.0.0 (0x00007fe3ed5cc000)
    libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007fe3ed206000)
    libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007fe3ed002000)
    /lib64/ld-linux-x86-64.so.2 (0x00007fe3edc1d000)
```

这样做的[好处](https://b.mtjm.eu/shared-libraries-advantages.html)有很多，常见的好处有:

* 这样节省硬盘空间。相对于静态连接，即每个用户程序均将其依赖的软件库中需要的函数直接链接进二进制库，在动态连接下，系统中只需要保存一个动态的`xxx.so`即可。
* 另一个好处，修复bug变得更简单。例如，如果软件 A依赖了库B，而库B依赖了库C。这时候发现了库C的实现上有一个bug，那么A和B都不需要修改，直接替换库C的二进制即可让整个系统跑起来。

这样做的问题也很明显，维护多个文件的依赖关系变得非常复杂。程序不再是单独复制一个二进制文件就可以在另一台机器上执行的，需要在多台机器上维护一样的二进制依赖。所以，需要使用各种`Toolchain`修改[rpath](https://en.wikipedia.org/wiki/Rpath)、各种`LD_LIBRARY_PATH`完成这个工作。

## Linux Chroot

root目录，即linux系统的根目录。一般常见的linux程序，会将自己的搜索动态库的路径，设置到一个以系统根目录的绝对路径。以之前的`openssl`举例，它依赖的动态库有 libssl.so，这个库的路径是`/lib/x86_64-linux-gnu/libssl.so.1.0.0`。

如果我们在CentOS里面将某一版本的ubuntu系统全部下载下来，解压缩到某一个路径里。例如解压缩到 `~/ubuntu/`里面，里面的所有二进制程序，还是依赖着root目录下的动态库。这时候，如果直接查找某一个文件依赖的动态库，便会报错。例如

```
$ ldd cmake
./cmake: /lib64/libc.so.6: version `GLIBC_2.15' not found (required by ./cmake)
./cmake: /lib64/libc.so.6: version `GLIBC_2.14' not found (required by ./cmake)
    linux-vdso.so.1 =>  (0x00007fff36beb000)
    libdl.so.2 => /lib64/libdl.so.2 (0x000000318aa00000)
    libstdc++.so.6 => /usr/lib64/libstdc++.so.6 (0x00007fc003607000)
    libm.so.6 => /lib64/libm.so.6 (0x000000318b600000)
    libgcc_s.so.1 => /lib64/libgcc_s.so.1 (0x000000318da00000)
    libc.so.6 => /lib64/libc.so.6 (0x000000318ae00000)
    /lib64/ld-linux-x86-64.so.2 (0x000000318a600000)
```

报错可能是因为找不到所需要的动态库，或者是找到的动态库版本不对。这时候，我们该怎么办呢？

通常的办法，这个二进制如果是我们自己写的代码，当然可以重新编译。并且将其所需要的动态链接库一起打包发行。但是，很多时候，重新编译比较麻烦，或者是，我们需要使用的不只是自己写的代码，而是另一个完整的发行版(例如在安装好CentOS的机器上运行整个ubuntu的程序)。

*nix系统为我们提供了一个简单的工具，[chroot](https://en.wikipedia.org/wiki/Chroot)来帮我们达到这个目的。所谓chroot，即change root，可以将某一个进程和他的子进程的root路径全部修改成一个当前目录树里子目录。也就是，对于两个进程，他们看到的目录结构是不一样的，chroot之后的进程看到的根目录是chroot之前进程看到的某一个子目录。

```
root@5f3ed64ed3f1 ~/p/rootfs# echo $PWD; md5sum $PWD/bin/bash
/root/proot/rootfs
164ebd6889588da166a52ca0d57b9004  /root/proot/rootfs/bin/bash
root@5f3ed64ed3f1 ~/p/rootfs# chroot . /bin/bash
root@5f3ed64ed3f1:/# md5sum /bin/bash
164ebd6889588da166a52ca0d57b9004  /bin/bash
root@5f3ed64ed3f1:/#
```

例如，当前路径是`/root/proot/rootfs`，我查询当前路径下的 `$PWD/bin/bash`的md5值是`164ebd6889588da166a52ca0d57b9004 `。调用`chroot`，将当前目录`.`变成新的root，并启动新root下的进程`/bin/bash`后，我们便切入进新的root路径了。

切入进新的root后，我们再计算`/bin/bash`的md5值，可见新的root下`/bin/bash`和之前的`/root/proot/rootfs/bin/bash`是同一个文件。

chroot之后，这个rootfs下的所有文件的动态库就搜索就正确了。

chroot非常好用，但是缺陷也比较明显:

* 无法限制资源使用。如果要限制资源使用，还是需要手写cgroups之类的东西。
* 必须使用root权限才能chroot

那么对于没有root权限的普通用户，怎么样在本地使用一个完整的rootfs呢？

## Proot -- 非root用户下的chroot

刚才提到，一般程序的动态库搜索路径，都是从根目录开始。我们之前使用chroot，是将根目录的位置进行修改。能不能在非root用户下，也有一个办法，让程序**显得** 根目录改变了呢？

答案是可以使用[proot](http://proot-me.github.io/)，proot本质上是使用了linux `ptrace` 这个接口，在运行时对程序行为进行一些修改。`ptrace`是linux进程的调试接口，`GDB`等调试器均是在linux下调用`ptrace`实现的。使用`ptrace`，可以在运行的时候修改打开文件等实现，动态的替换成某个新目录下的实现。比如用户程序里面写了`open("/bin/bash")`这个系统调用(这里只是举例子，我不确定`open`本身是不是系统调用，只是假设它是)，`PRoot`会使用`ptrance`将这个open函数重新定义到另一个函数中，这个函数将open里面的路径参数`/bin/bash`拿出来，转写成另一个参数，譬如`/home/xxx/ubuntu.rootfs/bin/bash`，然后再调用真正的`open`函数。

因为`ptrace`这个linux调用不需要root权限(否则用户模式下也没法gdb调试程序了，对吧)，使用`ptrace`在运行时修改用户程序行为，是不需要root权限的。类似于chroot的行为，也可以模拟出来。只是这种路径的转译是在运行时完成的，所以会损失一点性能(字符串替换的开销)。

简单的使用方法如下:

```bash
./proot  -R $PWD/rootfs  -0 /bin/bash
```

即将当前目录下的rootfs路径，变成子进程的root路径，并且在这个root路径里面使用root权限(uid=0)。启动的shell脚本是bash。

简单的用法和chroot类似，可以参考一下几个文章。

* [How to install nix in home](https://nixos.org/wiki/How_to_install_nix_in_home_(on_another_distribution)#PRoot_Installation) -- 如何在Home目录下安装另一个linux系统
* [Debian Noroot](https://play.google.com/store/apps/details?id=com.cuntubuntu) -- 在Android系统里安装一个Debian环境

不过proot还支持一些更有意思的功能。例如PRoot可以使用[QEMU](http://www.qemu-project.org/)来透明的执行为其他CPU编译的二进制。

更多的事情我自己也没有更深入的了解，只是先拿出来介绍一下，希望能够帮助上其他的人。

## 总结

顺便这里列一个表格来总结一下修改LD_LIBRARY_PATH, Docker, Chroot，Proot在解决Dependency Hell问题上的一些优缺点吧

|  | 支持资源控制 | 支持替换root目录 | 是否需要root权限 | 是否需要Linux Kernel版本 | 性能开销 | 备注 |
| --- | --- | ------ | ---  | --- | --- | --- |
| 修改LD\_LIBRARY_PATH | 否 |  否 | 否 | 否 | 无 | 修改LD\_LIBRARY_PATH来需要linux经验，有时甚至需要重新编译原始程序 |
| Docker | 是 | 是 | 启动dockerd需要root权限，但是使用docker不需要root | 需要 Kernel版本在3以上| 基本上没有 | |
| Chroot | 否 | 是 | 需要 | 否 | 基本上没有 | |
| PRoot | 否 | 是 | 否 | 否 | 运行时字符串替换，性能开销较小 | |

