---
date: 2017-01-23
layout: post
title: Docker network remote drivrer 开发介绍
categories: 技术
tags: docker libnerwork
---

## Introduction

Docker network模型(CNM)设计了一种可插拔式架构，官方实现为Libnetwork，它为Docker daemon 和network driver之间提供了接口，因此libnetwork相当于一个controller负责对接driver和daemon。network driver按提供方分为原生和第三方，按作用域分为local和global，而其原生的macvlan driver默认为local作用域，本文主要介绍如何使用其接口开发基于global作用域的macvlan driver。

## Prepare

* 开发和编译环境

1.开发环境搭建参考如下

[使用vagrant搭建统一开发环境](https://xiaoweiqian.github.io/note/vagrant/)

2.编译环境搭建参考如下

[Docker编译环境](https://xiaoweiqian.github.io/note/docker-compile-on-centos7/)

## Develop

* 目录结构

driver源码路径为$GOPATH/src/cnsuning/macvlan-driver，工程结构如下：

```
--macvlan-driver   
    --drivers/         #源码主目录
        --macvlan.go   #remote driver api 具体实现   
        ...   
    --Godeps/          #golang 包管理工具godep配置目录   
        --Godeps.json  #godep依赖管理文件   
    --utils/           #工具包代码目录   
        --netutils/    
    --vendor/          #第三方依赖包目录，由godep自动维护   
        --github.com/   
        --golang.org/   
    --main.go          #driver 启动入口   
    --Vagrantfile      #开发环境vagrant配置文件   
```

* 包管理godep

golang自带有包管理工具go get，通过go get可以将依赖的第三方库下载到GOPATH目录，在代码中直接import相应的代码库就可以了。但是还存在一些不足，比如：

1. 通常我们会在本地开发多个项目，所有项目共同使用GOPATH中的第三方库。

2. 因为在项目的版本管理里没有存放第三方库的代码，其他人下载下来的时候要重新go get所有依赖库。

3. 假如我们换了一台电脑开发，要重新下载依赖库。

上面的问题总结起来其实就是项目引用的第三方库没有列入到项目里面，如果把第三方库直接放到项目里就能解决这些问题了，而Godep这个工具，就是用来解决这个问题的。

```sh
#安装godep
go get github.com/tools/godep

#把第三方包纳入管理，会把所有的依赖包存到vendor目录，并生成Godeps.json文件
godep save

#加载vendor中第三方包运行，加载优先级高于GOPATH
godep go run main.go

#加载vendor中第三方包编译
godep go build main.go

```

* Docker 插件机制说明

docker 提供了一套第三方插件的开发规范，具体内容如下:

1. 插件发现   
当用户或者容器尝试通过插件名称使用插件的时候docker就从插件目录查找并发现插件。  
有如下三种类型的文件可以放在插件目录中:   
.sock 文件是UNIX域套接字。   
.spec 文件是包含URL（如 unix:///other.sock 或者 tcp://localhost:8080）的文本文件。   
.json 文件是包含了插件的完整参数的文本文件。   
使用UNIX域套接字的插件必须和docker运行在同一台主机上，而使用spec和json文件并且文件中指定远程URL的插件可以运行在任意主机上。   
UNIX域套接字文件必须放在/run/docker/plugins下面，而spec文件可以放在/etc/docker/plugins 或者/usr/lib/docker/plugins下,文件的名称（不包括扩展名）决定了插件的名。   
Docker总是先在/run/docker/plugins目录中搜索UNIX域套接字，如果域套接字不存在则在/etc/docker/plugins和/usr/lib/docker/plugins下检查spec文件和json文件,一旦找到给定插件就停止目录扫描。 

2. 插件生命周期   
插件应该在docker daemon启动前启动，在docker daemon停止后再停止,当使用systemd的服务打包一个插件的时候，应该使用systemd依赖去管理启动和停止的顺序。   
升级插件的时候要先停止docker daemon，升级插件，然后再启动docker daemon。

3. 插件激活   
当插件第一次被引用的时候（不管是用户通过插件名引用<如 docker network create --driver=macvlan>还是启动一个配置了插件的容器），docker在插件目录查找指定插件并且通过握手来激活它。插件在docker daemon启动的时候不会自动激活，而是在需要的时候即刻激活。

4. Systemd 激活   
插件也能够通过systemd进行套接字激活,官方插件帮助原生支持套接字激活。为了使用套接字激活需要一个service文件和一个socket文件。   
service文件(例如/lib/systemd/system/your-plugin.service)   
```sh
[Unit]
Description=Your plugin
Before=docker.service
After=network.target your-plugin.socket
Requires=your-plugin.socket docker.service

[Service]
ExecStart=/usr/lib/docker/your-plugin

[Install]
WantedBy=multi-user.target

#socket文件(例如/lib/systemd/system/your-plugin.socket)
[Unit]
Description=Your plugin

[Socket]
ListenStream=/run/docker/plugins/your-plugin.sock

[Install]
WantedBy=sockets.target
```
这就确保了当docker daemon连接socket的时候插件已经启动完成了（例如daemon第一次使用sockets或者插件意外崩溃）。

5. API 设计   
插件API是运行于HTTP之上的JSON格式的远程过程调用，类似于webhooks，请求从docker daemon流向插件，因此插件需要实现HTTP服务端，并且服务端绑定在“插件发现”小节中提到的UNIX套接字上。   
所有的请求都是HTTP POST请求，API通过Accept头提供版本号, 目前总被设置成 application/vnd.docker.plugins.v1+json。

6. 插件助手   
为了简化插件开发，官方在docker/go-plugins-helpers为docker当前支持的每种插件都提供了sdk，当需要开发第三方插件时需要引入此包。   
关于以上插件机制说明的原文链接:[Docker Plugin API](http://dockone.io/article/1297)

* Libnetwork remote driver api 实现说明

实现docker/go-plugins-helpers/network/api.go中Driver API，具体API如下：

1. GetCapabilities() (\*CapabilitiesResponse, error)   
*network创建时被调用，返回driver scope。   
参数:   
null   
返回:   
CapabilitiesResponse - driver scope信息GlobalScope 或者 LocalScope   
error - 异常信息*

2. AllocateNetwork(\*AllocateNetworkRequest) (\*AllocateNetworkResponse, error)   
*Driver scope 为 global时，swarm manager 创建network络时调用，保存信息到global store(swarm 默认实现为etcd)。   
参数:   
AllocateNetworkRequest - 需要创建的network，包括sunbnet、gateway、IPrange和Options等数据   
返回:   
AllocateNetworkResponse - 经过解析处理的Options信息   
error - 异常信息*

3. FreeNetwork(\*FreeNetworkRequest) error   
*Driver scope 为 global时，swarm manager 删除network会调用，会从global store删除指定network。   
参数:   
FreeNetworkRequest - 需要删除的network ID   
返回:   
error-异常信息*

4. CreateNetwork(\*CreateNetworkRequest) error   
*swarm node 创建网络时调用，在node节点创建network。   
参数:   
CreateNetworkRequest - 需要创建的network，包括sunbnet、gateway、IPrange和Options等数据   
返回:    
error - 异常信息*

5. DeleteNetwork(\*DeleteNetworkRequest) error   
*swarm node 删除网络时调用，在node节点删除指定network。   
参数:   
DeleteNetworkRequest - 需要删除的network ID   
返回:   
error - 异常信息*

6. CreateEndpoint(\*CreateEndpointRequest) (\*CreateEndpointResponse, error)   
*swarm node 创建container时调用，在node节点上生成endpoint并把数据写入driver localstore。   
参数:   
CreateEndpointRequest - 需要创建的endpoint，包含network id、endpoint id和ip等数据   
返回:   
CreateEndpointResponse - container interface 包含ip、gateway和mac等数据   
error - 异常信息*

7. DeleteEndpoint(\*DeleteEndpointRequest) error   
*swarm node 删除container时调用，在node节点上删除endpoint并从driver localstore中remove相关数据。   
参数:   
DeleteEndpointRequest - 需要删除的endpoint，包含network id 和 endpoint id 数据   
返回:   
error - 异常信息*

8. EndpointInfo(\*InfoRequest) (\*InfoResponse, error)   
*swarm manager 查询endpoint 时调用，返回endpoint的自定义数据。   
参数:   
InfoResponse - 需要查询的endpoint，包含network id和endpoint id数据   
返回:    
InfoResponse - endpoint 额外的用户自定义数据   
error - 异常信息*

9. Join(\*JoinRequest) (\*JoinResponse, error)   
container start时调用，把endpoint(macvlan device)加入对应的sandbox(network namespace)。   
参数:   
JoinRequest - 需要启动的container，包含network id、endpoint id、sandox和Options数据   
返回:   
JoinResponse - 启动后的container，包含macvlan device name、gateway和container interface name数据   
error - 异常信息*

10. Leave(\*LeaveRequest) error   
*container stop时调用，把endpoint(macvlan device)移出对应的sandbox(network namespace)。   
参数:   
LeaveRequest - 需要移出的endpoint，包含network id和endpoint id数据   
返回:   
error - 异常信息*

11. DiscoverNew(\*DiscoveryNotification) error   
*swarm manager 发现事件通知后续处理策略，例如:集群中新增加一个节点。   
参数:   
DiscoveryNotification - 发现事件通知，包含事件类型和事件数据   
返回:   
error - 异常信息*

12. DiscoverDelete(\*DiscoveryNotification) error   
*swarm manager 删除事件通知后续处理策略，例如:集群中新移除一个节点。   
参数:
DiscoveryNotification - 删除事件通知，包含事件类型和事件数据   、
返回:   
error - 异常信息*

13. ProgramExternalConnectivity(\*ProgramExternalConnectivityRequest) error   
*container endpoint join完成后调用，配置endpoint额外的网络信息，例如:L4 Data。   
参数:   
ProgramExternalConnectivityRequest - 指定endpoint，包含network id、endpoint id、sandox和Options数据   
返回:   
error - 异常信息*

14. RevokeExternalConnectivity(\*RevokeExternalConnectivityRequest) error   
*container endpoint leave前调用，删除endpoint额外的网络信息，例如:L4 Data。   
参数:    
RevokeExternalConnectivityRequest - 指定endpoint，包含network id、endpoint id、sandox和Options数据   
返回:   
error - 异常信息*

## Test

golang内置了testing包用于执行相关测试，并引入第三方测试包"github.com/stretchr/testify/assert"，用于简化case编写复杂度。

* Test Case 编写

Case文件golang best practice建议直接与待测方法文件放在同一个包下并加上“_test”后缀，case编写示例:

```go
import (
	"net"
	"testing"

	"github.com/stretchr/testify/assert"
)

func TestGenerateRandomMAC(t *testing.T) {
	mac1 := GenerateRandomMAC()
	mac2 := GenerateRandomMAC()
	assert.NotEqual(t, mac1.String(), mac2.String())
}

```


* Test Case 执行

golang内置了 go test 工具用于执行相关测试，示例:

```sh
# 执行drivers包中所有test case
go test -v ./drivers

# 执行drivers包中所有匹配的test case，并显示覆盖率
go test -v -cover./drivers -run=TestGen

# 结果 
=== RUN   TestGenerateRandomMAC
--- PASS: TestGenerateRandomMAC (0.00s)
PASS
coverage: 12.5% of statements
ok  	github.com/XiaoweiQian/macvlan-driver/utils/netutils	0.004s
```

End
