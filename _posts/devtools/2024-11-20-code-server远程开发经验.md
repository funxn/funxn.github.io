---
layout: post
title: code-server远程开发经验
category: system
typora-root-url: ../..
---

## 部署

### 安装code-server
直接使用ghproxy加速下载并安装：
```bash
#下载
VERSION=4.95.2
curl -fOL https://github.com/coder/code-server/releases/download/v${VERSION}/code-server_${VERSION}_amd64.deb 
#安装
sudo dpkg -i code-server_${VERSION}_amd64.deb
#启动
sudo systemctl enable --now code-server@$USER
```

### 配置code-server
启动后就可以查看默认的配置了
cat ~/.config/code-server/config.yaml

使用编辑器修改配置，如下：
```bash
bind-addr: 0.0.0.0:24041 # 0.0.0.0表示在所有网口上工作
auth: password
password: 123456 # 登录密码
cert: true
```

重启服务以加载配置
sudo systemctl restart code-server@$USER

之后可以通过浏览器登录，连接code-server进行操作。

## 问题处理记录

### https证书安全性问题，无法加载图片/preview等。
开启https后，默认情况下，code-server会生成自签名证书，导致浏览器无法加载图片等资源，因此需要手动生成证书。
我们使用mkcert工具生成证书，并配置到code-server中。

```bash
#安装mkcert
sudo apt install -y mkcert

#生成证书
mkcert -cert-file mycode.crt -key-file mycode.key <自签证书的公网ip> 127.0.0.1                                              

# 将证书复制到code-server配置目录
sudo cp mycode.crt ~/.config/code-server/
sudo cp mycode.key ~/.config/code-server/

# 修改配置文件
sudo vim ~/.config/code-server/config.yaml
```
修改配置文件，将cert: true改为cert: ~/.config/code-server/mycode.crt，将key: ~/.config/code-server/mycode.key，重启服务即可。

此时如果使用firefox，已经可以正常访问。但通过chrome访问时，会提示不安全的证书，需要将mkcert的CA证书安装到访问的主机上。
1. 通过mkcert命令查找CA证书位置：mkcert -CAROOT
2. 进入对应文件夹，找到 rootCA.pem文件，将其后缀修改为 .crt，拷贝到客户端。
3. 在windows中可以通过双击安装。安装时，要选择受信任的根证书颁发机构。
4. 重启chrome，访问code-server，此时可以正常访问。

### 配置c++开发环境
1. 安装依赖软件
```bash
sudo apt install -y build-essential gcc g++ cmake cmake-preset clang-format ninja
```

2. 安装C/C++插件：由于code server中不含这个插件，直接在插件中找不到，需要手动安装。
* 下载：`curl -fOL https://gh-proxy.com/github.com/microsoft/vscode-cpptools/releases/download/v1.22.11/cpptools-linux-x64.vsix`
* 安装：使用“从VSIX安装”。

3. 安装其他依赖插件：
* CMake
* CMake tools
* GitLens
* CodeGeeX：AI代码补全

### 登录页面自定义
1. 在code-server/src/browser/pages下就是登录页面的代码信息
2. 修改login.html，login.css文件，根据自己的需要修改 