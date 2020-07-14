---
layout: post
title: "译 | 揭秘 Docker 镜像"
subtitle: 'What is in a Docker image?'
author: "yqsas"
header-img: "img/in-post/whats-in-a-docker-image/docker-home.png"
catalog: true
tags:
  - Docker
  - 译
---

原文：[What's in a Docker image?](https://cameronlonsdale.com/2018/11/26/whats-in-a-docker-image/)

Docker 镜像里有什么？这是一个非常好的问题，在知道答案之前，Docker 镜像看起来似乎非常神秘。现在我不仅仅将告诉你答案，并且还会告诉你我是如何得到这个答案的。

---

## 从 Dockerfile 到镜像

在开始之前，我假设你对于 Dockerfile 已经非常熟悉：Docker 通过 Dockerfile 说明如何构建一个镜像。下面是一个例子。

```Dockerfile
FROM ubuntu:15.04
COPY app.py /app
CMD python /app/app.py
```

Dockerfile 里的每一行都是 Docker 如何构建镜像的说明。它将使用`ubuntu:15.04`作为基础，然后复制一个 python 脚本。`CMD`指令是关于运行容器时要做什么的指令（将镜像转换为正在运行的进程），暂时按下不表。

然后我们运行`docker build .` 并检查输出。

```bash
$ docker build -t my_test_image .
Sending build context to Docker daemon  364.2MB
Step 1/3 : FROM ubuntu:15.04
 ---> d1b55fd07600
Step 2/3 : COPY app.py /app/
 ---> 44ab3f1d4cd6
Step 3/3 : CMD python /app/app.py
 ---> Running in c037c981012e
Removing intermediate container c037c981012e
 ---> 174b1e992617
Successfully built 174b1e992617
Successfully tagged my_test_image:latest
```

首先看最后两行，我们已经成功构建了一个 Docker 镜像，可以通过标识符`174b1e992617`（这个值是镜像内容的 SHA256 哈希片段）找到对应镜像。

我们已经得到了最终镜像，但是控制台输出独立步骤的 ID 又代表了什么？`d1b55fd07600` 和`44ab3f1d4cd6`? 它们也是镜像吗？确实如此。如果我们从 Dockerfile 中删除第 2 步（`COPY app.py / app`），Docker 仍然可以成功构建它作为镜像。所以镜像构建过程中的每一步，我们都得到一个镜像。

这告诉我们镜像可以建立在彼此之上！当考虑`Dockerfile`中的`FROM`指令只是说明要在哪个镜像上构建时，这是有道理的。

一个镜像的结构必须以这样一种方式来组织，但是如何组织呢？接下来我们将把 docker 镜像拆开来看看。

---

## 导出镜像并解压缩

为了简化使用，镜像可以作为独立文件导出，这便于我们看到镜像里面的内容。

`docker save my_test_image > my_test_image`

查看导出的文件。

```bash
$ file my_test_image
my_test_image: POSIX tar archive
```

一个压缩包！我们解压看看。

```bash
$ mkdir unpacked_image
$ tar -xvf my_test_image -C unpacked_image
x 174b1e9926177b5dfd22981ddfab78629a9ce2f05412ccb1a4fa72f0db21197b.json
x 28441336175b9374d04ee75fdb974539e9b8cad8fec5bf0ff8cea6f8571d0114/
x 28441336175b9374d04ee75fdb974539e9b8cad8fec5bf0ff8cea6f8571d0114/VERSION
x 28441336175b9374d04ee75fdb974539e9b8cad8fec5bf0ff8cea6f8571d0114/json
x 28441336175b9374d04ee75fdb974539e9b8cad8fec5bf0ff8cea6f8571d0114/layer.tar
x 4631663ba627c9724cd701eff98381cb500d2c09ec78a8c58213f3225877198e/
x 4631663ba627c9724cd701eff98381cb500d2c09ec78a8c58213f3225877198e/VERSION
x 4631663ba627c9724cd701eff98381cb500d2c09ec78a8c58213f3225877198e/json
x 4631663ba627c9724cd701eff98381cb500d2c09ec78a8c58213f3225877198e/layer.tar
x 6c91b695f2ed98362f511f2490c16dae0dcf8119bcfe2fe9af50305e2173f373/
x 6c91b695f2ed98362f511f2490c16dae0dcf8119bcfe2fe9af50305e2173f373/VERSION
x 6c91b695f2ed98362f511f2490c16dae0dcf8119bcfe2fe9af50305e2173f373/json
x 6c91b695f2ed98362f511f2490c16dae0dcf8119bcfe2fe9af50305e2173f373/layer.tar
x c4f8838502da6456ebfcb3f755f8600d79552d1e30beea0ccc62c13a2556da9c/
x c4f8838502da6456ebfcb3f755f8600d79552d1e30beea0ccc62c13a2556da9c/VERSION
x c4f8838502da6456ebfcb3f755f8600d79552d1e30beea0ccc62c13a2556da9c/json
x c4f8838502da6456ebfcb3f755f8600d79552d1e30beea0ccc62c13a2556da9c/layer.tar
x cac0b96b79417d5163fbd402369f74e3fe4ff8223b655e0b603a8b570bcc76eb/
x cac0b96b79417d5163fbd402369f74e3fe4ff8223b655e0b603a8b570bcc76eb/VERSION
x cac0b96b79417d5163fbd402369f74e3fe4ff8223b655e0b603a8b570bcc76eb/json
x cac0b96b79417d5163fbd402369f74e3fe4ff8223b655e0b603a8b570bcc76eb/layer.tar
x manifest.json
x repositories
```

我们从`manifest.json`文件开始研究。

```json
[
  {
    "Config": "174b1e9926177b5dfd22981ddfab78629a9ce2f05412ccb1a4fa72f0db21197b.json",
    "RepoTags": [
      "my_test_image:latest"
    ],
    "Layers": [
      "cac0b96b79417d5163fbd402369f74e3fe4ff8223b655e0b603a8b570bcc76eb/layer.tar",
      "28441336175b9374d04ee75fdb974539e9b8cad8fec5bf0ff8cea6f8571d0114/layer.tar",
      "4631663ba627c9724cd701eff98381cb500d2c09ec78a8c58213f3225877198e/layer.tar",
      "c4f8838502da6456ebfcb3f755f8600d79552d1e30beea0ccc62c13a2556da9c/layer.tar",
      "6c91b695f2ed98362f511f2490c16dae0dcf8119bcfe2fe9af50305e2173f373/layer.tar"
    ]
  }
]
```

清单（manifest）文件是一段元数据，它准确描述了该镜像中的内容。我们可以看到该镜像被标记为`my_test_image`，并且还有一些叫做`Layers`和`Config`的内容（层与配置）。

JSON 配置文件的前 12 个字符正好与我们构建过程中看到的镜像 ID 相同，这可不是巧合哦。

```bash
$ cat 174b1e9926177b5dfd22981ddfab78629a9ce2f05412ccb1a4fa72f0db21197b.json
{
  "architecture": "amd64",
  "config": {
    "Hostname": "d2d404286fc4",
    "Domainname": "",
    "User": "",
    "AttachStdin": false,
    "AttachStdout": false,
    "AttachStderr": false,
    "Tty": false,
    "OpenStdin": false,
    "StdinOnce": false,
    "Env": [
      "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
    ],
    "Cmd": [
      "/bin/sh",
      "-c",
      "python /app/app.py"
    ],
    "ArgsEscaped": true,
    "Image": "sha256:44ab3f1d4cd69d84c9c67187b378b1d1322b5fddf4068c11e8b11856ced7efc0",
    "Volumes": null,
    "WorkingDir": "",
    "Entrypoint": null,
    "OnBuild": null,
    "Labels": null
  },
  "container": "c037c981012e8f03ac5466fcdda8f78a14fb9bb5ee517028c66915624a5616fa",
  "container_config": {
    "Hostname": "d2d404286fc4",
    "Domainname": "",
    "User": "",
    "AttachStdin": false,
    "AttachStdout": false,
    "AttachStderr": false,
    "Tty": false,
    "OpenStdin": false,
    "StdinOnce": false,
    "Env": [
      "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
    ],
    "Cmd": [
      "/bin/sh",
      "-c",
      "#(nop) ",
      "CMD [\"/bin/sh\" \"-c\" \"python /app/app.py\"]"
    ],
    "ArgsEscaped": true,
    "Image": "sha256:44ab3f1d4cd69d84c9c67187b378b1d1322b5fddf4068c11e8b11856ced7efc0",
    "Volumes": null,
    "WorkingDir": "",
    "Entrypoint": null,
    "OnBuild": null,
    "Labels": {}
  },
  "created": "2018-11-01T03:19:16.8517953Z",
  "docker_version": "18.09.0-ce-beta1",
  "history": [
    {
      "created": "2016-01-26T17:48:17.324409116Z",
      "created_by": "/bin/sh -c #(nop) ADD file:3f4708cf445dc1b537b8e9f400cb02bef84660811ecdb7c98930f68fee876ec4 in /"
    },
    {
      "created": "2016-01-26T17:48:31.377192721Z",
      "created_by": "/bin/sh -c echo '#!/bin/sh' > /usr/sbin/policy-rc.d \t&& echo 'exit 101' >> /usr/sbin/policy-rc.d \t&& chmod +x /usr/sbin/policy-rc.d \t\t&& dpkg-divert --local --rename --add /sbin/initctl \t&& cp -a /usr/sbin/policy-rc.d /sbin/initctl \t&& sed -i 's/^exit.*/exit 0/' /sbin/initctl \t\t&& echo 'force-unsafe-io' > /etc/dpkg/dpkg.cfg.d/docker-apt-speedup \t\t&& echo 'DPkg::Post-Invoke { \"rm -f /var/cache/apt/archives/*.deb /var/cache/apt/archives/partial/*.deb /var/cache/apt/*.bin || true\"; };' > /etc/apt/apt.conf.d/docker-clean \t&& echo 'APT::Update::Post-Invoke { \"rm -f /var/cache/apt/archives/*.deb /var/cache/apt/archives/partial/*.deb /var/cache/apt/*.bin || true\"; };' >> /etc/apt/apt.conf.d/docker-clean \t&& echo 'Dir::Cache::pkgcache \"\"; Dir::Cache::srcpkgcache \"\";' >> /etc/apt/apt.conf.d/docker-clean \t\t&& echo 'Acquire::Languages \"none\";' > /etc/apt/apt.conf.d/docker-no-languages \t\t&& echo 'Acquire::GzipIndexes \"true\"; Acquire::CompressionTypes::Order:: \"gz\";' > /etc/apt/apt.conf.d/docker-gzip-indexes"
    },
    {
      "created": "2016-01-26T17:48:33.59869621Z",
      "created_by": "/bin/sh -c sed -i 's/^#\\s*\\(deb.*universe\\)$/\\1/g' /etc/apt/sources.list"
    },
    {
      "created": "2016-01-26T17:48:34.465253028Z",
      "created_by": "/bin/sh -c #(nop) CMD [\"/bin/bash\"]"
    },
    {
      "created": "2018-11-01T03:19:16.4562755Z",
      "created_by": "/bin/sh -c #(nop) COPY file:8069dbb6bfc301562a8581e7bbe2b7675c2f96108903c0889d258cd1e11a12f6 in /app/ "
    },
    {
      "created": "2018-11-01T03:19:16.8517953Z",
      "created_by": "/bin/sh -c #(nop)  CMD [\"/bin/sh\" \"-c\" \"python /app/app.py\"]",
      "empty_layer": true
    }
  ],
  "os": "linux",
  "rootfs": {
    "type": "layers",
    "diff_ids": [
      "sha256:3cbe18655eb617bf6a146dbd75a63f33c191bf8c7761bd6a8d68d53549af334b",
      "sha256:84cc3d400b0d610447fbdea63436bad60fb8361493a32db380bd5c5a79f92ef4",
      "sha256:ed58a6b8d8d6a4e2ecb4da7d1bf17ae8006dac65917c6a050109ef0a5d7199e6",
      "sha256:5f70bf18a086007016e948b04aed3b82103a36bea41755b6cddfaf10ace3c6ef",
      "sha256:9720cebfd814895bf5dc4c1c55d54146719e2aaa06a458fece786bf590cea9d4"
    ]
  }
}
```

这是一个非常大的 JSON 文件，但通过这个文件你可以看到这里有很多不同的元数据。尤其重要的是，这关乎如何将此镜像转换为可以运行的容器的元数据（要运行的命令和要添加的环境变量）。

---

## 镜像如洋葱

镜像与洋葱都有很多层。但是什么是层？接下来我选取`cac0b96b79417d5163fbd402369f74e3fe4ff8223b655e0b603a8b570bcc76eb`一探究竟，因为这是『层』列表中的第一个。

```bash
$ ls cac0b96b79417d5163fbd402369f74e3fe4ff8223b655e0b603a8b570bcc76eb
VERSION   json      layer.tar
```

又有一个压缩文件 (tarfile)，继续解压并打开看看。

```bash
$ tree -L 1
.
├── bin
├── boot
├── dev
├── etc
├── home
├── lib
├── lib64
├── media
├── mnt
├── opt
├── proc
├── root
├── run
├── sbin
├── srv
├── sys
├── tmp
├── usr
└── var
```

这就是 Docker 镜像的最大秘密，它由文件系统的不同视图组成！这一层有很多内容，用户空间二进制文件在`/bin`, 共享函数库在`/usr/lib`，你可以在一个标准 Ubuntu 文件系统中看到的几乎一切内容。那么每个层究竟包含什么？通过它可以知道哪些层来自基本镜像，以及哪些层是由我们添加的。

使用我们之前做的相同的过程，但在`ubuntu:15.04`我可以看到这些层。

```bash
cac0b96b79417d5163fbd402369f74e3fe4ff8223b655e0b603a8b570bcc76eb
28441336175b9374d04ee75fdb974539e9b8cad8fec5bf0ff8cea6f8571d0114
4631663ba627c9724cd701eff98381cb500d2c09ec78a8c58213f3225877198e
c4f8838502da6456ebfcb3f755f8600d79552d1e30beea0ccc62c13a2556da9c
```

这些都属于 ubuntu 基本镜像，也就是来自`FROM ubuntu:15.04`命令。由此我们预测`my_test_image`图像的最顶层`6c91b695f2ed98362f511f2490c16dae0dcf8119bcfe2fe9af50305e2173f373`应该来自命令`COPY app.py / app /`。

```bash
$ tree
.
└── app
    └── app.py
```

果不其然，所有内容都是我们对文件系统所做的更改，仅仅添加了 app.py 文件。

### 视图展示

从视觉上看，我们的图像是这样的。

![layers](/img/in-post/whats-in-a-docker-image/layers.png)

### 加分回合

手动完成这项工作需要付出相当大的时间，但至少这样做一次是有益的。如果您希望将来分析镜像，可以使用开源工具 [dive](https://github.com/wagoodman/dive)。
![dive](/img/in-post/whats-in-a-docker-image/dive.png)

---

## 容器是如何运行的

现在我们已经理解了一个 Docker 镜像是什么，那么 Docker 是如何把它变成一个运行中的容器呢？

### 文件系统

每个容器都有它们自己的文件系统视图，Docker 获取镜像所有的层，并使它们层叠在一起，以呈现为文件系统的一个视图。这个技术称为 [Union Mounting](https://en.wikipedia.org/wiki/Union_mount)，Docker 支持 Linux 上的几个 Union Mount 文件系统，主要是 [OverlayFS](https://en.wikipedia.org/wiki/OverlayFS) 和 [AUFS](https://en.wikipedia.org/wiki/Aufs)。

但是这并非全部，容器是短暂的，容器运行时对文件系统的更改不应该在容器停止时保存。一种方法是将整个镜像复制到其他地方，这样更改不会影响原始文件。这不是非常高效，替代方法（以及 Docker 所做的）是在容器中的文件系统的最顶部添加一个瘦读/写层，进行更改。如果您需要对下面某个层中的文件进行更改，则需要将该文件复制到进行更改的顶层。这称为 [Copy-On-Write](https://en.wikipedia.org/wiki/Copy-on-write)。当容器停止运行时，将丢弃最顶层的文件系统层。

来自 [Docker 文档](https://docs.docker.com/v17.09/engine/userguide/storagedriver/imagesandcontainers/#images-and-layers) 的下图显示了最顶层。

![container-layers](/img/in-post/whats-in-a-docker-image/container-layers.jpg)

### 其他

启动容器的完整过程超出了本文的范围。在文件系统之后，镜像除了用于配置接下来的一些步骤的元数据之外，没有许多其他用途。完整而言，要创建一个正在运行的容器，我们需要使用命名空间 *[Namespaces](https://en.wikipedia.org/wiki/Linux_namespaces)* 来控制进程可以看到的内容（文件系统，进程，网络，用户等）；使用 *[cgroups](https://en.wikipedia.org/wiki/Cgroups)* 来控制进程可以使用的资源（内存，CPU，网络等）；安全功能来控制进程可以执行的操作（Capabilities, AppArmor, SELinux, Seccomp）。