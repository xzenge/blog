---
title: CSAPP Docker实验环境搭建
date: 2020-10-11 17:57:26
tags: 
    - CSAPP
    - C
categories :
    - Technology
---

CSAPP 的Docker实验环境在windows下的安装过程，MacOS下一致。
<!-- more -->
### 搜索ubuntu镜像
``` shell
docker search ubuntu
```

{% asset_img search.png 800 800 search %}

### 拉取最新镜像
``` shell
docker pull ubuntu:latest
```

{% asset_img pull.png 800 800 pull %}

### 配置并启动ubuntu容器
1. -it 中断交互式
2. -v 共享目录最好指定自己的的实验目录 (D:\work\workspace_c\CSAPP_LAB:csapplab)
3. -name 容器名称
4. /bin/bash 容器启动后运行bash
``` shell
docker run -it -v D:\work\workspace_c\CSAPP_LAB:/csapplab --name=csapp_env ubuntu:latest /bin/bash
```

### 配置ubuntu实验环境
1. 更新apt软件源
``` shell
apt-get update
```

2. 安装sudo
``` shell
apt-get instull sudo
```

3. 安装vim
``` shell
apt-get instull vim
```

4. 安装c/c++编译环境
``` shell
sudo apt-get install build-essential
sudo apt-get install gcc-multilib
```

### 退出容器
``` shell
exit
```

### 停止容器
``` shell
docker stop csapp_env
```

### 启动&进入容器
``` shell
docker start csapp_env
docker exec -it csapp_env /bin/bash
```