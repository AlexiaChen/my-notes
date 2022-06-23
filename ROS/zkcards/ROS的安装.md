[ROS: Getting Started](https://www.ros.org/blog/getting-started/)


```bash
sudo sh -c '. /etc/lsb-release && echo "deb http://mirrors.ustc.edu.cn/ros/ubuntu/ `lsb_release -cs` main" > /etc/apt/sources.list.d/ros-latest.list'

curl -s https://raw.githubusercontent.com/ros/rosdistro/master/ros.asc | sudo apt-key add -

sudo apt install ros-<ros-dist>-desktop-full

echo "source /opt/ros/<ros-dist>/setup.bash" >> ~/.bashrc
source ~/.bashrc

sudo apt install python3-rosdep python3-rosinstall python3-rosinstall-generator python3-wstool build-essential

sudo rosdep init 
rosdep update

```

注意`rosdep init`在国内下载出错，即使我开了**v2ray**的 http/https的翻墙代理也不行。所以参考[sudo rosdep init ERROR错误处理_知者智者的博客-CSDN博客](https://blog.csdn.net/lclfans1983/article/details/106440711/)

```txt
ERROR: cannot download default sources list from:  
https://raw.githubusercontent.com/ros/rosdistro/master/rosdep/sources.list.d/20-default.list  
Website may be down.
```

```txt
ERROR: Rosdep experienced an error: The read operation timed out  
Please go to the rosdep page [1] and file a bug report with the stack trace below.  
[1] : http://www.ros.org/wiki/rosdep  
  
rosdep version: 0.21.0  
  
Traceback (most recent call last):  
File "/usr/lib/python3/dist-packages/rosdep2/main.py", line 146, in rosdep_main  
exit_code = _rosdep_main(args)  
File "/usr/lib/python3/dist-packages/rosdep2/main.py", line 441, in _rosdep_main  
return _no_args_handler(command, parser, options, args)  
File "/usr/lib/python3/dist-packages/rosdep2/main.py", line 450, in _no_args_handler  
return command_handlers[command](options)  
File "/usr/lib/python3/dist-packages/rosdep2/main.py", line 594, in command_init  
data = download_default_sources_list()  
File "/usr/lib/python3/dist-packages/rosdep2/sources_list.py", line 340, in download_default_sources_list  
f = urlopen(url, timeout=DOWNLOAD_TIMEOUT)  
File "/usr/lib/python3.8/urllib/request.py", line 222, in urlopen  
return opener.open(url, data, timeout)  
File "/usr/lib/python3.8/urllib/request.py", line 525, in open  
response = self._open(req, data)  
File "/usr/lib/python3.8/urllib/request.py", line 542, in _open  
result = self._call_chain(self.handle_open, protocol, protocol +  
File "/usr/lib/python3.8/urllib/request.py", line 502, in _call_chain  
result = func(*args)  
File "/usr/lib/python3.8/urllib/request.py", line 1397, in https_open  
return self.do_open(http.client.HTTPSConnection, req,  
File "/usr/lib/python3.8/urllib/request.py", line 1358, in do_open  
r = h.getresponse()  
File "/usr/lib/python3.8/http/client.py", line 1348, in getresponse  
response.begin()  
File "/usr/lib/python3.8/http/client.py", line 316, in begin  
version, status, reason = self._read_status()  
File "/usr/lib/python3.8/http/client.py", line 277, in _read_status  
line = str(self.fp.readline(_MAXLINE + 1), "iso-8859-1")  
File "/usr/lib/python3.8/socket.py", line 669, in readinto  
return self._sock.recv_into(b)  
File "/usr/lib/python3.8/ssl.py", line 1241, in recv_into  
return self.read(nbytes, buffer)  
File "/usr/lib/python3.8/ssl.py", line 1099, in read  
return self._sslobj.read(len, buffer)  
socket.timeout: The read operation timed out
```

根据traceback判断，修改 `/usr/lib/python3/dist-packages/rosdep2/sources_list.py`下的URL。当然只修改这一个URL，还会报出另外的错误。所以需要修改以下py源码的URL:

```bash
sudo vim /usr/lib/python3/dist-packages/rosdep2/sources_list.py   
sudo vim /usr/lib/python3/dist-packages/rosdep2/gbpdistro_support.py  
sudo vim /usr/lib/python3/dist-packages/rosdistro/__init__.py
```

因为我的WSL2上是ubuntu20.04，默认是python3，所以跟CSDN上python2的目录不一样。

URL修改成，`git clone --depth 1 https://github.com/ros/rosdistro.git` 本地clone的路径URL,用 `file:///home/<username>/rosdistro` 替换成本地的文件URL。不用https请求了。

再把clone下来的rosdistro工程里面的`rosdistro/rosdep/sources.list.d/20-default.list  ` 这个文件里面的URL替换成 `file:///` 为前缀的本地文件URL访问形式，注意路径别填写错了。

最后就成功了:

```txt
mathxh@MathxH:~$ sudo rosdep init  
Wrote /etc/ros/rosdep/sources.list.d/20-default.list  
Recommended: please run  
  
rosdep update  
  
mathxh@MathxH:~$ rosdep update  
reading in sources list data from /etc/ros/rosdep/sources.list.d  
Hit file:///home/mathxh/rosdistro/rosdep/osx-homebrew.yaml  
Hit file:///home/mathxh/rosdistro/rosdep/base.yaml  
Hit file:///home/mathxh/rosdistro/rosdep/python.yaml  
Hit file:///home/mathxh/rosdistro/rosdep/ruby.yaml  
Hit file:///home/mathxh/rosdistro/releases/fuerte.yaml  
Query rosdistro index file:///home/mathxh/rosdistro/index-v4.yaml  
Skip end-of-life distro "ardent"  
Skip end-of-life distro "bouncy"  
Skip end-of-life distro "crystal"  
Skip end-of-life distro "dashing"  
Skip end-of-life distro "eloquent"  
Add distro "foxy"  
Add distro "galactic"  
Skip end-of-life distro "groovy"  
Add distro "humble"  
Skip end-of-life distro "hydro"  
Skip end-of-life distro "indigo"  
Skip end-of-life distro "jade"  
Skip end-of-life distro "kinetic"  
Skip end-of-life distro "lunar"  
Add distro "melodic"  
Add distro "noetic"  
Add distro "rolling"  
updated cache in /home/mathxh/.ros/rosdep/sources.cache 
```

