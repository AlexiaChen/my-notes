
# 1 环境准备

  

需要三台支持硬件虚拟化的主机，无论是AMD-V 还是Intel-VT


192.168.13.7  
192.168.13.9  
192.168.13.10 
root  


通过运行一下命令查看CPU是否支持Virtualization:

```bash
egrep -o '(vmx|svm)' /proc/cpuinfo
```

输出vmx就是Intel-VT，输出svm就是AMD-V

