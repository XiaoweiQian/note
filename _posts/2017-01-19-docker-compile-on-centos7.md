---
date: 2017-01-19
layout: post
title: Centos7下搭建Docker编译环境
categories: 技术
tags: centos7 docker
---

## Introduction

Docker 官方编译环境是基于ubuntu，而国内生产环境大多是Centos7，因此需要在Centos7下搭建编译环境。

## Prepare

* GO 1.7.3 环境安装

go 1.7.3 binary [go1.7.3 download](http://www.golangtc.com/static/go/1.7.3/go1.7.3.linux-amd64.tar.gz)

```sh
# 解压tar到/usr/local/
tar zxf -C /usr/local

# 创建go 源码目录
mkdir -p /opt/gocode

#配置环境变量
sudo tee ~/.profile <<-'EOF'
export GOROOT=/usr/local/go
export GOPATH=/opt/gocode
export PATH=$PATH:$GOROOT/bin:$GOPATH/bin
EOF

#生效
source ~/.profile
go env
```

* 在Centos7中只能动态编译Docker源码，需要依赖如下组件

```sh
# 增加Docker源
sudo tee /etc/yum.repos.d/docker.repo <<-'EOF'
[dockerrepo]
name=Docker Repository
baseurl=https://yum.dockerproject.org/repo/main/centos/7/
enabled=1
gpgcheck=1
gpgkey=https://yum.dockerproject.org/gpg
EOF

# 安装最新版本Docker
yum install docker-engine

# 安装文件系统依赖
yum install btrfs-progs-devel device-mapper-devel
```

## Compile

* git clone docker源码到go 源码目录并编译

```sh
mkdir -p /opt/gocode/src/github.com/docker/
cd /opt/gocode/src/github.com/docker/
git clone https://github.com/docker/docker.git

# 进入docker工程目录并编译
cd /opt/gocode/src/github.com/docker/docker
AUTO_GOPATH=1 ./hack/make.sh dynbinary
```

* 安装编译后的docker client 和 docker daemon

```sh
# stop docker
systemctl stop docker

#install
cp ./bundles/1.13.0/dynbinary-client/docker-1.13.0 /usr/bin/docker
cp ./bundles/1.13.0/dynbinary-daemon/dockerd-1.13.0 /usr/bin/dockerd

#start docker
systemctl start docker
```
