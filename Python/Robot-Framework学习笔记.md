# RobotFramework

这是Python的一个自动化测试框架


## 安装

[手把手教你python robotframework](https://baijiahao.baidu.com/s?id=1693020882166488853&wfr=spider&for=pc)

[Robot Framework python3安装教程](https://blog.csdn.net/dltsdr/article/details/106182277)

```bash
pip install wxPython -i https://pypi.tuna.tsinghua.edu.cn/simple/
pip install robotframework -i https://pypi.tuna.tsinghua.edu.cn/simple/
pip install robotframework-ride -i https://pypi.tuna.tsinghua.edu.cn/simple/

```

RIDE启动失败  https://github.com/robotframework/RIDE/issues/2238 

当(python < 3.9) Install current Beta version (2.0b1) with: 

```bash
pip install psutil -i https://pypi.tuna.tsinghua.edu.cn/simple/
pip install -U --pre robotframework-ride -i https://pypi.tuna.tsinghua.edu.cn/simple/
```

Ride其实是给小白用户使用robot framework的一个开发测试用例的环境，也是测试数据编辑器。



robot 支持一些内建的library和外部的库，比如selenium library就是自动化测试网页的，有打开浏览器等方法

```bash
pip install robotframework-selenium2library -i https://pypi.tuna.tsinghua.edu.cn/simple/
```

这个库就被安装到Python安装目录的Lib目录下的site-packages下面，这样只要在testcase中导入这个库，那么就可以使用这个库的对外暴露出的关键字(方法接口)了。之后就可以在ride的search keywor功能里搜索关键字了

注意，对于Web自动化测试这块，需要下载对应浏览器的WebDriver。

不然会报以下错误，要把WebDriver添加到PATH路径:

```txt
WebDriverException: Message: 'msedgedriver' executable needs to be in PATH.
```


## 教程及相关资料



[python robotframework测试框架分享](https://blog.csdn.net/xgh1951/article/details/123067010)

[Robot framework tutorial](https://www.tutorialspoint.com/robot_framework/index.htm)


## 测试用例样例

![[c408f8d38a7aec074630f15365e4c7b.png]]

![[822f0a7cb7b03bde22a4a0897cc30ed.png]]

以上就是robot的配置文件本身，只是Ride这个图形界面是这些配置文件的可视化和IDE开发环境而已，Ride通过调用robot 的 API来执行配置文件定义的测试样例。

