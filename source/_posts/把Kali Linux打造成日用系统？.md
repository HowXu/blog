---
title: 逆向-CE_Tutur9_ShareCode一般处理
date: 2023-04-29
tags: 逆向
index_img: https://i-blog.csdnimg.cn/direct/776e4bf6a12545ef9a80ae36ac7eb4cd.png
---

#### 目录


- [写在最前](#写在最前)
- [1.软件安装](#1软件安装)
	- [软件源](#软件源)
	- [关于显卡驱动](#关于显卡驱动)
	- [浏览器](#浏览器)
	- [中文输入法](#中文输入法)
	- [文本编辑器](#文本编辑器)
	- [下载工具](#下载工具)
	- [QQ](#qq)
	- [Steam](#steam)
	- [录屏软件](#录屏软件)
	- [邮件客户端](#邮件客户端)
	- [Markdown编辑器](#markdown编辑器)
	- [音乐软件](#音乐软件)
	- [视频播放器](#视频播放器)
	- [安装工具](#安装工具)
	- [办公软件](#办公软件)
- [2.使用优化](#2使用优化)
	- [磁盘开机挂载](#磁盘开机挂载)
	- [ssh服务](#ssh服务)
	- [Grub修改](#grub修改)
	- [Zsh和PowerShell命令标头修改](#zsh和powershell命令标头修改)
	- [VSCode内置终端字体间距过大问题](#vscode内置终端字体间距过大问题)
	- [更换桌面环境](#更换桌面环境)
	- [在这里插入图片描述](#在这里插入图片描述)
	- [删除Kali自带的渗透软件库](#删除kali自带的渗透软件库)
	- [关于Kali的启动器菜单选项](#关于kali的启动器菜单选项)
	- [关于Kali的grub定制问题](#关于kali的grub定制问题)
- [3.文末部分](#3文末部分)
	- [Firefox的二进制安装方式](#firefox的二进制安装方式)
	- [原版的Aria2下载器配置](#原版的aria2下载器配置)
	- [Aria2配置文件](#aria2配置文件)




## 写在最前


不定期更新


–2023.12.22–


主板坏了返修双系统估计是无了，以后这个文章不会更新了。


[另一篇关于Kali的软件的文章](https://blog.csdn.net/Coder_Kant/article/details/89462894)


## 1.软件安装


### 软件源


推荐使用中科大源，[官网](https://mirrors.ustc.edu.cn/help/kali.html)  
 在`/etc/apt/source.list`中替换以下内容



```
deb https://mirrors.ustc.edu.cn/kali kali-rolling main non-free contrib
deb-src https://mirrors.ustc.edu.cn/kali kali-rolling main non-free contrib

```

### 关于显卡驱动


如果你用的是N卡，那么百度上绝大多数时候都可以找到靠谱的驱动安装教程，但是你会发现死活找不到A卡的。事实上，A卡的显卡驱动是集成到Linux内核里的，而且是有官方的开源版本的。


也就是说，在系统安装好第一次`sudo apt upgrade`之后就已经安装好A卡驱动了。


但是只有驱动是不够的，还需要安装opengl和vulkan的图形库(类比Windows下的DirectX)。


AMD官方网站的就是闭源的兼容有各类图形库的图形库，称为AMDGPU-PRO。


AMDGPU-PRO的图形库很早停止了对游戏的优化，对于一般用户来说安装是没有意义的，只对某些需要类似AI计算和专业软件的用户才有用。


安装AMDGPU驱动安装:



```
#安装显卡驱动
sudo apt install firmware-linux firmware-linux-nonfree libdrm-amdgpu1 xserver-xorg-video-amdgpu
#如果你是Linux老用户就可以看出来 前面的是Linux默认驱动组，后面的在apt upgrade时就会自动安装


```

AMD官方图形库安装(AMDGPU-PRO)


(总的安装过程就是下载，安装，输指令，照着官方文档敲就行了很简单):


首先是适配问题，不管是什么，只要不是为Kali设计的，Kali大概率不适配(废话)，其次，Kali的内核更新比Ubuntu快太多了，有些时候amdgpu—dkms安装不了就是内核版本不适配。


目前如果想在最新的Kali发行版本上安装只能降低驱动或者系统的版本(后者肯定不可能)，找去年的版本基本上百分百到位。一些可能遇到的问题：


1.os-release未知：把`/etc/os-release`文件中的ID从kali改成debian，注意全小写。


2.大量缺少i386软件包：参考下文的steam安装方式，其中的`sudo dpkg --add-architecture i386` 即可开启i386安装允许下载i386版本软件。


3.有关xorg的abi包依赖无法解决：换版本。


安装完成后先使`sudo apt update`更新(这步不能省)，然后用下面的命令即可安装图形库:



```
amdgpu-install -y --usecase=graphics --vulkan=amdvlk --opencl=rocr,legacy --accept-eula

```

安装开源图形库mesa和vulkan:



```
# 安装DDX驱动支持
sudo apt install xserver-xorg-video-amdgpu

#安装vulkan库和工具
sudo apt install mesa-vulkan-drivers libvulkan1 vulkan-tools vulkan-utils vulkan-validationlayers
#其中的vulkan-utils可能已经被vulkan-tools取代

#安装OpenCL库和工具
apt install mesa-opencl-icd mesa-utils

#安装mesa库(仅限A卡)
sudo apt install mesa libgl1-mesa-dri libgl1-mesa-dev libgl1-amdgpu-mesa-dri libgl1-amdgpu-mesa-glx libgl1


```

最后，你可以用`glxinfo | grep "OpenGL version" && glxinfo | grep "rendering && glxinfo | grep -i vulkan"`在命令行中查看当前显卡情况，正常来讲能够看到上面的OpenGL-Mesa版本，下面的rendering会是Yes，最后可以看到vulkan版本。  
 ![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/0a3f05599fba6522b8c996f7dcb927c2.png)  
 这样驱动就算装完了。


写在最后,可能有用的一些文字：  
 <https://tieba.baidu.com/p/6088650261>  
 <https://tieba.baidu.com/p/7195580365>  
 [ArchWiki-AMDGPU](https://wiki.archlinux.org/title/AMDGPU)


–2023.6.22更新


今天在更新amd官方的图形库后xfce组件全部失效并且无法调整显示器分辨率。我只能说，慎装。


### 浏览器


---------------------------------------- 2024.4.3更新 ----------------------------------------


现在你可以完全放弃Firefox而使用Floorp：


![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/c7c135f3118063fb24bc21439f014855.png)


Floorp使用Firefox的源代码构建，但是更快更强，使用火狐国际版服务，且支持Linux仓库安装。


![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/ac0dbd558bde14d16988357b39332e14.png)


你可以理解为Firefox增强版。


除此之外，Floorp的代码托管在Github上，可以随时查看。


----------------------------------------- 2024.4.3 ---------------------------------------------


Kali自带的浏览器是**Firefox的esr版本(长期支持版)**，因为组织接管的问题用起来不方便，而且Kali的默认软件库里没有正常版本的Firefox安装包，所以要到其他Debian系的软件包仓库里自己下载，但是在这些软件包仓库里能找到的只有国际版格式的deb安装包，这里建议使用**和Kali自带esr版本firefox相同版本号**的安装包。


[PKGS.ORG](https://pkgs.org/download/firefox)  
 [Debian官方仓库](https://packages.debian.org/sid/firefox)


火狐浏览器有分中国版和国际版，推荐使用**国际版**(中国版夹带私货，但是二者共享同步速度几乎没有差别)。


**如果你非要用中国版**:


正常安装国际版Firefox，然后用下面链接的办法把它改成中国版。


[在Firefox国际版使用中国版账户（火狐通行证）傻瓜解决办法](https://blog.csdn.net/qq_32515081/article/details/110428503) 


你也可以想办法把中国版下载的归档文件打包成deb包使用。


[把零散的可执行文件打包成.deb安装包](https://blog.csdn.net/m0_74075298/article/details/133393903)


[中国版官网](http://www.firefox.com.cn/)


[国际版官网](https://www.mozilla.org/zh-CN/firefox/all/#product-desktop-release)


直接通过二进制压缩包格式安装Firefox的方法见[Firefox的二进制安装方式](#Firefox%E7%9A%84%E4%BA%8C%E8%BF%9B%E5%88%B6%E5%AE%89%E8%A3%85%E6%96%B9%E5%BC%8F)


### 中文输入法


你可以选择使用搜狗拼音输入法，[官网](https://shurufa.sogou.com/)


找到并下载搜狗拼音输入法的deb安装包


![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/0c52d47929a9ee212584a42638064444.png)


![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/9fca3990a994f008771d4df839e89b1f.png)


首先安装fcitx组件，如下命令:



```
sudo apt install fcitx

```

之后会报错，这是正常现象。然后输入如下命令：



```
sudo apt --fix-broken install

```

即可完成fcitx的安装。


再使用`sudo dpkg -i 软件包名.deb`安装输入法。


![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/d59223ab8c2494313269fec1d511b19c.png)


之后打开徽标，找到输入法设置，两次都选择是，然后选择fcitx小企鹅输入法，然后重启。


![请添加图片描述](https://i-blog.csdnimg.cn/blog_migrate/4f0c0c12f3679e58cbb27975f2a9d5ce.png)


![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/a3aec21c2eaeec94badd9964e861c72d.png)


安装就完成了。之后在任何可以输入的地方使用**Ctrl + 空格**就可以切换输入法了。


–2023.4.29更新–


新出的4.2版本搜狗输入法别用，用4.0版本的。


–2023.12.21更新–


搜狗最近有进行了一次更新，最新的版本可以在最新的Kali发行版使用。




---


2024.10.14更新


现在我推荐你使用`fcitx5`+`rime`的输入法配置:  
 ![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/248c7905bfcb43b99ae215ab41822230.png)  
 如果你之前使用搜狗输入法,可以先删除原有的`fcitx4`及其组件.


(1)安装`fcitx5`



```
sudo apt install fcitx5
im-config #这个命令会打开图形界面 选择fcitx5即可

```

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/9f64f5bbe7f149e786f7b9995828dfa7.png)


新建文件`~/.xprofile`，写入以下内容:



```
export GTK\_IM\_MODULE=fcitx5
export QT\_IM\_MODULE=fcitx5
export XMODIFIERS="@im=fcitx5"

export LANG="zh\_CN.UTF-8"
export LC\_CTYPE="zh\_CN.UTF-8"

```

之后注销重新登录即可,`fcitx5`安装完成.


(2)安装`rime`



```
sudo apt install fcitx5-rime

```

在托盘处点击配置，并添加 rime，重启`fcitx5`之后选择中州韵作为输入选项就可以输入中文了.


(3)主题配置


我使用的主题是[搜狗极简](https://github.com/sxqsfun/fcitx5-sogou-themes),安装过程大概就是一些文件下载和替换，在项目页写的很详细，这里不赘述。


(4)输入法设置,模糊拼音,词库设置




---


虽然不是必选，但是可以极高地提高使用体验。


我不推荐使用下面的教程 因为以及很老旧了:


先按照[rime-settings](https://github.com/wongdean/rime-settings)教程克隆这个懒人配置，再按照[fcitx5-rime 个人配置以及踩坑点解决简要](https://blacksand.top/2021/10/18/fcitx5-rime%E4%B8%AA%E4%BA%BA%E9%85%8D%E7%BD%AE%E4%BB%A5%E5%8F%8A%E8%B8%A9%E5%9D%91%E7%82%B9%E8%A7%A3%E5%86%B3%E7%AE%80%E8%A6%81/)这篇文章的配置修改部分稍微进行个性化就行了。


推荐使用这个项目:[雾凇拼音](https://github.com/iDvel/rime-ice)


词库可以使用[CustomPinyinDictionary](https://github.com/wuhgit/CustomPinyinDictionary) 这个项目，这已经足够了。如果想要更多其它词库可以参考[ArchWiki](https://wiki.archlinuxcn.org/wiki/Fcitx5?rdfrom=https%3A%2F%2Fwiki.archlinux.org%2Findex.php%3Ftitle%3DFcitx5_%28%25E7%25AE%2580%25E4%25BD%2593%25E4%25B8%25AD%25E6%2596%2587%29%26redirect%3Dno)。


请注意，如果你使用`fcitx5`+`rime`的组合，你的词库文件应该是`.dict.yaml`格式并且应该放在`~/.local/share/fcitx5/rime/`目录下，并且要在当前拼音输入方案的主配置文件中导入。


比如上面的`rime-settings`使用明月拼音，在主配置文件`default.custom.yaml`指定使用明月拼音，`luna_pinyin.custom.yaml`指定使用`luna_pinyin.extended.dict.yaml`载入扩充词库，`luna_pinyin.extended.dict.yaml`又在下方指定词库具体位置。


![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/0846ab1222cf463ab3d2854fee3dc348.png)  
 ![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/dfac45596c17421bada6364afa7cb043.png)  
 而在`rime-ice`项目，即雾凇拼音，的配置文件`rime_ice.schema.yaml`中，指定`rime_ice.dict.yaml`作为词库,`rime_ice.dict.yaml`又指定了各个词库的位置:


![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/43c7c01a2b14497090d4c89f209bfb0c.png)  
 ![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/2e875e117b0d4a368dd22407eacd35ad.png)  
 请灵活地调整需求。


### 文本编辑器


如果是远程服务器连接没有GUI界面的，又不喜欢Vim(抽象的按键和极难的配置和冷战风格的主题)，可以使用**micro**。micro和vim一样都是命令行式的文本编辑器，但是它不使用wq这种东西，它使用**Ctrl+s**进行保存，**Ctrl+q**进行退出(Windows党狂喜)。


附一篇介绍的文章[链接](https://wmdpd.com/qiang-lie-tui-jian-linuxzhong-duan-wen-ben-bian-ji-qi-micro/)


[Micro的官网链接](https://micro-editor.github.io/)


你可以直接使用apt安装micro：



```
sudo apt install micro

```

除此以外，Micro也支持lua格式的插件和各类主题。




---


再除此之外,如果确实很喜欢Vim可以使用[neovim](https://github.com/neovim/neovim)这款vim的分支,同时你可以使用[LazyVim](https://github.com/LazyVim/LazyVim)这个neovim的配置仓库,相当简单但是可以得到一个极其美观开箱即用的vim编辑器:


![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/b5861621333642298ad3a922cbedb88e.png)  
 配置只需要两行命令,如果你之前已经在使用neovim请先做好备份:



```
sudo apt install neovim #安装neovim
git clone https://github.com/LazyVim/starter ~/.config/nvim #克隆配置
rm -rf ~/.config/nvim/.git #删除git
nvim #启动neovim 之后会自动下载各种依赖和配置

```

### 下载工具


在Windows上常用的IDM是没有Linux版本的，推荐使用Motrix下载器，[官网](https://motrix.app/zh-CN/about)


直接下载安装即可，之后只需要像使用IDM一样使用它即可。


安装它的浏览器扩展，因为Motrix是以Aria2为基础的下载器，所以直接在商店搜索Aria2下载管理器，或者也可以到Motrix的扩展页直接点击超链接跳转。


![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/10ef27947914af7eb827cc05f4910fa3.png)  
 使用Motrix**不需要配置rpc密钥**，但是要**把端口改成16800**  
 ![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/5f0c67e00b002b0cf8e8c56dbece6867.png)  
 之后就可以正常使用了，当然你也可以使用原版的Aria2下载器，[原版的Aria2下载器配置](#%E5%8E%9F%E7%89%88%E7%9A%84Aria2%E4%B8%8B%E8%BD%BD%E5%99%A8%E9%85%8D%E7%BD%AE)


### QQ


这个没什么难度，直接百度搜新版LinuxQQ下载deb安装就可以了。


新版的LinuxQQ还可以(早他吗该上Electron的)


### Steam


天下第一Hacker能玩不了Steam？


依然是在steam官网下载deb格式的安装包后安装，但是启动时大概率会有下图的报错提示。


![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/07cbcfabf45d99d296d07ab7d6107b1d.png)


执行下面的命令安装一些运行库：



```
sudo dpkg --add-architecture i386
sudo apt update #这一步不能省
# 其实是安装32位的mesa
sudo apt install libgl1-mesa-dri:i386 libgl1:i386 libc6:i386 xdg-desktop-portal xdg-desktop-portal-gtk

```

等待安装完成即可。


### 录屏软件


虽然Kali有自带的录屏软件，但是我们这种隔壁Windows过来的都会喜欢用OBS.


OBS官网有提供Linux版本安装包，但是只能用ppa库安装，又因为不是Ubuntu的发行版所以不能直接apt下载，要到OBS的Github地址下载：


[Github](https://github.com/obsproject/obs-studio/releases)


![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/0fe39db996387344cff9554853665f14.png)


![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/ccf72352a737800ebff847db48af3fc1.png)  
 后缀是对应的Ubuntu发行版代号，其中的jammy是当前最新的22.04版本。


安装过程一定会遇上运行库的问题，一般用下面的命令解决：



```
sudo apt --fix-broken install

```

顺带一提，其实OBS在Linux上不好用，对显卡极其不友好，建议用Kali桌面环境自带的录屏。


### 邮件客户端


推荐使用BlueMail，使用Electron框架，界面非常美观：


![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/697b5c3d05ebb376ef139d874e87c0de.png)  
 直接下载官网的安装包安装即可，因为是Kali Linux，有些依赖包apt库里是没有的，要自己到Debian的存储库里下载。


[libgconf-2-4](https://packages.debian.org/bookworm/libgconf-2-4)  
 [gconf2-common](https://packages.debian.org/bookworm/gconf2-common)


大致是这两个。


如果提示`GPU process isn't usable. Goodbye`，把它的快捷方式里加上参数`--no-sandbox`即可。


–2023年11月25日更新–


最近在Github上又看到一个邮件客户端，14k的star什么含金量不说了。


[Mailspring\_Github](https://github.com/Foundry376/Mailspring/)  
 ![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/88e215584c5d003c86d317b71a841c10.png)


### Markdown编辑器


可以使用VSCode(真是啥都能干)，只需要下载几个小插件。  
 ![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/29e2b9f150d1f689f1e95b2711660be1.webp?x-image-process=image/format,png)![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/717aa110f989c238942a8eeb073b496b.webp?x-image-process=image/format,png)![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/42e612fa6ea8fe8bced75218b4911562.webp?x-image-process=image/format,png)


当然也可以尝试使用Github上40多k的项目[Marktext](https://github.com/marktext/marktext)，使用Electron构建的应用，界面非常好看。  
 ![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/c157600b1623f228b5ffc36a341615e3.png)  
 但是原项目是没有中文翻译的，可以到[Marktext中文移植](https://github.com/chinayangxiaowei/marktext-chinese-language-pack)下载中文版。


另外，Marktext是全平台通用的，Windows虽然有Typora，但是Marktext又何尝不是一个好选择。


### 音乐软件


我直接用的网易云Linux版，安装包和安装需要注意的地方都可以谷歌搜到。  
 ---------------------------------------- 2024.9.29更新 ----------------------------------------


现在网易云Linux版已经完全跟不上节奏了，不建议用


---------------------------------------- 2024.9.29 ----------------------------------------


也可以用下面两个高颜值的Electron项目


[YesPlayMusic](https://github.com/qier222/YesPlayMusic)


![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/c22048b8c40a104f037b9162be2b775a.png)


[落雪音乐](https://github.com/lyswhut/lx-music-desktop)


![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/7cccea9c05cae38271505e3a2a30e8eb.png)


这两个项目都是Electron全平台通用的。


### 视频播放器


推荐使用VLC


VLC是一个跨平台的免费开源播放器。


请不要用root权限运行VLC。



```
sudo apt install vlc

```

### 安装工具


正常安装deb包都是用`dpkg -i`命令，dpkg不会为我们补齐依赖，额外需要使用`sudo apt --fix-broken install`来补齐依赖。


使用Gdebi安装器可以在安装的同时补齐依赖。



```
sudo apt install gdebi

```

使用:



```
sudo gdebi xxx.deb

```

可以使用`sudo gdebi-gtk`打开它的GUI界面。


### 办公软件


推荐使用WPS Linux版，在官网(wps.cn)即可下载，它是WPS2019版本，无广告。


你也可以使用WPS国际版，访问wps.com获得下载链接，它是WPS2022版本，但是是全英文的，而且无法汉化。


## 2.使用优化


### 磁盘开机挂载


如果是Kali+Windows双系统，需要先到Windows系统中关闭电源管理中的快速启动选项。参考[链接](https://rog.asus.com.cn/support/FAQ/1045548#:~:text=%5BWindows%2010%5D%20%E5%A6%82%E4%BD%95%E5%85%B3%E9%97%ADWindows%E4%B8%AD%E7%9A%84%E5%BF%AB%E9%80%9F%E5%90%AF%E5%8A%A8%E5%8A%9F%E8%83%BD%201%20%E5%9C%A8Windows%20%E6%90%9C%E7%B4%A2%E6%A0%8F%E8%BE%93%E5%85%A5%20%5B%E7%94%B5%E6%BA%90%E5%92%8C%E7%9D%A1%E7%9C%A0%E8%AE%BE%E7%BD%AE%5D%E2%91%A0%20%EF%BC%8C%E7%84%B6%E5%90%8E%E7%82%B9%E5%87%BB,5%20%E5%8F%96%E6%B6%88%E5%8B%BE%E9%80%89%20%5B%E5%90%AF%E7%94%A8%E5%BF%AB%E9%80%9F%E5%90%AF%E5%8A%A8%5D%E2%91%A5%20%EF%BC%8C%E7%84%B6%E5%90%8E%E7%82%B9%E5%87%BB%20%5B%E4%BF%9D%E5%AD%98%E4%BF%AE%E6%94%B9%5D%E2%91%A6%20%E5%8D%B3%E5%8F%AF%E5%85%B3%E9%97%AD%20Windows%20%E4%B8%AD%E7%9A%84%E5%BF%AB%E9%80%9F%E5%90%AF%E5%8A%A8%E5%8A%9F%E8%83%BD%E3%80%82)


首先确定磁盘的挂载目录，比如希望sda1磁盘挂载在/home/kali/data1这个目录下。


通过下面的命令查找uuid：



```
sudo blkid

```

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/ad8e2d05a7598da3f93dee1ebe49465d.png)相应的驱动器后有相应的uuid，不确定是哪块磁盘可以使用`sudo fdisk -l`查看磁盘的大小和文件系统来确定。


之后编辑`/etc/fstab`文件，使用vim或者mousepad都可以。


以使用ntfs系统的sda1磁盘为例，向文件中添加:



```
UUID=5A00276F002750F5 /home/howxu/data/data1 ntfs-3g defaults,rw 0 3

```

其中uuid对应sda1磁盘的uuid；后面的路径是磁盘的挂载路径；ntfs-3g是文件系统；defaults,rw是操作磁盘的参数控制，可以不要改；0是一个默认值，不用改；3是驱动器序号，不可以重复。


这是我自己的fstab文件;


![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/ce7aec04a9c997c09c7cf8b9af833690.png)重启即可。


### ssh服务


通过以下命令控制ssh服务：



```
sudo service ssh start #开启
sudo service ssh restart #重启
sudo service ssh stop #关闭

```

### Grub修改


关于修改主题什么的别的教程都比这篇好，这里写一下怎么改启动项。


使用软件`grub-customizer`可视化更改，使用下列命令安装：



```
sudo apt install grub-customizer

```

之后直接在命令行中输入`sudo grub-customizer`即可启动


![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/b70107ad8d1f05b6a6836c028fa0b761.png)


需要注意的是在grub-customizer里更改启动等待时间是无效的，关于启动时间的配置被默认注释了，将`/etc/default/grub` 的`TIME_OUT`选项的注释去掉即可正常配置。


网上绝大多数的Grub美化教程在Kali上是不适用的，包括用grub美化安装包里的`install.sh`。


唯一的办法是把`/boot/grub/theme/`下的`kali`文件夹替换为别的主题文件夹。


### Zsh和PowerShell命令标头修改


在~目录下使用`sudo mousepad .zshrc` 打开.zshrc文件编辑。第99行的`twoline`之后的`PROMPT`即为命令标头的内容，具体的代表含义可以参考zsh主题编写。


![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/1a6482fc09a2d5933b2741948a84abb4.png)



```
twoline)
            PROMPT=$'%F{%(#.blue.green)}┌──${debian\_chroot:+($debian\_chroot)─}${VIRTUAL\_ENV:+($(basename $VIRTUAL\_ENV))─}(%B%F{%(#.red.blue)}%n'$prompt\_symbol$'%m%b%F{%(#.blue.green)})-[%B%F{reset}%(6~.%-1~/…/%4~.%5~)%b%F{%(#.blue.green)}]\n└─%B%(#.%F{red}#.%F{blue}$)%b%F{reset} '

```

在~目录下的`.config/powershell`文件夹可以找到`Microsoft.PowerShell_profile.ps1`文件。从15行开始的`twoline`是标头部分


![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/630f8295628dc1f68b7fae8aec7c446d.png)



```
If ($PROMPT\_ALTERNATIVE -eq 'twoline') {
    Write-Host "┌──(" -NoNewLine -ForegroundColor Blue
    Write-Host "${bold}$([environment]::username)㉿$([system.environment]::MachineName)${reset}" -NoNewLine -ForegroundColor Magenta
    Write-Host ")-[" -NoNewLine -ForegroundColor Blue
    Write-Host "${bold}$(Get-Location)${reset}" -NoNewLine -ForegroundColor White
    Write-Host "]" -ForegroundColor Blue
    Write-Host "└─" -NoNewLine -ForegroundColor Blue
    Write-Host "${bold}PS>${reset}" -NoNewLine -ForegroundColor Blue

```

### VSCode内置终端字体间距过大问题


如图


![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/085944b951cedc56968be63439015c67.png)


这个问题的根源在于Kali默认的终端字体在VSCode查找不到。打开终端，选择文件->参数配置，点击字体。在这个里面看到的字体都是可以被VSCode编辑器和终端正常识别的，随便换一个就可以了。我建议是和终端一致。


![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/99ea8963e2bbca11b445e2aa62fb2323.png)


在VSCode里选择设置，在搜索栏里搜索`terminal.font`，像下图一样把字体名填进去就可以了，不用加引号。


![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/d53948d78992ad50384ce1423251390c.png)


![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/83250f36daa0381cb2616b9dca054808.png)


### 更换桌面环境


Kali默认使用Xfce作为桌面，虽然说极客味道确实很浓，但是如果阁下看到如下的KDE桌面阁下又会如何应对：


![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/0680de6939454215302da6cf2eb243a0.png#pic_center)


推荐的文章：  
 [KDE官方关于桌面特效性能](https://userbase.kde.org/Desktop_Effects_Performance/zh-cn#%E6%A1%8C%E9%9D%A2%E7%89%B9%E6%95%88%E6%80%A7%E8%83%BD)  
 [KDE配置终端模糊和右键菜单模糊](https://www.cnblogs.com/architectforest/p/15185737.html)  
 [KDE 终极美化指南](https://hujiekang.top/posts/kde-customization/)


安装步骤如下：


首先安装kali自带的KDE桌面环境，kali官方设置了存储虚包非常方便：



```
sudo apt install kali-desktop-kde

```

安装过程中会更改显示适配服务之类的，反正是二选一选没没选那个就可以了。之后重启就可以进到KDE的等离子桌面了。



```
sudo apt remove kali-desktop-xfce #这条是重启之后删除xfce桌面的
sudo apt autoremove && sudo apt autoclen #删除一些缓存和垃圾

```

之后对其进行稍微的配置，打开设置，先把**工作区行为的单击文件行为改成选中**(改了就知道了)。


![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/44abb3f9bfc9798e5aba04c76c4fda5f.png)  
 之后下载kvantum管理器，也是一条命令：



```
sudo apt install qt5-style-kvantum

```

这样准备工作就一大半了，剩下的就是挑选主题的事情：


我使用的全局主题是[We10XOS-dark](https://www.pling.com/p/1368859)，图标用的是[Fluent icon](https://www.pling.com/p/1477945)，窗口装饰元素是默认的Breeze(这里推荐[FutureTheme](https://www.pling.com/p/1491484/)的窗口装饰，但是和Firefox不合所以我用的Breeze，如果有Breeze魔改的可以拿来用一下)。


——2023.1.28更新——  
 Firefox终于更新了，现在Firefox中国版有自己的窗口栏，可以自由地使用FutureDark窗口装饰了！  
 ——————————


对照着网站上给的文档安装就可以了，之后配置一下kvantum，打开kvantum manager:  
 ![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/931061f00b8470fdd2ffa43b18b97821.png)  
 选择变更删除主题：  
 ![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/d0b4d6dc502aae3be2d06ee2326970b8.png)


下拉选择We10XOS-dark:  
 ![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/9aaac76e7bbf35879b55345934a2ab3a.png)  
 之后到设置里把**应用程序风格**更改为kvantum，这一步很重要。之后重启就可以看到完全的效果了。


桌面壁纸使用的是[Smart Video Wallpaper](https://store.kde.org/p/1316299)，也是按着教程安装一下，然后切换壁纸设置里的管理器，选择自己的壁纸就可以了。


![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/1e58de7eea874c1845a82aa93b9a0ef7.png)  
 下方停靠栏使用的是latte-dock，一条指令安装：



```
sudo apt install latte-dock 
latte-dock #启动

```

配置的话看个人，别忘了给它加一个自启动：


![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/533155749d2fa6584103188727d7978b.png)上方任务栏从左到右是[Application Title](https://store.kde.org/p/1199712)，KDE自带的全局菜单，KDE自带的一些组件。最后的样子就是这样。如果有闲情雅致还可以换一下SDDM登录屏幕，但是我这里截不了图就不再细了。


[Sugar Candy for SDDM](https://www.pling.com/p/1312658/)


### 在这里插入图片描述


2024.10.14更新


现在欣赏一下更好地KDE吧。  
 ![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/776e4bf6a12545ef9a80ae36ac7eb4cd.png)


鉴于之前说的主题很多都没有更新，所以我又换了一些。这些主题都可以直接在KDE Store或者Pling或者直接可以下载：


应用程序风格:kvantum+We10Dark  
 Plasma视觉元素:Sweet  
 窗口装饰元素:Aritim-Dark  
 颜色:FutureDark  
 图标:ePapirus-Dark  
 光标:Win10OS CUrsors


### 删除Kali自带的渗透软件库


Kali自带了很多的渗透软件，但有些直到我们卸载Kali都用不上，一个一个麻烦的删除肯定不行。


Kali官方使用MetaPackage的方式进行渗透软件库的管理，即创建一个体量很小的被定义为系统级的包，该包的依赖为各类渗透测试软件，通过安装小体量包安装渗透软件库的方式。这个方式便于管理，也便于我们卸载它们。


[Kali官方MetaPackage目录](https://www.kali.org/tools/kali-meta)


![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/68d35ecb8267b92f8f432d09995e9b2e.jpg)


[Kali官方MetaPackage介绍](https://www.kali.org/docs/general-use/metapackages/)


![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/b7090aa0a72febeba95c45ab8932adda.jpg)


执行下面的命令卸载这些MetaPackage再使用apt的自动卸载功能即可卸载自带的渗透软件库:



```
#卸载指定xxxx类型
sudo apt remove kali-tools-xxxx
sudo apt --autoremove && sudo apt --autoclean

#卸载所有渗透工具库
sudo apt remove kali-tools-*
sudo apt --autoremove && sudo apt --autoclean

#

```

### 关于Kali的启动器菜单选项


参考[第九贴：清除KDE应用程序菜单](http://blog.chinaunix.net/uid-25117262-id-82029.html)


### 关于Kali的grub定制问题


如果你的Kali设置了单独的boot分区，那网上的包括换主题，换分辨率的教程是是无效的(原因我也不清楚)。但是通过`grub-customizer`进行启动项定制是可行的。


但是也不是确实没有办法，在`usr/share/grub`，`etc/grub`之外应该直接编辑`boot/grub`下的文件，编辑其中的grub.cfg可以修改分辨率，向themes文件夹添加主题并且在grub.cfg中设置也可以生效。但是这个方法非常危险容易直接干废引导，建议备个份再上手。除此之外，这个方式的修改会**在每次系统大更新或者update grub**之后失效。


## 3.文末部分


### Firefox的二进制安装方式






使用下列命令卸载自带火狐浏览器：



```
sudo apt remove firefox-esr

```

访问火狐浏览器的官方网站


[中国版官网](http://www.firefox.com.cn/)


[国际版官网](https://www.mozilla.org/zh-CN/firefox/all/#product-desktop-release)


下载火狐软件包，并解压到/opt目录下![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/6761cdcc02b165b602ccbcf9b8e46276.png)![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/0780d489c80689e0ca0196a97bfd4b59.png)之后打开徽标，找到默认应用程序，切换为/opt/firefox目录下的firefox应用程序。![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/f64e940037b1f019a8e9001971e5eff1.png)之后可以同时也修改任务栏上的firefox徽标，将启动命令替换为`/opt/firefox/firefox`。


另外的还可以为firefox创建软链接使随处可以运行。



```
sudo ln /opt/firefox/firefox /usr/bin/firefox

```

按照需求可以在桌面上创建快捷方式，右键新建文本文档，内容为:



```
[Desktop Entry]
Type=Application
Version=123 #应用版本
Name=Firefox #应用名称
Comment=Run Firefox #启动器描述
Icon=/example/firefox.png #启动器图标路径
Exec=/example/firefox exampleargs#启动命令，=后面可以有空格
Categories=Application #软件类别，这个是系统内置的
Terminal=false #是否开启
TerminalPath=StartupNotify=false

```

之后将该文件后缀名改为`.desktop`即可


如果想要让它在启动器里的位置好一点可以看这篇[文章](https://unix.stackexchange.com/questions/177212/add-new-category-to-applications)，添加自己的软件类别。


### 原版的Aria2下载器配置






[Github官方发布页](https://github.com/aria2/aria2)


可以使用下列命令安装；



```
sudo apt install aria2

```

在/opt目录下新建`aria2`目录用来放配置文件。


新建两个文件，分别命名为`aria2.session`和`aria2.conf`。


前者不用动，后者为配置文件(附文末[Aria2配置文件](#Aria2%E9%85%8D%E7%BD%AE%E6%96%87%E4%BB%B6))。


需要注意的是前四个配置以及rpc密钥设置。


之后添加在`/usr/lib/systemd/system`目录下添加服务文件`aria2.service`：



```
[Unit]
Description=Aria2
After= 

[Service]
Type=simpleExec
Start=aria2c --conf-path=/opt/aria2/aria2.conf
User=root
Group=root
ExecStop=
ExecReload= 

[Install]
WantedBy=multi-user.target


```

之后输入下面命令:



```
sudo systemctl enable aria2 #设置自启动sudo service aria2 start #启动sudo service aria2 restart #重启

```

安装完成，可以额外在firefox扩展商店安装Aria2的扩展，如果在配置文件中设置了rpc密钥，就需要在扩展中配置rpc密钥：![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/10ef27947914af7eb827cc05f4910fa3.png)  
 ![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/ae3d7d62b21f6428575c551617f94f68.png)


### Aria2配置文件







```

# 下载目录。可使用绝对路径或相对路径, 默认: 当前启动位置
dir=/home/howxu/下载

# 从会话文件中读取下载任务
input-file=/opt/aria2/aria2.session

# 会话文件保存路径
# Aria2 退出时或指定的时间间隔会保存`错误/未完成`的下载任务到会话文件
save-session=/opt/aria2/aria2.session

# RPC 密钥
rpc-secret=howxu

# 磁盘缓存, 0 为禁用缓存，默认:16M
# 磁盘缓存的作用是把下载的数据块临时存储在内存中，然后集中写入硬盘，以减少磁盘 I/O ，提升读写性能，延长硬盘寿命。
# 建议在有足够的内存空闲情况下适当增加，但不要超过剩余可用内存空间大小。
# 此项值仅决定上限，实际对内存的占用取决于网速(带宽)和设备性能等其它因素。
disk-cache=64M

# 文件预分配方式, 可选：none, prealloc, trunc, falloc, 默认:prealloc
# 预分配对于机械硬盘可有效降低磁盘碎片、提升磁盘读写性能、延长磁盘寿命。
# 机械硬盘使用 ext4（具有扩展支持），btrfs，xfs 或 NTFS（仅 MinGW 编译版本）等文件系统建议设置为 falloc
# 若无法下载，提示 fallocate failed.cause：Operation not supported 则说明不支持，请设置为 none
# prealloc 分配速度慢, trunc 无实际作用，不推荐使用。
# 固态硬盘不需要预分配，只建议设置为 none ，否则可能会导致双倍文件大小的数据写入，从而影响寿命。
file-allocation=none

# 文件预分配大小限制。小于此选项值大小的文件不预分配空间，单位 K 或 M，默认：5M
no-file-allocation-limit=64M

# 断点续传
continue=true

# 始终尝试断点续传，无法断点续传则终止下载，默认：true
always-resume=false

# 不支持断点续传的 URI 数值，当 always-resume=false 时生效。
# 达到这个数值从将头开始下载，值为 0 时所有 URI 不支持断点续传时才从头开始下载。
max-resume-failure-tries=0

# 获取服务器文件时间，默认:false
remote-time=true


## 进度保存设置 ##

# 任务状态改变后保存会话的间隔时间（秒）, 0 为仅在进程正常退出时保存, 默认:0
# 为了及时保存任务状态、防止任务丢失，此项值只建议设置为 1
save-session-interval=1

# 自动保存任务进度到控制文件(\*.aria2)的间隔时间（秒），0 为仅在进程正常退出时保存，默认：60
# 此项值也会间接影响从内存中把缓存的数据写入磁盘的频率
# 想降低磁盘 IOPS (每秒读写次数)则提高间隔时间
# 想在意外非正常退出时尽量保存更多的下载进度则降低间隔时间
# 非正常退出：进程崩溃、系统崩溃、SIGKILL 信号、设备断电等
auto-save-interval=20

# 强制保存，即使任务已完成也保存信息到会话文件, 默认:false
# 开启后会在任务完成后保留 .aria2 文件，文件被移除且任务存在的情况下重启后会重新下载。
# 关闭后已完成的任务列表会在重启后清空。
force-save=false


## 下载连接设置 ##

# 文件未找到重试次数，默认:0 (禁用)
# 重试时同时会记录重试次数，所以也需要设置 max-tries 这个选项
max-file-not-found=10

# 最大尝试次数，0 表示无限，默认:5
max-tries=0

# 重试等待时间（秒）, 默认:0 (禁用)
retry-wait=10

# 连接超时时间（秒）。默认：60
connect-timeout=10

# 超时时间（秒）。默认：60
timeout=10

# 最大同时下载任务数, 运行时可修改, 默认:5
max-concurrent-downloads=5

# 单服务器最大连接线程数, 任务添加时可指定, 默认:1
# 最大值为 16 (增强版无限制), 且受限于单任务最大连接线程数(split)所设定的值。
max-connection-per-server=16

# 单任务最大连接线程数, 任务添加时可指定, 默认:5
split=64

# 文件最小分段大小, 添加时可指定, 取值范围 1M-1024M (增强版最小值为 1K), 默认:20M
# 比如此项值为 10M, 当文件为 20MB 会分成两段并使用两个来源下载, 文件为 15MB 则只使用一个来源下载。
# 理论上值越小使用下载分段就越多，所能获得的实际线程数就越大，下载速度就越快，但受限于所下载文件服务器的策略。
min-split-size=4M

# HTTP/FTP 下载分片大小，所有分割都必须是此项值的倍数，最小值为 1M (增强版为 1K)，默认：1M
piece-length=1M

# 允许分片大小变化。默认：false
# false：当分片大小与控制文件中的不同时将会中止下载
# true：丢失部分下载进度继续下载
allow-piece-length-change=true

# 最低下载速度限制。当下载速度低于或等于此选项的值时关闭连接（增强版本为重连），此选项与 BT 下载无关。单位 K 或 M ，默认：0 (无限制)
lowest-speed-limit=0

# 全局最大下载速度限制, 运行时可修改, 默认：0 (无限制)
max-overall-download-limit=0

# 单任务下载速度限制, 默认：0 (无限制)
max-download-limit=0

# 禁用 IPv6, 默认:false
disable-ipv6=true

# GZip 支持，默认:false
http-accept-gzip=true

# URI 复用，默认: true
reuse-uri=false

# 禁用 netrc 支持，默认:false
no-netrc=true

# 允许覆盖，当相关控制文件(.aria2)不存在时从头开始重新下载。默认:false
allow-overwrite=false

# 文件自动重命名，此选项仅在 HTTP(S)/FTP 下载中有效。新文件名在名称之后扩展名之前加上一个点和一个数字（1..9999）。默认:true
auto-file-renaming=true

# 使用 UTF-8 处理 Content-Disposition ，默认:false
content-disposition-default-utf8=true

# 最低 TLS 版本，可选：TLSv1.1、TLSv1.2、TLSv1.3 默认:TLSv1.2
#min-tls-version=TLSv1.2


## BT/PT 下载设置 ##

# BT 监听端口(TCP), 默认:6881-6999
# 直通外网的设备，比如 VPS ，务必配置防火墙和安全组策略允许此端口入站
# 内网环境的设备，比如 NAS ，除了防火墙设置，还需在路由器设置外网端口转发到此端口
listen-port=51413

# DHT 网络与 UDP tracker 监听端口(UDP), 默认:6881-6999
# 因协议不同，可以与 BT 监听端口使用相同的端口，方便配置防火墙和端口转发策略。
dht-listen-port=51413

# 启用 IPv4 DHT 功能, PT 下载(私有种子)会自动禁用, 默认:true
enable-dht=true

# 启用 IPv6 DHT 功能, PT 下载(私有种子)会自动禁用，默认:false
# 在没有 IPv6 支持的环境开启可能会导致 DHT 功能异常
enable-dht6=false

# 指定 BT 和 DHT 网络中的 IP 地址
# 使用场景：在家庭宽带没有公网 IP 的情况下可以把 BT 和 DHT 监听端口转发至具有公网 IP 的服务器，在此填写服务器的 IP ，可以提升 BT 下载速率。
#bt-external-ip=

# IPv4 DHT 文件路径，默认：$HOME/.aria2/dht.dat
dht-file-path=/root/.aria2/dht.dat

# IPv6 DHT 文件路径，默认：$HOME/.aria2/dht6.dat
dht-file-path6=/root/.aria2/dht6.dat

# IPv4 DHT 网络引导节点
dht-entry-point=dht.transmissionbt.com:6881

# IPv6 DHT 网络引导节点
dht-entry-point6=dht.transmissionbt.com:6881

# 本地节点发现, PT 下载(私有种子)会自动禁用 默认:false
bt-enable-lpd=true

# 指定用于本地节点发现的接口，可能的值：接口，IP地址
# 如果未指定此选项，则选择默认接口。
#bt-lpd-interface=

# 启用节点交换, PT 下载(私有种子)会自动禁用, 默认:true
enable-peer-exchange=true

# BT 下载最大连接数（单任务），运行时可修改。0 为不限制，默认:55
# 理想情况下连接数越多下载越快，但在实际情况是只有少部分连接到的做种者上传速度快，其余的上传慢或者不上传。
# 如果不限制，当下载非常热门的种子或任务数非常多时可能会因连接数过多导致进程崩溃或网络阻塞。
# 进程崩溃：如果设备 CPU 性能一般，连接数过多导致 CPU 占用过高，因资源不足 Aria2 进程会强制被终结。
# 网络阻塞：在内网环境下，即使下载没有占满带宽也会导致其它设备无法正常上网。因远古低性能路由器的转发性能瓶颈导致。
bt-max-peers=128

# BT 下载期望速度值（单任务），运行时可修改。单位 K 或 M 。默认:50K
# BT 下载速度低于此选项值时会临时提高连接数来获得更快的下载速度，不过前提是有更多的做种者可供连接。
# 实测临时提高连接数没有上限，但不会像不做限制一样无限增加，会根据算法进行合理的动态调节。
bt-request-peer-speed-limit=10M

# 全局最大上传速度限制, 运行时可修改, 默认:0 (无限制)
# 设置过低可能影响 BT 下载速度
max-overall-upload-limit=2M

# 单任务上传速度限制, 默认:0 (无限制)
max-upload-limit=0

# 最小分享率。当种子的分享率达到此选项设置的值时停止做种, 0 为一直做种, 默认:1.0
# 强烈建议您将此选项设置为大于等于 1.0
seed-ratio=1.0

# 最小做种时间（分钟）。设置为 0 时将在 BT 任务下载完成后停止做种。
seed-time=0

# 做种前检查文件哈希, 默认:true
bt-hash-check-seed=true

# 继续之前的BT任务时, 无需再次校验, 默认:false
bt-seed-unverified=false

# BT tracker 服务器连接超时时间（秒）。默认：60
# 建立连接后，此选项无效，将使用 bt-tracker-timeout 选项的值
bt-tracker-connect-timeout=10

# BT tracker 服务器超时时间（秒）。默认：60
bt-tracker-timeout=10

# BT 服务器连接间隔时间（秒）。默认：0 (自动)
#bt-tracker-interval=0

# BT 下载优先下载文件开头或结尾
bt-prioritize-piece=head=32M,tail=32M

# 保存通过 WebUI(RPC) 上传的种子文件(.torrent)，默认:true
# 所有涉及种子文件保存的选项都建议开启，不保存种子文件有任务丢失的风险。
# 通过 RPC 自定义临时下载目录可能不会保存种子文件。
rpc-save-upload-metadata=true

# 下载种子文件(.torrent)自动开始下载, 默认:true，可选：false|mem
# true：保存种子文件
# false：仅下载种子文件
# mem：将种子保存在内存中
follow-torrent=true

# 种子文件下载完后暂停任务，默认：false
# 在开启 follow-torrent 选项后下载种子文件或磁力会自动开始下载任务进行下载，而同时开启当此选项后会建立相关任务并暂停。
pause-metadata=false

# 保存磁力链接元数据为种子文件(.torrent), 默认:false
bt-save-metadata=true

# 加载已保存的元数据文件(.torrent)，默认:false
bt-load-saved-metadata=true

# 删除 BT 下载任务中未选择文件，默认:false
bt-remove-unselected-file=true

# BT强制加密, 默认: false
# 启用后将拒绝旧的 BT 握手协议并仅使用混淆握手及加密。可以解决部分运营商对 BT 下载的封锁，且有一定的防版权投诉与迅雷吸血效果。
# 此选项相当于后面两个选项(bt-require-crypto=true, bt-min-crypto-level=arc4)的快捷开启方式，但不会修改这两个选项的值。
bt-force-encryption=true

# BT加密需求，默认：false
# 启用后拒绝与旧的 BitTorrent 握手协议(\19BitTorrent protocol)建立连接，始终使用混淆处理握手。
#bt-require-crypto=true

# BT最低加密等级，可选：plain（明文），arc4（加密），默认：plain
#bt-min-crypto-level=arc4

# 分离仅做种任务，默认：false
# 从正在下载的任务中排除已经下载完成且正在做种的任务，并开始等待列表中的下一个任务。
bt-detach-seed-only=true


## 客户端伪装 ##

# 自定义 User Agent
user-agent=Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/93.0.4577.63 Safari/537.36 Edg/93.0.961.47

# BT 客户端伪装
# PT 下载需要保持 user-agent 和 peer-agent 两个参数一致
# 部分 PT 站对 Aria2 有特殊封禁机制，客户端伪装不一定有效，且有封禁账号的风险。
#user-agent=Deluge 1.3.15
peer-agent=Deluge 1.3.15
peer-id-prefix=-DE13F0-


## 执行额外命令 ##

# 下载停止后执行的命令
# 从 正在下载 到 删除、错误、完成 时触发。暂停被标记为未开始下载，故与此项无关。
on-download-stop=/root/.aria2/delete.sh

# 下载完成后执行的命令
# 此项未定义则执行 下载停止后执行的命令 (on-download-stop)
on-download-complete=/root/.aria2/clean.sh

# 下载错误后执行的命令
# 此项未定义则执行 下载停止后执行的命令 (on-download-stop)
#on-download-error=

# 下载暂停后执行的命令
#on-download-pause=

# 下载开始后执行的命令
#on-download-start=

# BT 下载完成后执行的命令
#on-bt-download-complete=


## RPC 设置 ##

# 启用 JSON-RPC/XML-RPC 服务器, 默认:false
enable-rpc=true

# 接受所有远程请求, 默认:false
rpc-allow-origin-all=true

# 允许外部访问, 默认:false
rpc-listen-all=true

# RPC 监听端口, 默认:6800
rpc-listen-port=6800


# RPC 最大请求大小
rpc-max-request-size=10M

# RPC 服务 SSL/TLS 加密, 默认：false
# 启用加密后必须使用 https 或者 wss 协议连接
# 不推荐开启，建议使用 web server 反向代理，比如 Nginx、Caddy ，灵活性更强。
#rpc-secure=false

# 在 RPC 服务中启用 SSL/TLS 加密时的证书文件(.pem/.crt)
#rpc-certificate=/root/.aria2/xxx.pem

# 在 RPC 服务中启用 SSL/TLS 加密时的私钥文件(.key)
#rpc-private-key=/root/.aria2/xxx.key

# 事件轮询方式, 可选：epoll, kqueue, port, poll, select, 不同系统默认值不同
#event-poll=select


## 高级选项 ##

# 启用异步 DNS 功能。默认：true
#async-dns=true

# 指定异步 DNS 服务器列表，未指定则从 /etc/resolv.conf 中读取。
#async-dns-server=119.29.29.29,223.5.5.5,8.8.8.8,1.1.1.1

# 指定单个网络接口，可能的值：接口，IP地址，主机名
# 如果接口具有多个 IP 地址，则建议指定 IP 地址。
# 已知指定网络接口会影响依赖本地 RPC 的连接的功能场景，即通过 localhost 和 127.0.0.1 无法与 Aria2 服务端进行讯通。
#interface=

# 指定多个网络接口，多个值之间使用逗号(,)分隔。
# 使用 interface 选项时会忽略此项。
#multiple-interface=


## 日志设置 ##

# 日志文件保存路径，忽略或设置为空为不保存，默认：不保存
#log=

# 日志级别，可选 debug, info, notice, warn, error 。默认：debug
#log-level=warn

# 控制台日志级别，可选 debug, info, notice, warn, error ，默认：notice
console-log-level=notice

# 安静模式，禁止在控制台输出日志，默认：false
quiet=false

# 下载进度摘要输出间隔时间（秒），0 为禁止输出。默认：60
summary-interval=0


## 增强扩展设置(非官方) ##

# 仅适用于 myfreeer/aria2-build-msys2 (Windows) 和 P3TERX/Aria2-Pro-Core (GNU/Linux) 项目所构建的增强版本

# 在服务器返回 HTTP 400 Bad Request 时重试，仅当 retry-wait > 0 时有效，默认 false
#retry-on-400=true

# 在服务器返回 HTTP 403 Forbidden 时重试，仅当 retry-wait > 0 时有效，默认 false
#retry-on-403=true

# 在服务器返回 HTTP 406 Not Acceptable 时重试，仅当 retry-wait > 0 时有效，默认 false
#retry-on-406=true

# 在服务器返回未知状态码时重试，仅当 retry-wait > 0 时有效，默认 false
#retry-on-unknown=true

# 是否发送 Want-Digest HTTP 标头。默认：false (不发送)
# 部分网站会把此标头作为特征来检测和屏蔽 Aria2
#http-want-digest=false


```




