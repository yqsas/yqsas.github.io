---
layout: post
title: "折腾之 Manjaro 安装使用指北"
subtitle: ''
author: "yqsas"
header-img: "img/in-post/manjaro-deepin/manjaro-home.jpg"
catalog: true
tags:
  - 折腾
  - manjaro
  - linux
---

## 一、前言

记录一下使用 manjaro 的过程，一方面备忘，另一方面希望可以帮助到需要的人，内容持续更新。

## 二、安装

安装过程简单说，Manjaro 安装非常简单，基本上开箱即用，和其他系统区别不大。

1. 下载 ISO 镜像，[官网地址](https://manjaro.org/download/)
2. 制作 U 盘启动盘，强烈推荐 [Ventoy](https://github.com/ventoy/Ventoy/releases)
3. U 盘启动安装界面，时区设置为 Asia/Shanghai，语言选择 zh_CN，driver 选择 nonfree。选择 Boot 启动安装；
4. 进入桌面（此时系统还在 U 盘中，可以体验桌面效果），双击桌面 Install Manjaro Linux 进入系统安装，然后一路按自己需求 next；
5. 双系统情况下，注意分区选择自定义分区，然后分配 Manjaro 系统分区挂在为/（内存大不需要 swap 分区，一般使用不需要给/home 独立分区，后期调整系统大小也方便），启动分区选择已存在的 EFI system partition，挂在为/boot/efi，并选择保留（默认选项）。
6. 安装完成后重启系统即可。

## 三、基本配置

### 3.1 软件包管理配置

  ```bash
#国内源，可多次尝试选择速度最快的
sudo pacman-mirrors -i -c China -m rank
#系统更新
sudo pacman -Syyu
# 配置 archlinux 软件仓库
sudo vim /etc/pacman.conf
  ```

  ```bash
  [archlinuxcn]
  Server = https://mirrors.cloud.tencent.com/archlinuxcn/$arch
  ```

  ```bash
  sudo pacman -Sy archlinuxcn-keyring
  # AUR 包管理工具
  sudo pacman -S yay
  # 必备工具
  sudo pacman -S git vim net-tools base-devel
  # 解决双系统时间不同步问题
  timedatectl set-local-rtc true
  ```

### 3.2 zsh/oh-my-zsh

  ```bash
  sudo pacman -S zsh
  # curl 使用代理: curl -x "127.0.0.1:7890"
  sh -c "$(curl -fsSL https://raw.github.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
  # chsh -s /bin/zsh
  ```

  ```bash
  # 必备插件安装
  git clone https://github.com/zsh-users/zsh-completions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-completions

  git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting

  git clone https://github.com/zsh-users/zsh-autosuggestions.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions

  vim ~/.zshrc
  # edit plugins & save
  plugins=(git zsh-syntax-highlighting docker docker-compose zsh-autosuggestions zsh-completions)

  autoload -U compinit && compinit
  ```

### 3.3 中文输入法

使用fcitx5即可。

1. 配置使用 fcitx 输入法

    ```bash
    sudo vim /etc/profile
    ```

    ```bash
    export GTK_IM_MODULE=fcitx5
    export QT_IM_MODULE=fcitx5
    export XMODIFIERS=@im=fcitx5
    ```

1. 安装输入法

    ```bash
    sudo pacman -S fcitx5 fcitx5-qt fcitx5-gtk fcitx5-configtool fcitx5-gtk fcitx5-qt
    #拼音输入法
    sudo pacman -S fcitx5-chinese-addons fcitx5-pinyin-zhwiki
    #主题
    sudo pacman -S fcitx5-pinyin-zhwiki fcitx5-material-color
    ```
    > 主题设置：前往 Fcitx5设置 -> 配置附加组件 -> 经典用户界面 -> 主题


### 3.4 必备字体安装

  ```bash
  sudo pacman -S wqy-bitmapfont wqy-microhei \
  wqy-zenhei adobe-source-code-pro-fonts \
  adobe-source-sans-pro-fonts adobe-source-serif-pro-fonts \
  adobe-source-han-sans-cn-fonts ttf-monaco ttf-dejavu ttf-hanazono \
  noto-fonts noto-fonts-cjk noto-fonts-emoji
  ```

### 3.5 双显卡配置

  1. 更换默认驱动

      ```bash
      sudo pacman -S virtualgl lib32-virtualgl lib32-primus primus

      # 查看当前已安装驱动
      mhwd -li
      # 卸载不需要的驱动
      sudo mhwd -r pci video-hybrid-intel-nvidia-418xx-bumblebee
      # 安装最新闭源驱动
      sudo mhwd-tui # 选择4

      sudo systemctl enable bumblebeed
      ```

  2. 用户组设置

      ```bash
      #添加 bumblebee 用户组
      sudo groupadd bumblebee
      #添加当前用户到该组
      sudo gpasswd -a $USER bumblebee
      ```

  3. 重启电脑

### 3.6 VPN 客户端配置

> 系统默认未安装客户端，需要先行安装客户端支持。

  1. L2TP 客户端安装

      ```bash
      # 安装 strongswan  原因：need strongswan (or libreswan) for IPSec support.
      sudo pacman -S strongswan
      # 安装 l2tp 依赖
      yay -S libnm-git
      # 安装 l2tp 客户端，先选 2 一遍安装一些依赖，再选 1 一遍编译安装
      yay -S networkmanager-l2tp
      ```

  2. 控制命令

      ```bash
      # 获取所有 vpn 连接列表
      nmcli con show
      # 连接一个标记为已断开的网络接口
      nmcli con up <uuid/name>
      # 断开一个网络连接
      nmcli con down <uuid/name>
      # 添加内网路由
      sudo route add -net 10.0.0.0 netmask 255.0.0.0 gw <gateway ip>
      ```

## 四、开发环境

### 4.1 Docker

  ```bash
  sudo pacman -S docker docker-compose

  # 设置普通用户使用 Docker 不需要使用 sudo
  sudo groupadd docker
  sudo usermod -aG docker $USER
  ```

### 4.2 IDE/编辑器

  ```bash
  # snap
  sudo pacman -S snapd
  sudo systemctl enable --now snapd.socket
  sudo ln -s /var/lib/snapd/snap /snap
  # IDEA
  sudo snap install intellij-idea-ultimate --classic
  # rider
  sudo snap install rider --classic
  # vscode
  sudo pacman -S vscode
  # datagrip 数据库管理
  yay -S datagrip
  sudo pacman -S mysql-workbench
  ```

### 4.3 Java 环境

  ```bash
  sudo pacman -S maven
  ```

### 4.4 Nodejs 环境

  ```bash
  sudo pacman -S nodejs npm
  ```

### 4.5 Ruby+Jekyll

  ```bash
  # Ruby
  sudo pacman -S ruby
  gem install jekyll bundler
  #项目依赖安装：bundle install/update
  ```

### 4.6 其他

  ```bash
  # pip
  yay -S python-pip
  ```

## 五、软件推荐

```bash
# 日常
  sudo pacman -S google-chrome

  sudo pacman -S netease-cloud-music
  sudo pacman -S filezilla  # FTP/SFTP

  sudo pacman -S virtualbox  virtualbox-guest-dkms # 选择当前内核对应版本

  sudo pacman -S goldendict # 翻译、取词
    # 不推荐有道词典 高分屏坐标偏移，屏幕取词不便
    # [英汉字典下载](https://github.com/skywind3000/ECDICT/releases)

  # 多平台笔记应用，替代印象笔记
  sudo pacman -S joplin

  sudo pacman -S linuxqq             #qq
  sudo pacman -S deepin-wine-wechat  # 微信

# 开发
  sudo pacman -S tmux

# 办公
  #字体切记采用这种方式安装
  sudo pacman -S ttf-wps-fonts wps-office wps-office-mui-zh-cn

# 装 X
  sudo pacman -S neofetch
    #配合食用：neofetch --ascii_distro arch
  sudo pacman -S screenfetch
    #配合食用：screenfetch -A 'Arch Linux'

# 其他
  sudo pacman -S light # 命令调节亮度
  sudo pacman -S guake # 下拉终端，同类：tilda
  sudo pacman -S sshpass # 指定密码登录ssh： sshpass -p passwd ssh user@xx.xx.xx.xx

```

## 六、桌面环境

> 2019.12 更新： manjaro 官网 Deepin 环境已经消失，不建议继续使用

目前使用过 KDE、Gnome、Deepin 三种桌面环境。KDE 界面特效多、自定义程度高，Gnome 简洁直观，建议大家按需选择。

1. 安装 KDE

    ```bash
    sudo pacman -S plasma kio-extras
    sudo pacman -S kdebase

    sudo pacman -S manjaro-kde-settings sddm-breath-theme manjaro-settings-manager-knotifier manjaro-settings-manager-kcm

    sudo systemctl enable sddm.service --force
    ```

## 七、其他记录

### 7.1 Hidpi 配置

  1. xorg 配置缩放
      > 参考文档： [HiDPI （简体中文）](https://wiki.archlinux.org/index.php/HiDPI_（简体中文）)
      - 在 设置 --> 设备 --> 显示器中，缩放设置为 200%（缩放倍率可以按整数倍率调整，但是容易出现 1 倍太小，2 倍太大的情况）
      - 利用 xrandr 微调合适倍率： `xrandr --output eDP1 --scale 1.2x1.2`
      - 命令启动时执行： `vim /etc/profile`，在文件最后添加上一条命令内容

  1. chrome

      ```bash
      sudo vim /usr/share/applications/google-chrome.desktop
      ```

      ```vim
      # 这一行修改
      Exec=/usr/bin/google-chrome-stable %U
      # 添加启动命令参数
      Exec=/usr/bin/google-chrome-stable --force-device-scale-factor=1.7 %U
      ```

  1. wine 安装的软件

      ```bash
      WINEPREFIX=~/.deepinwine/<你的 APP 目录> wine winecfg
      #或者
      WINEPREFIX=~/.deepinwine/<你的 APP 目录> deepin-wine winecfg
      ```

### 7.2 科学上网

  1. 和windows一样的GUI版本：
  ```
  yay -S clash-for-windows-bin
  #启动
  sudo cfw  --no-sandbox
  ```

  2. 使用 docker 启动 clash 的方式，由 clash 进行全局代理，根据自定义规则选择上网方式。
      1. 启动 clash 容器
      准备 docker-compose 文件：`vim ~/.config/clash/docker-compose.yml`

       ```yaml
       version: '3'
       services:
         clash:
           image: dreamacro/clash
           volumes:
             - ~/.config/clash/config.yaml:/root/.config/clash/config.yaml
             # dashboard volume
             # - ./ui:/ui
           ports:
             - "7890:7890"
             - "7891:7891"
             # If you need external controller, you can export this port.
             # - "9090:9090"
           restart: always
           # When your system is Linux, you can use `network_mode: "host"` directly.
           network_mode: "host"
           container_name: clash
       ```
      启动： `docker-compose -f ~/.config/clash/docker-compose.yml up -d`
  4. 准备配置文件

      `vim ~/.config/clash/config.yaml`

      配置文件模板请参考：[Hackl0us/SS-Rule-Snippet/clash.yaml](https://github.com/Hackl0us/SS-Rule-Snippet/blob/master/LAZY_RULES/clash.yaml)

      根据模板，添加代理服务器配置，策略组需要增加一个名称为`Proxy`的配置。

  1. 配置全局代理

      进入系统设置 --> 网络 --> 网络代理 --> 手动。
      - http(s) 配置： 127.0.0.1:7890
      - socket 配置： 127.0.0.1:7891

### 7.3 常用命令

  ```bash
  lspci|grep -i net #查看网卡信息

  systemctl list-unit-files --state=enabled #查看已经启用的服务
  systemd-analyze critical-chain xxx.service #查看关联性服务启动耗费时间
  systemd-analyze blame #按时间排序，查看服务启动耗费时间

  # git 代理设置，推荐放到 .zshrc 中作为常用命令
  git-proxy(){
    git config --global http.proxy socks5://127.0.0.1:1080
    git config --global https.proxy socks5://127.0.0.1:1080
  }
  git-noproxy(){
    git config --global --unset http.proxy
    git config --global --unset https.proxy
  }
  ```
