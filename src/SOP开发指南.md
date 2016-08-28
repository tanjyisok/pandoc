---
title: SOP开发指南
---

| 修订版本 | 负责人     | 日期        | 备注 |
|:---------|:-----------|:------------|:-----|
| 1.0      | 欧阳雄弈   | 2016.08.11  | 补全文档，并分离部分章节到其他文档 |
| 0.3      | 欧阳雄弈   | 2016.05.05  | 添加烧机说明 |
| 0.2      | 欧阳雄弈   | 2016.04.11  | 添加一些参考提交 |
| 0.1      | 欧阳雄弈   | 2016.03.30  | 创建初稿 |

# 术语
SOP
 :  SOHO OpenWrt Platform。为我司基于OpenWrt设计的产品化的路由软件平台。

LKM
 :  Linux Kernel Module

# 概述
SOP是基于 MTK OpenWrt SDK 的路由软件平台，本文面向此平台上的软件开发者，介绍了平台的一些基础开发知识、开发和使用方法以及平台开发规范。

对于平台的整体设计思想可以阅读[《SOP搭建方案》](./SOP搭建方案.html)一文。

# 一些值得阅读的文档
_注：我们对编译系统、系统工作流程进行了不少修改以适应我们的需求，所以官方文档的一些内容不能完全参考，特别是一些编译构建相关的内容，但对于了解OpenWrt系统而言，这些文档无疑仍是最有价值的）_

- [《OpenWrt manual》](http://downloads.openwrt.org/kamikaze/docs/openwrt.html)，通过此文：
    - 可以了解到如何配置一台OpenWrt系统的路由
    - 可以了解到如何开发OpenWrt系统
- [《OpenWrt Developer Guide》](https://wiki.openwrt.org/doc/playground/developer)，OpenWrt官方的开发指南

# 典型工作流程
当你已经有权限得到SOP源码后，便开启了你的SOP平台开发之旅，但是在此之前，你首先要确认自己的开发环境是否满足平台开发要求，跟着下面几个小节介绍的内容去实践，你应该就具备SOP的开发环境了

## 选择操作系统
统一开发环境，减少不必要的麻烦，请安装**64位**ubuntu14.04的操作系统

## 安装依赖工具

```bash
sudo apt-get install git-core build-essential subversion libncurses5-dev zlib1g-dev gawk flex libssl-dev gcc-multilib gettext libiconv-hook-dev zlib1g-dev libncurses5-dev bison flex sharutils
```

## 编译

```bash
source build/envsetup.sh

# 这时候需要选择编译机型，例如输入"K2"
K2

# 编译整机固件
build

#### 以下为一些其他常用的命令或者tips ####

# 查看其他开发相关命令
help

# build/logs 下保存了每次整机编译日志，用于调试和解决编译问题

# 需要编译其他机型时，可以使用下面的命令重新选择机型
chooseproduct

# 软件负责人通过下面的命令定制机型的特性
configproduct

# 编译单独的package或者内核
mmm netifd
mmm -c linux

# 修改不生效或者遇到一些莫名其妙的编译错误时，往往是因为你不理解SOP的编译流程导致，此时最简单的方式是编译时添加`-c`参数，这会在编译前清空中间产物，例如
build -c
mmm -c xxx

```

## 烧机

在烧机之前，你要知道SOP固件的Flash分区，示意图如下

```
+-----------------+ -----
|                 |   ^
|                 |   |
|   Bootloader    | 0x30000
|                 |   |
|                 |   v   ---> 固定192KB，作为bootloader
+-----------------+ -----
|                 |   ^
|                 |   |
|    Config       | 0x10000
|                 |   |
|                 |   v   ---> 固定64KB，保留
+-----------------+ -----
|                 |   ^
|                 |   |
|    Factory      | 0x10000
|                 |   |
|                 |   v   ---> 固定64KB，存放无线校准参数
+-----------------+ -----
|                 |   ^
|                 |   |
|    firmware     |  xxx
|                 |   |
|                 |   v   ---> 除掉前面几个已知的分区，Flash上剩余的空间全给此分区
+-----------------+ -----
```

firmware分区又被分为几部分，如下

```
+-----------------+ -----
|                 |   ^
|                 |   |
|  kernel image   |  xxx
|                 |   |
|                 |   v   ---> 大小不固定，根据编译时实际大小决定，存放linux内核镜像
+-----------------+ -----
|                 |   ^
|                 |   |
|     rootfs      |  xxx
|                 |   |
|                 |   v   ---> 大小不固定，根据编译时实际大小决定，属于文件系统的只读部分
+-----------------+ -----
|                 |   ^
|                 |   |
|   rootfs_data   |  xxx
|                 |   |
|                 |   v   ---> firmware分区剩余的部分全给此分区，属于文件系统的可读写部分
+-----------------+ -----
```

记不住分区的，在路由的终端下执行命令`cat /proc/mtd`可以查到

OK，现在回到编译。SOP源码编译生成的**分区镜像**会放在`<src_root>/bin/ramips/`，其中

- `openwrt-ramips-K2-u-boot.bin`为Bootloader分区
- `openwrt-ramips-mt7620-mt7620-squashfs-sysupgrade.bin`为firmware分区

> 【注】方案不同，编译产物名称可能稍有差异

我们需要更新的一般也就只有上面提到的两个分区

烧机方法是在开机启动阶段（bootloader刚起来），出现如下**操作提示**时，快速输入数字，并按照提示往下执行

```
Please choose the operation:
   1: Load system code to SDRAM via TFTP.
   2: Load system code then write to Flash via TFTP.
   3: Boot system code via Flash (default).
   4: Entr boot command line interface.
   6: Load all then write to Flash via TFTP.
   7: Load Boot Loader code then write to Flash via Serial.
   9: Load Boot Loader code then write to Flash via TFTP.
```

- 输入 “2” 即为烧写firmware
- 输入 “9” 即为烧写Bootloader

对于**正式固件的发布**而言，相关文件会放到`<src_root>/bin/ramips/image/`，这里面的文件与前面提到的分区镜像是不同的，其中：

- `SOP-ramips-mt7620-K2-local-debug.bin`为发布给用户的升级固件。文件内部有校验数据，为了防刷hW
- `SOP-ramips-mt7620-K2-local-debug.fac`为发布给工厂的编程器镜像，文件中的每个字节与Flash一一对应

## 开发调试
进入开发板有三种方法，但各自有一些前置条件，下面的章节分别进行介绍

### 通过串口
串口是比较通用的一种进入开发板终端的方式，通过串口通信不依赖网络，连接也十分稳定，但前提是需要引出开发板的UART口，一般成品样机由于已经装上壳体，就不太适用了。

硬件连接就绪后，使用工具即可进入开发板终端，Windows下一般使用SecureCRT，Linux下一般使用minicom

在此简单介绍minicom的配置方法：

1. 安装，`sudo apt-get install minicom`
2. 打开程序，`sudo minicom`
3. 一般第一次打开会自动进入配置页面，其他时候你也可以通过"CTRL-A+o"重新进入，你需要修改的
    - "Serial port setup" ->
        - "Bps/Par/Bits" 改为 "57600 8N1"
        - "Hardware Flow Control" 改为 "No"
    - "Save setup as dfl" 将刚才的修改设为默认配置
4. 重启minicom

### 通过ssh
debug类型固件[^固件类型]会预置ssh服务，可以通过ssh进入开发板的终端，默认的账户名和密码均为"root"

[^固件类型]: 固件可以区分不同的类型，例如debug类型的固件是面向我们内部的调试开发，加入了后门和调试工具，而release类型的固件则是面向用户，没有后门。了解详情可阅读[《SOP固件中的关键信息》](./SOP固件中的关键信息.html)

### 通过telnet
release类型固件也能使用，非常通用，但是我们的开发中很少使用，原因如下
- 由于既然用户的固件也能开启此后门，所以权限管控比较严格，激活此后门需要找相关人员获得相关工具
- 都是通过网络连入开发板终端，但不如ssh方便，ssh有文件传输协议scp，可以利用工具（Linux下使用scp命令，Windows下使用WinSCP）与开发板互传文件

# SOP软件框架特性
SOP基于OpenWrt，并保持OpenWrt的大部分框架特性，这里引用一句非常简短的话来概括这个框架：

> 万物即包子（**Package**），钩子（**Hook**）满天飞

**Package** 的含义是：OpenWrt把每个功能模块都划分为一个 Package，从host上的toolchain，到target上的bootloader、kernel、LKM、各种daemon、Web页面框架等。OpenWrt为每个 Package 提供一个Makefile以适配OpenWrt的编译系统，同时为每个 Package 提供源码补丁以适配OpenWrt嵌入式Linux系统，最后为 Package 提供OpenWrt的包管理系统。OpenWrt基于Buildroot编译框架，并结合这一系列的工作，将各种开源项目有机整合到一起，构建了一个强大的开源路由系统。

**Hook** 的含义是：OpenWrt无论在编译系统还是软件框架上都设置了非常多的钩子点（我想大部分思想实际都是发行版Linux上的延续），通过在这些钩子点上做特定的处理，得以让OpenWrt在一套软件上支持成百上千种机型，这些机型可以是不同的芯片方案，拥有不同的硬件规格。

# 重要目录及用途
> 【注】下文中“[ ]”表示可变的, "*"表示开发中可能经常接触的

```
├── phicomm_docs        --SOP文档
├── bin/
    ├── [ramips]/       --存放最终的固件
        ├── packages/   --存放编译生成的ipk
├── build/              --我司添加的目录，用于实现机型编译
    ├── logs/           --历史编译日志，个别时候对于排错很有帮助
    ├── products/       --存放所有机型配置的目录
        ├── [K2]/       --具体机型目录
├── build_dir/          --make时创建，所有源码最终被拷贝到这里进行编译，未打包的产物也在这里
    ├── target-[XXX]/  --略
        ├── *root-[ramips]  --最终根文件系统，基本等于登陆路由界面后执行"ls /"，下文统一称为rootfs
├── dl/                 --原始的源码压缩包会放到这里
├── scripts/            --编译系统或者开发会用到的脚本工具
    ├── phicomm/        --存放我司的脚本工具
├── include/            --↓
├── rules.mk            --↓
├── Makefile            --这三者基本就构成了OpenWrt的编译系统，了解编译流程、查找变量定义就应该在其中搜索
├── *package/           --所有package的源码以及配置文件都放到这里
    ├── phicomm/        --我司写的package
├── staging_dir/        --存放编译的中间件，例如某个package安装的头文件、so库都会放到这里供其他package编译使用
├── target/
    ├── generic         --基于OpenWrt的内核和系统文件修改，编译时覆盖编译路径下的文件（第一层覆盖）
    ├── [ramips]        --基于芯片方案和机型的内核和文件系统修改（第二层覆盖）
├── tmp/                --收集package的Makefile信息，构建编译依赖关系
```

# Package 的编译流程
对于功能开发而言，理解这一点十分重要。

OpenWrt的编译系统十分复杂，很多地方设计得非常好，这才能支撑它一套代码支持几十种路由厂商的上百款路由，但其中的一些设计对于我们开发而言比较冗余，会降低我们的开发效率，所以我们对这部分进行了修改。当前package的编译流程如下（以busybox为例）：

1. 主Makefile执行`packages/utils/busybox/`下的Makefile，获得一些变量的定义，使busybox这个开源项目能被OpenWrt的编译系统编译，而整个过程你并不需要修改busybox原工程携带的Makefile

2. 拷贝源码到编译路径，存在两种方式：
    2'.  将`dl/`目录下的`busybox-1.22.1.tar.bz2`解压获得源码，打上OpenWrt提供的patch（位于`package/utils/busybox/patches/`），最后将源码拷贝到`build_dir/target-mipsel_24kec+dsp_uClibc-0.9.33.2/busybox-1.22.1/`

    2''. 如果都以上面这种方式进行开发，可以发现我们是很难对源码进行修改和定制的，因为源码是从压缩包内获得，任何修改都是以patch的形式实现，不便于开发和代码评审，这是我们不能接受的。所以一旦我们需要对开源项目的package进行定制，**我们会将整个项目的代码解压出来入库**，具体会在["修改一个现有package"](#修改一个现有package)中介绍

3. 在`build_dir/target-mipsel_24kec+dsp_uClibc-0.9.33.2/busybox-1.22.1/`中编译

4. 生成产物。重点关注编译目录下的`ipkg-ramips_24kec`（决定哪些东西打包到ipk）和`ipkg-install`（决定哪些东西安装到rootfs）。目录中一般都包含有一些配置文件和脚本是服务于OpenWrt系统的，这些文件是从`packages/utils/busybox/[config|files]/`拷贝而来

5.  对于一个模块而言，编译属性是“m”代表可安装卸载的模块，会打包成ipk（ipk就是OpenWrt系统的安装包，位于`bin/ramips/packages/`，我们可以拷贝到开发板上使用opkg命令安装）；编译属性是“y”代表编入固件，既会默认预置在固件文件系统中，同时也会生成ipk放到`bin/ramips/packages/`（这一点对于功能开发而言简直**意义非凡**，我们开发调试单一功能时，不再需要每次耗费大量的时间重编固件和烧机）

# 模块开发Howto
## 开发一个新的package
引入新的**内核**或**应用**package都应该放在`package/phicomm`目录下，目录中包括Makefile及源码文件夹`src/`（如果有配置文件还应有`config/`；如果有脚本文件还应有`files/`），package Makefile的编写请参考"hello-looper"模块。Makefile的语法请查看 [OpenWrt manual][] “Creating packages ” “Creating kernel modules packages ”两节。

> 【注】部分我们实现的内核模块需要修改内核，例如`ip_hash_stat`，它们虽然也实现了LKM，但还对内核原有文件中进行了修改，这种情况下一般都添加了宏，会影响内核的编译。这类修改我们将其作为内核特性看待，而非简单的内核package，具体如何添加这类模块，请参考“[添加内核特性](#添加内核特性)”一节

## 修改一个旧的package
修改原OpenWrt的package时，如果发现对应package目录下没有`src/`目录，则说明其编译期有可能从互联网下载源码，并对源码打补丁。基于我们的开发模式，需要将源码压缩包解压放到对应package目录下的`src/`，并修改Makefile，让package编译时直接使用`src/`中的源码，这样更利于我们的开发，具体修改流程如下：

1. 解压`dl/`中对应源码压缩包，拷贝到对应package目录下命名为`src/`。

2. 如果package需要打patch（package目录下存在`patches/`目录），则在对应package路径下执行脚本：`<src_root>/scripts/phicomm/patch-src-with-commit.sh`，commit_msg.tmp会保存已打补丁列表，提交时请附上这些内容，方便后续维护。

3. 修改package下的Makefile，添加如下代码

    ```makefile
    define Build/Prepare
        mkdir -p $(PKG_BUILD_DIR)
        $(CP) ./src/* $(PKG_BUILD_DIR)/
    endef
    ```

    **特别注意： 上面示例代码的显示存在问题，这里的缩进一定要用TAB，不能用空格替代，以符合Makefile的语法，否则会编译失败。**

4. 此时做一个初始提交。这个提交的目的是让代码库中保留原始代码，有利于追溯。

    > 【参考提交】
    > 
    > ```
    > commit 005f80e60cf5109fa70a8eaa0ed2cd54d487456f
    > Author: Zhao Chong <chong.zhao@feixun.com.cn>
    > Date:   Thu Apr 7 15:07:13 2016 +0800
    > 
    >     iptables Package源码初始提交
    > 
    >     Summary:
    >     ./patches/002-layer7_2.22.patch
    >     ./patches/020-iptables-disable-modprobe.patch
    >     ./patches/030-no-libnfnetlink.patch
    >     ./patches/100-bash-location.patch
    >     ./patches/200-configurable_builtin.patch
    >     ./patches/300-musl_fixes.patch
    >     ./patches/400-lenient-restore.patch
    >     ./patches/500-add-xt_id-match.patch
    > 
    >     Reviewers: xiongyi.ouyang
    > 
    >     Reviewed By: xiongyi.ouyang
    > 
    >     Subscribers: #sop_group
    > 
    >     Differential Revision: http://172.17.48.129/D68
    > ```

5. 接下来则可以进行我们的修改，并生成的新的提交。*tips：执行`mmm [-c] <package_name>`即可单独编译package*

## 机型的特性定制
机型的特性定制被放到`build/products/<机型名称>`下统一管理。其中：

- `config`是机型的特性列表，构建时会拷贝到根目录。对任何特性的控制都应该尽量通过修改此文件来达成。

    > 参考流程，以`build/products/K2`为例:
    >
    > ```bash
    > cd <src_root> && source build/envsetup.sh
    >
    > # 输入"K2"
    >
    > configproduct
    >
    > # 此时进入TUI界面，完成你的配置并保存退出，此时修改会被自动保存到"build/products/<机型名称>/config"
    >
    > # 修改如果需要入库，需要提交 "build/products/<机型名称>/config" 的变化。
    > ```

## 实现新的特性
讲到了机型的特性定制，就要考虑如何实现可定制的特性，以及如何做好特性间的解耦。

### 添加功能特性

### 添加机型特性

### 添加内核特性
内核特性仍通过机型配置控制，特性的添加流程如下：

1. 修改`package/kernel/linux/modules/phicomm.mk`，添加模块定义
2. 在内核源码路径（`package/phicomm/linux/<内核源码目录>/`）修改代码，添加特性
3. 如果特性还包含独立的LKM，则将LKM放在`package/phicomm/linux/<内核源码目录>/phicomm/<模块名>/`下

>
> 【参考提交】
>
> ```
> commit 700449324ab1e6ba49e3815110d8f44cbc5601fb
> Author: Zhao Chong <chong.zhao@feixun.com.cn>
> Date:   Fri Apr 8 10:16:15 2016 +0800
> 
>     添加内核built-in模块ip_hash_stat
> 
>     Reviewers: xiongyi.ouyang
> 
>     Reviewed By: xiongyi.ouyang
> 
>     Differential Revision: http://172.17.48.129/D72
> ```

# SOP注意事项
- 任何时候都不能直接编辑修改机型目录（例如`build/products/K2`）下的`config`，开关特性请使用`configproduct`，这会自动帮你解决配置依赖并将修改添加到机型`config`中，更方便，也更可靠。
