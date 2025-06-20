---
layout: post
title: 1_环境搭建hello_world
category: coder
typora-root-url: ../../../..
---

本系列假设使用者已经有较多的C/C++开发经验



### 安装rust

```shell

# rust安装完毕, 手动添加到PATH
vi ~/.zshrc.local   
# 添加：
# PATH="$HOME/.cargo/bin:$PATH"
# export RUSTUP_DIST_SERVER=https://mirrors.ustc.edu.cn/rust-static
# export RUSTUP_UPDATE_ROOT=https://mirrors.ustc.edu.cn/rust-static/rustup
source ~/.zshrc

# 下载并安装rustup工具, 它会自动提示安装rust相关工具集
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
# 安装完毕后刷新环境变量, 这样才会有rustup等命令
source ~/.cargo/env

# 检查是否安装成功. 以下列举安装的一系列工具 
# rustc是编译器, 相当于gcc
rustc --version
# rustfmt是rust提供的代码整理规范化工具
rustfmt --version
# cargo, rust项目构建工具和包管理工具. 尤其是包管理, 是C/C++中没有的!
cargo --version

# 添加国内包构建加速
cat <<EOF > ~/.cargo/config 
[source.crates-io]
registry = "https://github.com/rust-lang/crates.io-index"
replace-with = 'ustc'
[source.ustc]
registry = "git://mirrors.ustc.edu.cn/crates.io-index"
EOF
```

我们的hello world程序就用cargo来构建

```shell
cargo new ch01_hello
cd ch01_hello
tree -L 2 -a
'''
.
├── Cargo.toml
├── .git
│   ├── config
│   ├── description
│   ├── HEAD
│   ├── hooks
│   ├── info
│   ├── objects
│   └── refs
├── .gitignore
└── src
    └── main.rs
'''
```

可以看到cargo定义的项目框架, 默认会初始化git仓库并添加默认的.gitignore, 这点考虑的很到位. cargo使用的配置文件是Cargo.toml:

```toml
[package]
name = "ch01_hello"
version = "0.1.0"
authors = ["flyimx <flyimx@gmail.com>"]
edition = "2018"

[dependencies]
```

创建的rust源文件是main.rs

```rust
fn main() {
    println!("Hello, world!");
}
```

rust语言规范接近C语言规范, 主要有如下几点:

* main函数依然是整个程序的入口.
* 在println函数后面有个`!`号, 代表调用的是宏而不是函数, 后面有介绍
* 行结尾还是需要`;`
* rust风格的缩进是4个空格, 不是tab

接下来就是构建cargo项目

```shell
# 快速检查代码, 保证代码可被正确编译
cargo check
# 编译代码, 生成二进制文件在./target/debug/目录下
# 第一次还会生成Cargo.lock, 该文件是cargo自动做包依赖版本跟踪的, 一般我们不需要改动它
cargo build

# 生成正式版目标文件, 经过优化, 运行更快, 但编译耗时也会变长
cargo build --release

# 一键编译并运行
cargo run
```

## 离线安装



## 编译加速

### 可观测性

首先，我们需要识别造成编译缓慢的瓶颈在哪里，使用如下命令执行编译，会在浏览器中打开HTML界面显示程序各个 crate 之间的依赖关系及其编译耗时：
```shell
cargo build --timings
```

### 优化crate构建速度(不修改代码)
主要有如下方法：
1、使用sccache引入编译缓存，加速编译。
```shell
#1. 安装
cargo install sccache

#2. 修改 cargo 的配置文件，加入rustc指令封装
...

[build]
rustc-wrapper = "/home/USERNAME/.cargo/bin/sccache"
...

#3. 后续编译第三方库就会生产缓存保存到 ~/.cache/sccache 目录
```
2、使用`cargo-udeps`检查无用的依赖并移除
3、对于cargo依赖分析获取crates.io-index采用稀疏索引。
```shell
[registries.crates-io]
protocol = "sparse"
```
4、优化链接时速度，选择比默认链接器更快的链接器，如`mold`。
5、优化crate的相关配置，减少构建动作。


### 优化代码组织结构
通过重构优化代码的组织结构，也可以一定程度加速构建。主要做法如下：
1、拆分到更合理的 crates 规模，可以帮助 Rust 更快地处理代码编译。
2、使用`cargo nextest`更快的运行测试用例。
3、【慎用！】将单元测试整合到一个二进制文件，并关掉 tests mod 和 docs tests 。该方法不优雅。

