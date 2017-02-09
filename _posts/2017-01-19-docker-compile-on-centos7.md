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

# 安装RPM依赖
yum install rpmdevtools

#安装selinux
yum install selinux-policy-devel

#安装go-md2man
yum install go-md2man
```

## Compile

此章节内容适合在开发测试环境中手动编译和部署，如果需要发布到生产环境直接参照下一章节打包RPM

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

## RPM package

* 生成RPM目录

```sh
cd ~
rpmdev-setuptree
```

* 修改docker默认spec

```sh
cd /opt/gocode/src/github.com/docker/docker
vi ./hack/make/.build-rpm/docker-engine.spec

#打开注释，修改如下:
./man/md2man-all.sh

#修改默认docker bin路径,由/usr/local/bin/ 修改为/usr/bin,例如:
install -p -m 755 /usr/bin/docker-proxy $RPM_BUILD_ROOT/%{_bindir}/docker-proxy

```

* 编写打包脚本

```sh
cd /opt/gocode/src/github.com/docker/docker

sudo tee ./hack/package-rpm <<-'EOF'
#!/bin/bash

VERSION="1.13.0"
rpmName=docker-engine
rpmVersion=$VERSION
rpmRelease="1.sn"

DOCKER_GITCOMMIT=$(git rev-parse --short HEAD)
if [ -n "$(git status --porcelain --untracked-files=no)" ]; then
    DOCKER_GITCOMMIT="$DOCKER_GITCOMMIT-unsupported"
fi

echo "Package start."
echo "rpmVersion="$rpmVersion
echo "rpmRelease="$rpmRelease
cp ./hack/make/.build-rpm/* /root/rpmbuild/SPECS/
cp -r ../docker ../${rpmName}
tar --exclude .git -zcf /root/rpmbuild/SOURCES/${rpmName}.tar.gz -P ../${rpmName}
tar -zcf /root/rpmbuild/SOURCES/${rpmName}-selinux.tar.gz -C ./contrib/selinux/ ${rpmName}-selinux
rm -rf ../${rpmName}
rpmbuild -ba \
            --define "_gitcommit $DOCKER_GITCOMMIT" \
            --define "_release $rpmRelease" \
            --define "_version $rpmVersion" \
            --define "_origversion $VERSION" \
            /root/rpmbuild/SPECS/${rpmName}.spec

rpmbuild -ba \
                    --define "_gitcommit $DOCKER_GITCOMMIT" \
                    --define "_release $rpmRelease" \
                    --define "_version $rpmVersion" \
                    --define "_origversion $VERSION" \
                    /root/rpmbuild/SPECS/${rpmName}-selinux.spec

echo "Package end."
EOF

chmod +x ./hack/package-rpm
```

* 打包

```sh
cd /opt/gocode/src/github.com/docker/docker/
./hack/package-rpm
#RPM包默认目录为/root/rpmbuild/RPMS
```

End
