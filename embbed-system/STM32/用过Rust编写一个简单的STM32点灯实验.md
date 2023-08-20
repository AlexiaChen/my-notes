
设备用的就是[[STM32学习笔记]] 中提到的，STM32F103  Medium Density


## 安装

首先Rust安装不做赘述

因为是STM32F103，所以用的是Cortex-M3。

```bash
rustup target add thumbv7m-none-eabi
```

接下来:

```bash
cargo install cargo-binutils
rustup component add llvm-tools-preview
```



## References

- [Introduction - The Embedded Rust Book (rust-embedded.org)](https://docs.rust-embedded.org/book/)
- [Rust and STM32: A Quick Start Guide - Nautilus (bacelarhenrique.me)](https://bacelarhenrique.me/2021/02/21/rust-and-stm32-a-quick-start-guide.html)
- [Rust 嵌入式开发 STM32 & RISC-V - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/416497709)
- [Hardware - The Embedded Rust Book (rust-embedded.org)](https://docs.rust-embedded.org/book/start/hardware.html)
- [rust-embedded/cortex-m-quickstart: Template to develop bare metal applications for Cortex-M microcontrollers (github.com)](https://github.com/rust-embedded/cortex-m-quickstart)
- [Program your microcontrollers from WSL2 with USB support - Golioth](https://blog.golioth.io/program-mcu-from-wsl2-with-usb-support/)
