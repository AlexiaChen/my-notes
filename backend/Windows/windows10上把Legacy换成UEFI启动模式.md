由于最近想预备升级windows 11（据说bug多，我会暂时等一等，这只是准备工作），发现11必须是UEFI启动，只能这么做了。

先要用win10自带的Disk partition工具查看一下引导卷是否是MBR，如果是就把MBR转成GPT的就可以。

具体来说就是CMD管理员权限运行：

```bash
mbr2gpt.exe /convert /allowfullos
```

成功后，重启电脑，然后进入BIOS设置启动模式，把legacy改成UEFI启动就可以了

参考以下：

- https://www.maketecheasier.com/convert-legacy-bios-uefi-windows10/
- https://www.youtube.com/watch?v=0aRUXQz5ygc