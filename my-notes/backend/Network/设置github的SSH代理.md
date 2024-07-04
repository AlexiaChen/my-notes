
有一天我用ssh协议git clone代码仓库已经不work了，自然也pull push不了。感谢同事帮助。可以用http来代理ssh。

[adangel.org - Using github ssh behind http proxy](https://adangel.org/2020/10/15/github-behind-proxy/)

在`~/.ssh/config`

```conf
Host github.com
	Hostname ssh.github.com
    Port 443
	ProxyCommand nc -X connect -x 127.0.0.1:10809 %h %p
```

以上127.0.0.1:10809 是本机的http代理的客户端的http代理的地址和端口

这样就可以自由地用ssh协议clone push代码仓库了。