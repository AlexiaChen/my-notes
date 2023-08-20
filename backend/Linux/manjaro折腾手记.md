
> *青葱岁月，一去不复返啊*

## 前言

对于Linux呢，我自己算不上熟悉，但也不是很陌生。大一上学期的时候在对于操作系统什么都不懂的情况下，废了九牛二虎之力装了个ubuntu。随便敲了下命令行，于是便草草了事。

工作了以后，使用的开发环境windows和Linux都接触过，不过对于windows接触的是最多的。因为大学主要业余写的是win32的代码。工作的时候呢，如果遇到程序需要运行在Linux上，大部分的时候是在windows上用IDE写好代码，然后配置好[cmake](https://cmake.org/),然后cmake生成Linux下的GNU Makefile文件，就可以make工程了。因为工作的稳定需要，接触最多的Linux发行版是CentOS，RedHat，ubuntu，当然还有Fedora和OpenSUSE。

所以对于Linux说实话，还是没有依赖得过多，它对于我来说，不过就是个工具。我没有像其他人那样对某个发行版的热爱上升到信仰程度。但是由于网络上某些人的安利，曾经试着装过具有高度自由，高度灵活的Linux发行版---[Arch Linux](https://www.archlinux.org/)。

当然，对于刚开始接触Arch的我，完全通过命令行安装Arch的过程，还是把我折腾的够呛，不过最后还是装上了。之后，在上面写了个hello world就把Arch卸载了。没用过Arch社区提供的[pacman](https://wiki.archlinux.org/index.php/Pacman)和[AUR](https://aur.archlinux.org/)。

但是最近工作的原因，需要完全在Linux下工作更加方便，所以把家里这台个人电脑的windows全部格了，因为考虑到我这台电脑性能的问题，没有装[Deepin Linux](https://www.deepin.org/)，Deepin UI太炫了，占用资源高，听社区反映有些时候桌面环境会卡死，于是索性就装了个[manjaro Linux](https://manjaro.org/)。

选择manjaro的几个原因如下：

- Arch的衍生版，继承自Arch几乎所有优点（软件源更新非常快，软件时常保持最新，内核滚动升级，利用AUR和pacman的包管理）。

- 图形化的安装界面，一路Next，几乎开箱即用。

- 硬件支持非常好。

- 高度可定制化，而且Xfce的DE，资源占用低。

## 折腾手记

#### 更新系统

因为manjaro是滚动发布的，所以安装完成重启后，进入系统第一件事情最好是打开terminal，运行以下命令:

```bash
sudo pacman -Syu
```
目的是更新本地的源数据库和更新系统。不然如果长时间不升级系统，容易滚挂掉。不过听说manjaro对于Arch很少发生这种意外，非常稳定。

#### 通过NTP服务设置当前系统时间

安装完成的系统跟当前时区的时间不同步，采用NTP协议来同步时间:

```bash
sudo ntpdate cn.ntp.org.cn #可以换成任何一个公开的NTP Server地址
```

#### 安装yaourt

yaourt是pacman的一个外壳，同时它还支持AUR。AUR是一个Arch Linux用户的软件仓库，也就是说，很多用户可以把自己喜欢的软件包上传上去，分享给其他用户。我们在AUR里面搜索得到的软件，可以通过yaourt来简化安装流程。

如果不用yaourt，你恐怕得手动编译软件的很多东西，要多敲十几个命令行，不划算。yaourt这些步骤都对用户透明化了。方便不少。不过这里提示一点，AUR里面的软件毕竟是野生的，没有经过完全测试，没有官方源提供的安全，所以有一定风险，安装一个软件前，需要仔细评估，查看用户评论。但是Arch软件巨多，就是强大在AUR上面了。

运行一下命令就安装yaourt了:

```bash
sudo pacman -S yaourt
```

之后你就可以通过以下命令搜索AUR上的软件了:

```bash
yaourt -Ss [package name]
```

#### 安装Visual Studio Code

毕竟我是微软的粉丝，最近几年微软大力支持开源。VSCode真是良心的开源跨平台的编辑器产品，获得网络上的一致好评，在windiws下用它也用的多。反正Linux下的vi vim nano这些用得不顺手，也不熟悉。

直接用yaourt安装VSCode:

```bash
yaourt -S visual-studio-code-bin
```

#### 安装中文输入法

然后就是安装中文输入法--搜狗输入法。Deepin自带了，毕竟国产Linux。但是manjaro可没这么好。

1. 首先添加国内源

打开/etc/pacman.conf配置文件。并在文件末尾添加:

```conf
[archlinuxcn]
SigLevel = Optional TrustedOnly
Server = https://mirrors.ustc.edu.cn/archlinuxcn/$arch
```

2. 更新源并导入GPG Key

```bash
sudo pacman -Syy && sudo pacman -S archlinuxcn-keyring
```

3. 安装搜狗输入法

```bash
sudo pacman -S fcitx-im #默认全部安装
sudo pacman -S fcitx-configtool # 图形化配置工具
sudo pacman -S fcitx-sogoupinyin
```

4. 编辑添加输入法配置文件

生成并编辑~/.xprofile文件，添加以下内容:

```bash
export GTK_IM_MODULE=fcitx
export QT_IM_MODULE=fcitx
export XMODIFIERS="@im=fcitx"
```

5. 重启系统，使输入法安装生效
6. 通过fcitx-configtool配置工具添加搜狗输入法
7. 完成

#### 安装CLion

喜欢JetBrain系列的IDE，没有IDE简直没法干活，windows上当然是Visual Studio，但是Linux上只能是CLion了。当然，感受了下，它同样强大，作为Linux下的C++ IDE，它够用了，支持CMake工程模型。

```bash
sudo yaourt -S clion
sudo yaourt -S clion-cmake
sudo yaourt -S clion-gdb
sudo yaourt -S clion-jre

```

#### 安装Adobe Flash Player Plugin

这样最起码可以看bilibili了，= =。

```bash
sudo pacman -S flashplugin
```

#### 安装迅雷

我发现Linux下的aria2这个下载神器堪比迅雷，我下载aria2，并配置了aria2.conf中的tracker，还是不行，下载几乎没啥速度。而且aria2也不支持ed2k。而在Linux上支持ed2k的amule客户端下载速度也是以0计算。太烂了，也许不能开箱即用吧。

想想还是迅雷好，国产Deepin已经通过deepin-wine已经完美移植迅雷过来了。所以我们安装下就可以了。

```bash
sudo pacman -S deepin-wine
sudo pacman -S deepin.com.thunderspeed
```

这里我感谢武汉深之度的程序员日日夜夜的努力，这种造福人类的事情值得表扬。当然安装上迅雷有点BUG，就是中文字符显示不出来。我只能猜着点击按钮下载，不过好在界面布局我比较熟悉，下载没出啥意外。电影天堂的大部分ftp电影都能下载下来。

#### 影音娱乐

音乐就听听网易云的Web版本了。电影的话，manjaro自带VLC，这个极度强大，但是界面不好看。把它卸载了。

```bash
sudo pacman -Rss vlc
```

换了deepin出品的deepin-movie。

```bash
sudo pacman -S deepin-movie
```

#### 推荐些命令行神器

话说，工欲善其事必先利其器。

方便好用，我也安装了，这个也是收集大部分网友的答案。

- [ag](https://github.com/ggreer/the_silver_searcher),比grep、ack更快的递归搜索文件内容。

- [shellcheck](https://github.com/koalaman/shellcheck), shell脚本静态检查工具，能够识别语法错误以及不规范的写法。

- [htop](https://hisham.hm/htop/), 提供更美观、更方便的进程监控工具，替代top命令。

- [glances](https://github.com/nicolargo/glances), 更强大的 htop / top 代替者,做综合监控。在某些方面，并没有htop好。

- [ncdu](https://dev.yorhel.nl/ncdu), 可视化的目录占用空间分析程序

- [pandoc](http://www.pandoc.org/), 可以将 markdown 转成各式各样的格式：PDF、DOCX、EPUB、MOBI,我通产用它生成PPT和简历。

- [Graphviz](http://www.graphviz.org/),可以将文本转化为图表

- [httpie](https://github.com/jakubroztocil/httpie), 测试http接口的时候，感觉比curl简单。curl毕竟更底层些，但是curl却更加灵活。