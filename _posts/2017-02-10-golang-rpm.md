---
date: 2017-02-10
layout: post
title: Golang项目RPM打包总结
categories: 技术
tags: centos7 golang rpm
---

## Introduction

由于Golang的包管理机制相对简单，导致当项目依赖复杂时，RPM打包就变得异常繁琐，本文主要总结RPM打包时的SPEC文件和编译脚本的编写。

## Prepare

* RPM打包环境搭建

```sh
#Golang编译环境搭建参考
[Docker编译环境](https://xiaoweiqian.github.io/note/docker-compile-on-centos7/)

# 安装RPM依赖
yum install rpmdevtools

#生成RPM目录
cd ~
rpmdev-setuptree
```

## Script

* SPEC脚本文件

编写SPEC文件，存放到~/rpmbuild/SPECS 目录

```sh
# RPM包和源码包名称
Name: docker-macvlan
# RPM包版本
Version: %{_version}
# RPM包发布号
Release: %{_release}%{?dist}
# 源码包名称
Source: %{name}.tar.gz

# 以下这些信息可随意搭配填写
Summary: The open-source application container engine
Group: Tools/Docker
License: ASL 2.0
URL: https://dockerproject.org
Vendor: xiaowei
Packager: xiaowei

# 判断服务模式
# is_systemd conditional
%if 0%{?fedora} >= 21 || 0%{?centos} >= 7 || 0%{?rhel} >= 7 || 0%{?suse_version} >= 1210
%global is_systemd 1
%endif

# 处理依赖
# required packages for build
# only require systemd on those systems
%if 0%{?is_systemd}
%if 0%{?suse_version} >= 1210
BuildRequires: systemd-rpm-macros
%{?systemd_requires}
%else
%if 0%{?fedora} >= 25
# Systemd 230 and up no longer have libsystemd-journal (see https://bugzilla.redhat.com/show_bug.cgi?id=1350301)
BuildRequires: pkgconfig(systemd)
Requires: systemd-units
%else
BuildRequires: pkgconfig(systemd)
Requires: systemd-units
BuildRequires: pkgconfig(libsystemd-journal)
%endif
%endif
%else
Requires(post): chkconfig
Requires(preun): chkconfig
# This is for /sbin/service
Requires(preun): initscripts
%endif

# required packages on install
Requires: /bin/sh
Requires: iptables
%if !0%{?suse_version}
Requires: libcgroup
%else
Requires: libcgroup1
%endif
Requires: tar
Requires: xz
%if 0%{?fedora} >= 21 || 0%{?centos} >= 7 || 0%{?rhel} >= 7 || 0%{?oraclelinux} >= 7
# Resolves: rhbz#1165615
Requires: device-mapper-libs >= 1.02.90-1
%endif
%if 0%{?oraclelinux} >= 6
# Require Oracle Unbreakable Enterprise Kernel R4 and newer device-mapper
Requires: kernel-uek >= 4.1
Requires: device-mapper >= 1.02.90-2
%endif

# DWZ problem with multiple golang binary, see bug
# https://bugzilla.redhat.com/show_bug.cgi?id=995136#c12
%if 0%{?fedora} >= 20 || 0%{?rhel} >= 7 || 0%{?oraclelinux} >= 7
%global _dwz_low_mem_die_limit 0
%endif

%description
Docker macvlan driver for swarmkit.

%prep
%if 0%{?centos} <= 6 || 0%{?oraclelinux} <=6
%setup -n %{name}
%else
%autosetup -n %{name}
%endif

# 执行编译脚本
%build
PKG_DIR=%{_pkgdir} ./make.sh

# 执行打包后检查
%check
./docker-macvlan-%{_origversion} -v

#执行编译后安装
%install
# install binary
install -d $RPM_BUILD_ROOT/%{_bindir}
install -p -m 755 ./docker-macvlan-%{_origversion} $RPM_BUILD_ROOT/%{_bindir}/docker-macvlan

#install service
install -d $RPM_BUILD_ROOT/%{_unitdir}
install -p -m 644 docker-macvlan.service $RPM_BUILD_ROOT/%{_unitdir}/docker-macvlan.service

# list files owned by the package here
%files
/%{_bindir}/docker-macvlan
/%{_unitdir}/docker-macvlan.service

%post
%systemd_post docker-macvlan

%preun
%systemd_preun docker-macvlan

%postun
%systemd_postun_with_restart docker-macvlan

%changelog

```

* 编译脚本文件

编译脚本用以打包RPM时调用，GOlang打包编译时由于rpmbuild不在GOPATH目录中，会出现各种编译问题，目前想到的解决办法如下。

1. 动态修改GOPATH   
在rpmbild中对应编译目录创建隐式文件夹.gopath，并把当前环境GOPATH软连接到.gopath文件夹，再用.gopath的绝对路径重写GOPATH。   

2. 使用相对路径，指定源码路径为编译目录   
rpmbuild中的源码不能直接用于编译，需要指定.gopath中源码路径为编译目录。   

```sh
#!/bin/bash

VERSION="0.1"
PKGDIR=./
if [ "$PKG_DIR" ]; then
    echo "set gopath"
    # 创建.gopath目录
    rm -rf .gopath
    mkdir -p .gopath
    # 软连接源码并重写GOPATH
    ln -sf $GOPATH/src .gopath/
    export GOPATH="${PWD}/.gopath"
    echo $GOPATH

    # 获取源码编译目录的相对路径
    PKGDIR=./.gopath/src/${PKG_DIR##*src}
fi

# 指定源码相对路径编译
go build -o ./docker-macvlan-$VERSION $PKGDIR
```

* 打包脚本文件

package-rpm.sh 脚本文件，在工程源码目录执行此脚本开始打包。

```sh
#!/bin/bash

VERSION="0.1"
rpmName=docker-macvlan
rpmVersion=$VERSION
rpmRelease="1.sn"

# 传入源码目录
pkgDir=${PWD}

echo "Package start."
echo "rpmVersion="$rpmVersion
echo "rpmRelease="$rpmRelease
# 拷贝spec到编译目录
cp ./docker-macvlan.spec /root/rpmbuild/SPECS/
# 打包压缩源码
cp -r ../macvlan-driver ../${rpmName}
tar --exclude .git -zcf /root/rpmbuild/SOURCES/${rpmName}.tar.gz -P ../${rpmName}
rm -rf ../${rpmName}
# 执行打包，可以通过--define 指定传入参数
rpmbuild -ba \
            --define "_release $rpmRelease" \
            --define "_version $rpmVersion" \
            --define "_origversion $VERSION" \
            --define "_pkgdir $pkgDir" \
            /root/rpmbuild/SPECS/${rpmName}.spec

echo "Package end."
```

End
