
昨天买的小米新款笔记本到货，配置是i5 内存8G DDR4。反正是够我拿来写代码了 = =。 昨晚兴致勃勃地冲回新家，准备把小米本里面的原装windows 10 格了重做一个manjaro linux，但是奇怪的事情是，原本对硬件一向支持友好的manjaro居然装起来找不到无线网络，我靠居然没有支持，查了网卡的型号是Reatek的rtl8821ce的无线网卡，上网一搜，我擦，原来在去年的时候Linux还不支持这个型号的无线网卡。到今年为止才可以[下载到这个型号网卡的驱动](https://ask.csdn.net/questions/662758)。

而且一般OS都不内置，要自己安装，有些时候可能还得[自己编译驱动](https://blog.csdn.net/qq_33042187/article/details/80462412)。最奇怪的是，像manjaro这种内核滚动升级的，时刻保持最新的，居然内核没有带这个驱动。唉，暂时不去研究了。

说多了是泪。TT ， 只有去小区背后的五金店买了个网线先插起来用着，然后通过：

```bash
yay -S rtl8821ce-dkms-git
```

来安装驱动，安装了驱动重启电脑，居然又没有起作用，后来[一查](https://forum.manjaro.org/t/wifi-adapter-not-working/65463), 才知道还需要安装内核的headers

无奈又通过：

```bash
sudo pacman -S linux419-headers
```

因为我是重装的manjaro所以是最新的manjaro18  内核是4.19.x，所以要安装对应的headers。

然后，我擦了，还不起作用，原来还需要启动驱动模块：

```bash
sudo modprobe 8821ce
```

然后manjaro的xfce的无线管理才能突然搜索到无线网络，唉，可真把我折腾的够呛。

