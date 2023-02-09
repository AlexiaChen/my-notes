
#### TEE之Intel SGX的学习资料

    

-   Intel SGX官方入门文档入口 https://www.intel.com/content/www/us/en/developer/tools/software-guard-extensions/get-started.html
    

-   Teaclave 是Apache基金会下孵化中的TEE项目，还没有毕业，也是基于Intel SGX SDK之上做的一个SGX 框架 https://github.com/apache/incubator-teaclave-sgx-sdk
    

-   Intel SGX的一些资料整理 aweome sgx系列 https://github.com/Maxul/Awesome-SGX-Open-Source

## 查看本机Linux环境是否支持SGX

```bash
sudo cpuid | grep -i sgx
```

如果都是看到false，那么肯定是不支持或者没enable了。经过我的验证WSL2环境也是不支持SGX的。Teaclave也是必须要求硬件必须支持SGX。[Solved: SGX in WSL2 - Intel Communities](https://community.intel.com/t5/Intel-Software-Guard-Extensions/SGX-in-WSL2/m-p/1232231)   https://github.com/occlum/occlum/issues/356#issuecomment-798535617 


### 在普通云服务器上安装occlum

因为Occlum的simulation mode可以在无SGX的环境中运行 [occlum/install_occlum_packages.md at master · occlum/occlum (github.com)](https://github.com/occlum/occlum/blob/master/docs/install_occlum_packages.md) 

```bash
# if kernel before v5.9
# on ubuntu20.04  cloud server
git clone --depth 1 
make  
sudo make install  
ls  
./install.sh  
lsmod | grep rdfs
sudo apt update
DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends ca-certificates gnupg2 jq make gdb wget libfuse-dev libtool tzdata
# install intle SGX driver refer to [My First Function | Apache Teaclave (incubating)](https://teaclave.apache.org/docs/my-first-function/) 
wget https://download.01.org/intel-sgx/latest/linux-latest/distro/ubuntu20.04-server/sgx_linux_x64_driver_2.11.054c9c4c.bin
chmod +x ./sgx_linux_x64_driver_2.11.054c9c4c.bin
sudo ./sgx_linux_x64_driver_2.11.054c9c4c.bin
lsmod | grep sgx
sudo modprobe isgx
# to check sgx if it wrong or somthing happens
dmesg | grep sgx
# install SGX PSW etc.
echo 'deb [arch=amd64] https://download.01.org/intel-sgx/sgx_repo/ubuntu focal main' | sudo tee /etc/apt/sources.list.d/intel-sgx.list
wget -qO - https://download.01.org/intel-sgx/sgx_repo/ubuntu/intel-sgx-deb.key | sudo apt-key add -
sudo apt-get update
sudo apt-get install libsgx-launch libsgx-urts libsgx-epid  libsgx-quote-ex  libsgx-aesm-quote-ex-plugin libsgx-aesm-epid-plugin libsgx-dcap-ql libsgx-uae-service libsgx-dcap-quote-verify-dev
# check if it aesmd runing
sudo service aesmd status
# install occlum
echo 'deb [arch=amd64] https://occlum.io/occlum-package-repos/debian focal main' | sudo tee /etc/apt/sources.list.d/occlum.list
wget -qO - https://occlum.io/occlum-package-repos/debian/public.key | apt-key add -
sudo apt-get install -y occlum
```

SGX的软件三大组件是以下三个:

- Intel® SGX Software Development Kit (SDK), which aids Software Developers in creating applications that use Intel SGX Technology. 
- Intel® SGX Platform Software (PSW) for Linux* OS, which provides software modules to run Intel® SGX applications on the Linux* OS. 
- Intel® SGX Data Center Attestation Primitives (DCAP) for Linux* OS, which provides software modules to aid Intel® Applications in performing attestation within the data center.

IAS(Intel Attestation service)
Architectural Enclave Service Manager Deamon（aesm）

从以上的命令来看，期间你用dmsg，或者ls /dev会看到一些错误或者CPU missing sgx的错误现象，但是occlum是支持无SGX的，所以暂时可以不管这些Driver安装的错误。

目前测试，是可以用sim mode运行成功的, 成功输出hello world

```bash
occlum-gcc -o hello_world ./main.c
occlum new hello-world-instance
cd hello-world-instance
cp ../hello_world ./image/bin
occlum build --sgx-mode SIM
occlum run /bin/hello_world
```


## SGX 2.0的资料

[What does SGX 2.0 (scalable SGX) sacrifice (e.g., removing integrity tree) in security for better performance （e.g., large EPC size)? · Issue #899 · intel/linux-sgx (github.com)](https://github.com/intel/linux-sgx/issues/899)

[Intel® 64 and IA-32 Architectures Software Developer's Manual Volume 3D: System Programming Guide, Part 4](https://cdrdv2.intel.com/v1/dl/getContent/671269)

Intel® Software Guard Extensions Developer Guide