---
title: Node
date: 2017-03-15 10:24:07

categories:
- 技术

tags:
- node
---

## 安装
先去官网下载[源码][1]

```
//首先确保系统安装来python,gcc,g++,如果没有则安装：
sudo apt-get install python
sudo apt-get install build-essential
sudo apt-get install gcc
sudo apt-get install g++

//默认安装:
./configure //可以指定路径 ./configure –prefix=/opt/local/node
make
sudo make install
```

## npm升级
```
curl -0 -L https://npmjs.com/install.sh | sudo sh
//或者
curl -L https://npmjs.org/install.sh | sh
```

## npm国内镜像源
```
npm config set registry https://registry.npm.taobao.org --global
npm config set disturl https://npm.taobao.org/dist --global
```
也可以配置cnpm,具体可取ｎｐｍ官网查看

## node升级
node有一个模块叫n，是专门用来管理node.js的版本的
```
//首先安装n模块：
npm install -g n

//修改n的配置文件N_PREFIX为你的安装路径
N_PREFIX=${N_PREFIX-/opt/local/node/lib}

//使用或安装最新的官方发布：
n latest

//使用或安装稳定的正式版本：
n stable

//使用或安装最新的LTS正式版本：
n lts
```
 * 注意n安装了以后使用node要选择n安装路径下面的node来使用
[点击查看更多命令][2]

[1]: https://nodejs.org/en/download/
[2]: https://www.npmjs.com/package/n
