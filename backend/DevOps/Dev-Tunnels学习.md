
这个工具跟ngrok一样，可以代理一个端口到公网上，而且比ngrok更好，主要是使用时间更长，微软的。ngrok太黑了。[What is dev tunnels? - Microsoft dev tunnels | Microsoft Learn](https://learn.microsoft.com/en-us/azure/developer/dev-tunnels/overview)



```bash
devtunnel create -a
devtunnel port create -p 8080 --protocol http  [tunnel id]
devtunnel host [tunnel id]
```