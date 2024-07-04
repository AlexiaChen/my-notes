
在Linux ubuntu上安装Pritunl这个VPN Client

```bash
sudo tee /etc/apt/sources.list.d/pritunl.list << EOF
deb https://repo.pritunl.com/stable/apt focal main
EOF

sudo apt --assume-yes install gnupg
sudo gpg --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 7568D9BB55FF9E5287D586017AE645C0CF8E292A
sudo gpg --armor --export 7568D9BB55FF9E5287D586017AE645C0CF8E292A | sudo tee /etc/apt/trusted.gpg.d/pritunl.asc
sudo apt update
# 其中一个是命令行，一个是图形界面的client
sudo apt install pritunl-client-electron pritunl-client
```

[Installation (pritunl.com)](https://docs.pritunl.com/docs/installation-client)

[Pritunl Client - Open Source OpenVPN Client](https://client.pritunl.com/#install)

