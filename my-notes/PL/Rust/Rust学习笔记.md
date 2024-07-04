### Rust register管理

[wtklbm/crm: Cargo registry manager (Cargo 注册表管理器)，用于方便的管理和更换 Rust 国内镜像源 (github.com)](https://github.com/wtklbm/crm)

### VSCode Rust analyzer源码级编译安装

用这个可以自动切换镜像源，一行命令就可以 https://github.com/wtklbm/crm

```bash
cargo install crm
crm best
```

源码编译安装RA：

```bash
$ git clone https://github.com/rust-analyzer/rust-analyzer.git && cd rust-analyzer
$ cargo xtask install # Both server and code plugin
$ cargo xtask install --server # If you’re not using Code, you can compile and install only the LSP server:
```

这篇文章有修改，因为RA的repo已经移动到官方维护了。所以可能安装VSCode的插件就自带了，因为我新装的WSL2没有安装RA，直接下载RA的VSCode扩展就可以用了，然后用起来更舒服了。

 https://rust-analyzer.github.io/manual.html#building-from-source



