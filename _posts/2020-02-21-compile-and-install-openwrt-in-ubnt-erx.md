---
filename: 2020-02-21-compile-and-install-openwrt-in-ubnt-erx
layout: post
title: "折腾之 ER-X 编译尝鲜 OpenWrt"
subtitle: ''
author: "yqsas"
header-style: text
catalog: true
tags:
  - 折腾

---

EdgeMax 虽然基于debian扩展性比较高，但是鉴于每次修改路由器配置的麻烦，还是刷成熟悉的OpenWrt，毕竟有NAS的情况下，路由器做好自己的网络管理就好了。

## 准备环境

1. 操作系统推荐 Ubuntu 18.04，避免编译依赖的问题。如果使用 win10，通过 wsl 安装也可具备环境，推荐使用这种方式。

1. 替换国内源

1. 安装所需的依赖

      ```bash
      sudo apt-get -y install git build-essential asciidoc binutils bzip2 gawk gettext git libncurses5-dev libz-dev patch python3 unzip zlib1g-dev lib32gcc1 libc6-dev-i386 subversion flex uglifyjs git-core gcc-multilib p7zip p7zip-full msmtp libssl-dev texinfo libglib2.0-dev xmlto qemu-utils upx libelf-dev autoconf automake libtool autopoint device-tree-compiler
      ```

1. 代理工具

    - 安装 proxychains4

        `sudo apt-get install proxychains4`

    - 配置 proxychains4.conf

        `sudo vim /etc/proxychains4.conf`

        `socks5 xxx.xxx.xxx.xxx 1080`

1. git 代理配置

    ```bash
    git config --global http.proxy 'socks5://xxx.xxx.xxx.xxx:1080'
    git config --global https.proxy 'socks5://xxx.xxx.xxx.xxx:1080'

    # 取消代理
    git config --global --unset http.proxy
    git config --global --unset https.proxy
    ```

## 编译

自己编译可以只有选择需要的插件和功能，对于内存吃紧的同学来说比较有用，不想折腾的同学可以直接使用官方编译的镜像。
OpenWrt 官方镜像地址：<https://downloads.openwrt.org/>

以下开始编译步骤：

1. `git clone https://github.com/coolsnowwolf/lede` 命令下载好源代码，然后 `cd lede` 进入目录

1. `./scripts/feeds update -a`
   `./scripts/feeds install -a`

1. 对固件功能进行配置
   `make menuconfig`
   - Target System 选择 `MediaTek Ralink MIPS`，Subtarget 选择`MT7621 based boards`，Target Profile 选择`EdgeRouter X`
   - 选择自用插件、主题等。 主题使用

1. 预下载编译过程所需工具

    - `proxychains4 make download` ；此处注意需要配置好 `proxychains4`
    - 如果编译过程中网络问题卡住，可手动下载依赖包到 dl 文件夹：<https://sources.openwrt.org/>。
    - 提供我编译过程总用到的依赖文件，解压后文件放在 `lede/dl` 路径下即可。需要的同学自取： 链接: [lede-dl.zip](https://pan.baidu.com/s/1jBgBXaSLkk8wM35lpMFpcw) 提取码: tb92。

2. 开始编译：`make -j1 V=s`（-j1 后面是线程数。第一次编译推荐用单线程，国内请尽量全局科学上网）即可开始编译你要的固件了。如果过程中因为网络问题卡住，也建议加上 `proxychains4` 代理

3. 编译完成，在 `lede/bin/targets/ramips/mt7621` 下找到编译生成的文件：`openwrt-ramips-mt7621-ubiquiti_edgerouterx-initramfs-kernel.bin`,`openwrt-ramips-mt7621-ubiquiti_edgerouterx-squashfs-sysupgrade.bin`

## 安装

编译结果从 19.x 版本后不支持生成 tar 版的内核镜像包，实际编译结果是 initramfs-kernel.bin ，据说是由于后续内核超过一定大小的原因。而 bin 类型的内核刷机较复杂，需要连接 TTL 线或者在 uboot 下刷入。
所以接下来我们使用一个只需 ssh 的简单的方式，就是在网上找一个 18.x 版本的内核 tar 包，刷入内核成功后，再用 `sysupgrade` 升级安装系统，`sysupgrade` 可以使用 `.bin` 文件。
> 18.x 的内核 tar 文件，可使用 [UBNT-ER-X 刷机教程-openwrt](https://www.right.com.cn/forum/thread-338384-1-1.html) 帖子中的附件。

1. 上传`ubnt-erx-squashfs-factory-initramfs.tar`到 ERX 原版系统的/tmp 下面

1. 用 ssh 登录进原版系统，输入命令`add system image openwrt-ramips-mt7621-ubnt-erx-squashfs-factory-initramfs.tar`

1. 输入 reboot 重启路由器

1. 用 ssh 登录路由器，网线连接路由器，默认没有密码，现在应该已经是 openwrt 了，上传 squashfs-sysupgrade.bin 到路由器 /tmp 目录里，
再运行 `sysupgrade squashfs-sysupgrade.bin` 完成后系统会自动重启，等待即可。

1. 再次连上路由器，就可以进入路由器了网页界面了
    > 默认地址：`http(s)://192.168.1.1`，默认用户`root`，默认密码 `password`。

## 参考链接

- <https://downloads.openwrt.org/>
- <https://github.com/coolsnowwolf/lede>
- <https://sources.openwrt.org/>
- [OpenWrt/LEDE 编译小记（斐讯 K2P）](https://www.jianshu.com/p/eed71e8a22cc)
- [UBNT-ER-X 刷机教程-openwrt](https://www.right.com.cn/forum/thread-338384-1-1.html)
