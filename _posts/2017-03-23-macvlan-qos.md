---
date: 2017-03-23
layout: post
title: 容器网络QoS设计可行性分析
categories: 技术
tags: docker macvlan qos
---

## 简介

目前docker 17.03.0-ce版本，swarm service中只支持cpu和memory的QoS功能，对于网络QoS功能还没看到社区相关的开发计划，本文主要分析在macvlan网络中实现QoS功能的可行性。

## 思路

从下面的macvlan网络架构图中可以分析出，容器网络的QoS功能需要在容器内部的网络设备中实现，在此端可以对进出容器的网络数据包进行限速和调度。

![](https://raw.githubusercontent.com/XiaoweiQian/note/master/img/macvlan.png)

## 理论分析

容器中的网络设备是通过linux namespace技术来实现网络隔离，因此可以通过linux TC(traffic controll)框架对namespace中macvlan设备来实现QoS功能，其分为ingress(入口流量)和egress(出口流量)两部分。
入口部分主要用于进行入口流量限速(policing)，出口流量部分主要用于队列调度(queuing scheduling)，大多数排队规则(qdisc)都是用于输出方向。

* egress(出口流量)
    
    通常TC中要对网卡进行出口流量控制的配置，需要进行如下的步骤：

    * 为网卡配置一个队列；
    
    * 在该队列上建立分类；
        
    * 根据需要建立子队列和子分类；
        
    * 为每个分类建立过滤器。
    
    一般使用TC中的HTB(Hierarchical Token Bucket) 队列，与其他复杂的队列类型相比，HTB具有功能强大，配置简单等优点，而流量的处理由三种对象控制，分别是：qdisc，class，filter

    * QDISC (队列)
    
        QDisc 是理解流量控制的基础。无论何时，内核如果需要通过某个网络接口发送数据包，它都需要按照为这个接口配置的 qdisc 把数据包加入队列。然后内核会尽可能多得从 qdisc 里面取出数据包， 把它们交给网络适配器驱动模块。

    * CLASS (分类策略)
    
        class 用来表示控制策略，很多时候，可能要对不通的 IP 实行不同的流量控制策略， 这时候我们就要用不同的 class 来表示不同的控制策略了

    * FILTER (过滤器)
    
        filter 用来将用户划入到具体的控制策略中（即不同的class中），比如我们要对 a 和 b 两个 IP 实行不同的控制策略 A 和 B ， 这时我们可以用 filter 将 a 划入控制策略 A，将 b 划入控制策略 B， filter 划分的标志位可用 u32 打标功能或者 iptables 的 set-mark 功能来实现。

* ingress(入口流量)
    
    TC中默认网卡入口流量控制策略无队列机制，因此无法对入口流量进行分类调度，只能实现简单的限速。
    另外一种方法是通过TC可以把网卡的入口流量导入ifb虚拟设备，转化为ifb设备的出口流量，此时只需要在ifb虚拟设备上应用TC出口流量流控，就可以间接的实现网卡的入口流量限速。

## 理论验证

为了测试网卡入口流量和出口流量限速实际效果，做如下测试，首先准备相关数据，创建3个macvlan设备，veth0、veth1和veth2，并放入到3个namespce中ns0、ns1和ns2，分别设置IP为192.168.1.2、192.168.1.3和192.168.1.4

* ingress 无队列策略
    
    * 在192.168.1.4上做入口流量限速，配置步骤如下：

    ```
    # 增加ingress规则
    ip netns exec ns2 tc qdisc add dev veth2 ingress

    # 配置policy，限速5000kbps，带宽约为40Mbits
    ip netns exec ns2 tc filter add dev veth2 parent ffff: protocol ip prio 10 u32 match ip src 0.0.0.0/0 police rate 5000kbps burst 500k mtu 15k drop flowid :1

    # 启动iperf服务器端
    ip netns exec ns2 iperf -s -i 1 -w 416k
    
    # 在ns1，192.168.1.3中启动iperf客户端
    ip netns exec ns1 iperf -c 192.168.1.4 -i 1 -w 416k -t 60
    ```
    * 测试结果如下：
    
    在192.168.1.4上做的入口流量限速带宽在9m～20m之间波动，与参数设置的40m并不一致，由此可见没有队列机制做入口限速是不可靠的。
    
    * 192.168.1.4 服务器端接收流量：
    
    ```
    
    Server listening on TCP port 5001
    TCP window size:  416 KByte
    
    [  4] local 192.168.1.4 port 5001 connected with 192.168.1.3 port 56512
    [ ID] Interval       Transfer     Bandwidth
    [  4]  0.0- 1.0 sec  2.22 MBytes  18.6 Mbits/sec
    [  4]  1.0- 2.0 sec  1.68 MBytes  14.1 Mbits/sec
    [  4]  2.0- 3.0 sec  1.05 MBytes  8.83 Mbits/sec
    [  4]  3.0- 4.0 sec  1.18 MBytes  9.93 Mbits/sec
    [  4]  4.0- 5.0 sec  1.12 MBytes  9.38 Mbits/sec
    [  4]  5.0- 6.0 sec  1.08 MBytes  9.02 Mbits/sec
    [  4]  6.0- 7.0 sec  1.12 MBytes  9.38 Mbits/sec
    [  4]  7.0- 8.0 sec  1.16 MBytes  9.74 Mbits/sec
    [  4]  8.0- 9.0 sec  1.08 MBytes  9.02 Mbits/sec
    [  4]  9.0-10.0 sec  2.42 MBytes  20.3 Mbits/sec
    [  4]  0.0-10.4 sec  14.5 MBytes  11.6 Mbits/sec
    ```

    * 192.168.1.3 客户端发送流量：
    
    ```
    
    Client connecting to 192.168.1.4, TCP port 5001
    TCP window size:  416 KByte
    
    [  3] local 192.168.1.3 port 56512 connected with 192.168.1.4 port 5001
    [ ID] Interval       Transfer     Bandwidth
    [  3]  0.0- 1.0 sec  2.50 MBytes  21.0 Mbits/sec
    [  3]  1.0- 2.0 sec  1.75 MBytes  14.7 Mbits/sec
    [  3]  2.0- 3.0 sec  1.12 MBytes  9.44 Mbits/sec
    [  3]  3.0- 4.0 sec  1.12 MBytes  9.44 Mbits/sec
    [  3]  4.0- 5.0 sec  1.12 MBytes  9.44 Mbits/sec
    [  3]  5.0- 6.0 sec  1.12 MBytes  9.44 Mbits/sec
    [  3]  6.0- 7.0 sec  1.12 MBytes  9.44 Mbits/sec
    [  3]  7.0- 8.0 sec  1.00 MBytes  8.39 Mbits/sec
    [  3]  8.0- 9.0 sec  1.12 MBytes  9.44 Mbits/sec
    [  3]  9.0-10.0 sec  2.50 MBytes  21.0 Mbits/sec
    [  3]  0.0-10.2 sec  14.5 MBytes  11.9 Mbits/sec
    ```

* ingress ifb队列策略
    
    * 在192.168.1.4上做入口流量限速，虽然TC的ingress策略默认无队列机制，但是启用linux内核中的ifb模块，通过把流量导入到ifb队列中，可以间接实现入口流量限速，配置步骤如下：
    
    ```
    # 内核加载ifb模块
    modprobe ifb numifbs=1
    
    # ifb0设备放入namespace
    ip link set ifb0 netns ns2
    
    # 在namespace中启用ifb设备
    ip netns exec ns2 ip link set dev ifb0 up

    # ns2 创建网卡veth2的入口策略
    ip netns exec ns2 tc qdisc add dev veth2 handle ffff: ingress

    # 把veth2的入口流量导入ifb0
    ip netns exec ns2 tc filter add dev veth2 parent ffff: protocol ip u32 match u32 0 0 action mirred egress redirect dev ifb0

    # 创建htb队列，未分类的流量默认走子分类10
    ip netns exec ns2 tc qdisc add dev ifb0 root handle 1: htb default 10

    # 创建根分类策略，保证根分类及其子分类总和共有50mbit带宽,突发流量5MB
    ip netns exec ns2 tc class add dev ifb0 parent 1: classid 1:1 htb rate 50mbit ceil 50mbit burst 5m

    # 创建子分类策略，保证子分类10拥有10mbit带宽，突发流量1MB
    ip netns exec ns2 tc class add dev ifb0 parent 1:1 classid 1:10 htb rate 10mbit ceil 10mbit burst 1m

    # 创建子分类策略，保证子分类11拥有40mbit带宽，突发流量4MB
    ip netns exec ns2 tc class add dev ifb0 parent 1:1 classid 1:11 htb rate 40mbit ceil 40mbit burst 4m

    # 创建过滤器，源IP192.168.1.3客户端的入口流量走分类策略11
    ip netns exec ns2 tc filter add dev ifb0 protocol ip parent 1:0 prio 1 u32 match ip src 192.168.1.3 flowid 1:11

    # 启动iperf服务器端
    ip netns exec ns2 iperf -s -i 1 -w 416k
    
    # 在ns0和ns1中同时启动iperf客户端
    ip netns exec ns0 iperf -c 192.168.1.4 -i 1 -w 416k -t 60
    ip netns exec ns1 iperf -c 192.168.1.4 -i 1 -w 416k -t 60
    ```
    * 测试结果如下：
    
    在192.168.1.4上做的入口流量限速带宽总和在47mbit左右，其中192.168.1.3的入口流量在38mbit左右，192.168.1.2的入口流量在9mbit左右，这与设置的分类策略基本一致，由此可见通过ifb设备的入口流量导入可以比较好的满足入口限速需求。

    * 192.168.1.4 服务器端入口流量：
    
    ```
    Server listening on TCP port 5001
    TCP window size:  416 KByte
    
    [  4] local 192.168.1.4 port 5001 connected with 192.168.1.3 port 59026
    [ ID] Interval       Transfer     Bandwidth
    [  4]  0.0- 1.0 sec  4.55 MBytes  38.1 Mbits/sec
    [  5] local 192.168.1.4 port 5001 connected with 192.168.1.2 port 36168
    [  4]  1.0- 2.0 sec  4.54 MBytes  38.1 Mbits/sec
    [  5]  0.0- 1.0 sec   771 KBytes  6.31 Mbits/sec
    [  4]  2.0- 3.0 sec  4.53 MBytes  38.0 Mbits/sec
    [  5]  1.0- 2.0 sec  1.14 MBytes  9.58 Mbits/sec
    [  4]  3.0- 4.0 sec  4.53 MBytes  38.0 Mbits/sec
    [  5]  2.0- 3.0 sec  1.14 MBytes  9.55 Mbits/sec
    [  4]  4.0- 5.0 sec  4.52 MBytes  37.9 Mbits/sec
    [  5]  3.0- 4.0 sec  1.14 MBytes  9.55 Mbits/sec
    [  4]  5.0- 6.0 sec  4.55 MBytes  38.2 Mbits/sec
    [  5]  4.0- 5.0 sec  1.14 MBytes  9.56 Mbits/sec
    [  4]  6.0- 7.0 sec  4.53 MBytes  38.0 Mbits/sec
    [  5]  5.0- 6.0 sec  1.14 MBytes  9.57 Mbits/sec
    [  4]  7.0- 8.0 sec  4.53 MBytes  38.0 Mbits/sec
    [  5]  6.0- 7.0 sec  1.14 MBytes  9.53 Mbits/sec
    [  4]  8.0- 9.0 sec  4.50 MBytes  37.8 Mbits/sec
    [  5]  7.0- 8.0 sec  1.14 MBytes  9.52 Mbits/sec
    [  4]  9.0-10.0 sec  4.51 MBytes  37.8 Mbits/sec
    [  5]  8.0- 9.0 sec  1.14 MBytes  9.55 Mbits/sec
    [  4] 10.0-11.0 sec  4.52 MBytes  37.9 Mbits/sec
    [  5]  9.0-10.0 sec  1.14 MBytes  9.52 Mbits/sec
    [  4] 11.0-12.0 sec  4.50 MBytes  37.7 Mbits/sec
    [  5] 10.0-11.0 sec  1.14 MBytes  9.52 Mbits/sec
    ```
    
    * 192.168.1.3 客户端出口流量：
    ```
    Client connecting to 192.168.1.4, TCP port 5001
    TCP window size:  416 KByte
    
    [  3] local 192.168.1.3 port 59026 connected with 192.168.1.4 port 5001
    [ ID] Interval       Transfer     Bandwidth
    [  3]  0.0- 1.0 sec  5.00 MBytes  41.9 Mbits/sec
    [  3]  1.0- 2.0 sec  4.38 MBytes  36.7 Mbits/sec
    [  3]  2.0- 3.0 sec  4.62 MBytes  38.8 Mbits/sec
    [  3]  3.0- 4.0 sec  4.50 MBytes  37.7 Mbits/sec
    [  3]  4.0- 5.0 sec  4.50 MBytes  37.7 Mbits/sec
    [  3]  5.0- 6.0 sec  4.50 MBytes  37.7 Mbits/sec
    [  3]  6.0- 7.0 sec  4.62 MBytes  38.8 Mbits/sec
    [  3]  7.0- 8.0 sec  4.62 MBytes  38.8 Mbits/sec
    [  3]  8.0- 9.0 sec  4.38 MBytes  36.7 Mbits/sec
    [  3]  9.0-10.0 sec  4.62 MBytes  38.8 Mbits/sec
    [  3] 10.0-11.0 sec  4.38 MBytes  36.7 Mbits/sec
    ```

    * 192.168.1.2 客户端出口流量：
    ```
    Client connecting to 192.168.1.4, TCP port 5001
    TCP window size:  416 KByte
    
    [  3] local 192.168.1.2 port 36168 connected with 192.168.1.4 port 5001
    [ ID] Interval       Transfer     Bandwidth
    [  3]  0.0- 1.0 sec  1.25 MBytes  10.5 Mbits/sec
    [  3]  1.0- 2.0 sec  1.12 MBytes  9.44 Mbits/sec
    [  3]  2.0- 3.0 sec  1.00 MBytes  8.39 Mbits/sec
    [  3]  3.0- 4.0 sec  1.12 MBytes  9.44 Mbits/sec
    [  3]  4.0- 5.0 sec  1.25 MBytes  10.5 Mbits/sec
    [  3]  5.0- 6.0 sec  1.12 MBytes  9.44 Mbits/sec
    [  3]  6.0- 7.0 sec  1.12 MBytes  9.44 Mbits/sec
    [  3]  7.0- 8.0 sec  1.12 MBytes  9.44 Mbits/sec
    [  3]  8.0- 9.0 sec  1.12 MBytes  9.44 Mbits/sec
    [  3]  9.0-10.0 sec  1.12 MBytes  9.44 Mbits/sec
    [  3] 10.0-11.0 sec  1.12 MBytes  9.44 Mbits/sec
    ```

* egress
    
    * 在192.168.1.4上做出口流量限速，配置步骤如下：
    ```
    # 创建htb队列，未分类的出口流量默认走子分类10
    ip netns exec ns2 tc qdisc add dev veth2 root handle 1: htb default 10

    # 创建根分类策略，保证根分类及其子分类总和共有50mbit带宽,突发流量5MB
    ip netns exec ns2 tc class add dev veth2 parent 1: classid 1:1 htb rate 50mbit ceil 50mbit burst 5m

    # 创建子分类策略，保证子分类10拥有10mbit带宽，突发流量1MB
    ip netns exec ns2 tc class add dev veth2 parent 1:1 classid 1:10 htb rate 10mbit ceil 10mbit burst 1m

    # 创建子分类策略，保证子分类11拥有40mbit带宽，突发流量4MB
    ip netns exec ns2 tc class add dev veth2 parent 1:1 classid 1:11 htb rate 40mbit ceil 40mbit burst 4m

    # 创建过滤器，目的IP为192.168.1.3客户端的出口流量走分类策略11
    ip netns exec ns2 tc filter add dev veth2 protocol ip parent 1:0 prio 1 u32 match ip dst 192.168.1.3 flowid 1:11

    # 启动iperf服务器端
    ip netns exec ns2 iperf -s -i 1 -w 416k
    
    # 在ns0和ns1中同时启动iperf客户端
    ip netns exec ns0 iperf -c 192.168.1.4 -i 1 -w 416k -d
    ip netns exec ns1 iperf -c 192.168.1.4 -i 1 -w 416k -d
    ```
    * 测试结果如下：

    在192.168.1.4上做的出口流量限速带宽总和在47mbit左右，其中目的IP为192.168.1.3出口流量在37.5mbit左右，目的IP为192.168.1.2的出口流量在9.4mbit左右，这与设置的分类策略基本一致。

    * 192.168.1.4 服务器端流量数据：
    ```
    源IP为192.168.1.2和192.168.1.3的入口流量ID分别为4和5，目的IP为192.168.1.2和192.168.1.3的出口流量ID分别为6和8
    
    Server listening on TCP port 5001
    TCP window size:  416 KByte
    
    [  4] local 192.168.1.4 port 5001 connected with 192.168.1.2 port 36186
    
    Client connecting to 192.168.1.2, TCP port 5001
    TCP window size:  416 KByte
    
    [  6] local 192.168.1.4 port 43190 connected with 192.168.1.2 port 5001
    [ ID] Interval       Transfer     Bandwidth
    [  4]  0.0- 1.0 sec  9.09 MBytes  76.3 Mbits/sec
    [  6]  0.0- 1.0 sec  1.62 MBytes  13.6 Mbits/sec
    [  5] local 192.168.1.4 port 5001 connected with 192.168.1.3 port 59050
    
    Client connecting to 192.168.1.3, TCP port 5001
    TCP window size:  416 KByte
    
    [  8] local 192.168.1.4 port 48090 connected with 192.168.1.3 port 5001
    [  4]  1.0- 2.0 sec  9.20 MBytes  77.2 Mbits/sec
    [  6]  1.0- 2.0 sec  1.12 MBytes  9.44 Mbits/sec
    [  5]  0.0- 1.0 sec  15.3 MBytes   128 Mbits/sec
    [  8]  0.0- 1.0 sec  4.88 MBytes  40.9 Mbits/sec
    [  4]  2.0- 3.0 sec  8.50 MBytes  71.3 Mbits/sec
    [  6]  2.0- 3.0 sec  1.12 MBytes  9.44 Mbits/sec
    [  5]  1.0- 2.0 sec  12.7 MBytes   107 Mbits/sec
    [  8]  1.0- 2.0 sec  4.50 MBytes  37.7 Mbits/sec
    [  4]  3.0- 4.0 sec  8.78 MBytes  73.7 Mbits/sec
    [  6]  3.0- 4.0 sec  1.00 MBytes  8.39 Mbits/sec
    [  5]  2.0- 3.0 sec  12.5 MBytes   105 Mbits/sec
    [  8]  2.0- 3.0 sec  4.50 MBytes  37.7 Mbits/sec
    [  4]  4.0- 5.0 sec  8.97 MBytes  75.3 Mbits/sec
    [  6]  4.0- 5.0 sec  1.12 MBytes  9.44 Mbits/sec
    [  5]  3.0- 4.0 sec  12.8 MBytes   107 Mbits/sec
    [  8]  3.0- 4.0 sec  4.25 MBytes  35.7 Mbits/sec
    [  4]  5.0- 6.0 sec  8.97 MBytes  75.2 Mbits/sec
    [  6]  5.0- 6.0 sec  1.00 MBytes  8.39 Mbits/sec
    [  5]  4.0- 5.0 sec  12.4 MBytes   104 Mbits/sec
    [  8]  4.0- 5.0 sec  4.50 MBytes  37.7 Mbits/sec
    [  6]  6.0- 7.0 sec  1.12 MBytes  9.44 Mbits/sec
    [  4]  6.0- 7.0 sec  9.01 MBytes  75.6 Mbits/sec
    [  5]  5.0- 6.0 sec  12.7 MBytes   107 Mbits/sec
    [  8]  5.0- 6.0 sec  4.50 MBytes  37.7 Mbits/sec
    [  4]  7.0- 8.0 sec  8.53 MBytes  71.6 Mbits/sec
    [  6]  7.0- 8.0 sec  1.12 MBytes  9.44 Mbits/sec
    [  5]  6.0- 7.0 sec  12.8 MBytes   108 Mbits/sec
    [  8]  6.0- 7.0 sec  4.50 MBytes  37.7 Mbits/sec
    [  4]  8.0- 9.0 sec  9.73 MBytes  81.7 Mbits/sec
    [  6]  8.0- 9.0 sec  1.12 MBytes  9.44 Mbits/sec
    [  5]  7.0- 8.0 sec  12.4 MBytes   104 Mbits/sec
    [  8]  7.0- 8.0 sec  4.50 MBytes  37.7 Mbits/sec
    [  4]  9.0-10.0 sec  8.99 MBytes  75.4 Mbits/sec
    [  6]  9.0-10.0 sec  1.12 MBytes  9.44 Mbits/sec
    [  4]  0.0-10.1 sec  90.0 MBytes  75.0 Mbits/sec
    ```
    
    * 192.168.1.3 客户端出入口流量：
    ```
    Client connecting to 192.168.1.4, TCP port 5001
    TCP window size:  416 KByte
    
    [  5] local 192.168.1.3 port 59050 connected with 192.168.1.4 port 5001
    [  4] local 192.168.1.3 port 5001 connected with 192.168.1.4 port 48090
    [ ID] Interval       Transfer     Bandwidth
    [  5]  0.0- 1.0 sec  15.4 MBytes   129 Mbits/sec
    [  4]  0.0- 1.0 sec  4.50 MBytes  37.8 Mbits/sec
    [  5]  1.0- 2.0 sec  12.8 MBytes   107 Mbits/sec
    [  4]  1.0- 2.0 sec  4.45 MBytes  37.4 Mbits/sec
    [  4]  2.0- 3.0 sec  4.43 MBytes  37.2 Mbits/sec
    [  5]  2.0- 3.0 sec  12.4 MBytes   104 Mbits/sec
    [  5]  3.0- 4.0 sec  13.0 MBytes   109 Mbits/sec
    [  4]  3.0- 4.0 sec  4.47 MBytes  37.5 Mbits/sec
    [  4]  4.0- 5.0 sec  4.44 MBytes  37.3 Mbits/sec
    [  5]  4.0- 5.0 sec  12.4 MBytes   104 Mbits/sec
    [  4]  5.0- 6.0 sec  4.45 MBytes  37.4 Mbits/sec
    [  5]  5.0- 6.0 sec  12.9 MBytes   108 Mbits/sec
    [  4]  6.0- 7.0 sec  4.45 MBytes  37.4 Mbits/sec
    [  5]  6.0- 7.0 sec  12.8 MBytes   107 Mbits/sec
    [  5]  7.0- 8.0 sec  12.2 MBytes   103 Mbits/sec
    [  4]  7.0- 8.0 sec  4.49 MBytes  37.7 Mbits/sec
    [  5]  8.0- 9.0 sec  12.5 MBytes   105 Mbits/sec
    [  4]  8.0- 9.0 sec  4.41 MBytes  37.0 Mbits/sec
    [  4]  9.0-10.0 sec  4.46 MBytes  37.5 Mbits/sec
    [  5]  9.0-10.0 sec  12.4 MBytes   104 Mbits/sec
    [  5]  0.0-10.0 sec   129 MBytes   108 Mbits/sec
    [  4]  0.0-10.1 sec  45.0 MBytes  37.4 Mbits/sec
    ```
    
    * 192.168.1.2 客户端出入口流量：
    ```
    Server listening on TCP port 5001
    TCP window size:  416 KByte
    
    Client connecting to 192.168.1.4, TCP port 5001
    TCP window size:  416 KByte
    
    [  5] local 192.168.1.2 port 36186 connected with 192.168.1.4 port 5001
    [  4] local 192.168.1.2 port 5001 connected with 192.168.1.4 port 43190
    [ ID] Interval       Transfer     Bandwidth
    [  4]  0.0- 1.0 sec  1.13 MBytes  9.45 Mbits/sec
    [  5]  0.0- 1.0 sec  9.38 MBytes  78.6 Mbits/sec
    [  4]  1.0- 2.0 sec  1.10 MBytes  9.27 Mbits/sec
    [  5]  1.0- 2.0 sec  9.12 MBytes  76.5 Mbits/sec
    [  5]  2.0- 3.0 sec  8.50 MBytes  71.3 Mbits/sec
    [  4]  2.0- 3.0 sec  1.11 MBytes  9.35 Mbits/sec
    [  5]  3.0- 4.0 sec  8.75 MBytes  73.4 Mbits/sec
    [  4]  3.0- 4.0 sec  1.10 MBytes  9.26 Mbits/sec
    [  5]  4.0- 5.0 sec  8.88 MBytes  74.4 Mbits/sec
    [  4]  4.0- 5.0 sec  1.11 MBytes  9.30 Mbits/sec
    [  5]  5.0- 6.0 sec  9.00 MBytes  75.5 Mbits/sec
    [  4]  5.0- 6.0 sec  1.11 MBytes  9.31 Mbits/sec
    [  4]  6.0- 7.0 sec  1.12 MBytes  9.37 Mbits/sec
    [  5]  6.0- 7.0 sec  9.00 MBytes  75.5 Mbits/sec
    [  4]  7.0- 8.0 sec  1.11 MBytes  9.30 Mbits/sec
    [  5]  7.0- 8.0 sec  8.62 MBytes  72.4 Mbits/sec
    [  5]  8.0- 9.0 sec  9.62 MBytes  80.7 Mbits/sec
    [  4]  8.0- 9.0 sec  1.11 MBytes  9.30 Mbits/sec
    [  4]  9.0-10.0 sec  1.11 MBytes  9.33 Mbits/sec
    [  5]  9.0-10.0 sec  9.12 MBytes  76.5 Mbits/sec
    [  5]  0.0-10.0 sec  90.0 MBytes  75.3 Mbits/sec
    [  4] 10.0-11.0 sec  1.14 MBytes  9.53 Mbits/sec
    [  4]  0.0-11.7 sec  13.0 MBytes  9.35 Mbits/sec
    ```
    
* 实验结果分析

    根据实验结果可以分析出，基于macvlan的容器网络模型，使用TC技术可以在容器内实现网卡的入口和出口流量限速，其中入口流量限速需要把流量导入ifb设备才能保证限速的稳定性。

## 场景分析

当前容器云架构中，service维护容器实例个数、容器运行时参数、容器CPU和Memory的QoS以及容器调度等各种策略，因此在service中再增加容器网络QoS的管理和维护是比较符合应用场景的，主要有如下优点：

1. 支持service下所有容器应用相同的QoS策略
2. 支持对容器网络QoS参数实时的更新和维护
3. 支持对一个容器内多个网络同时应用QoS策略

## 行业参考

* K8S
    
    k8s 提供了基于pod的cpu和memory的QoS功能，网络QoS需要用户根据自身的网络插件来定制实现，无具体标准可以参考。
    
* Contiv netplugin
    
    Contiv是Cisco开源的针对容器的基础架构，主要功能是提供基于Policy的网络管理，其网络QoS功能主要通过ovs的“ingress_policing_rate”和“ingress_policing_burst”这两个参数实现入口流量限速，无出口流量限速相关功能。

* Neutron
    
    openstack的neutron组件提供了网络QoS的基础框架，其默认实现也是通过调用ovs的“ingress_policing_rate”和“ingress_policing_burst”这两个参数实现入口流量限速，无出口流量限速相关功能。

## 调研结论

1. 由于网络方案的复杂性，行业内开源的网络QoS策略大多只是提供了简单的参考实现，企业级用户需要根据自身的网络方案，定制开发符合自身业务需求的网络QoS功能。

2. 在当前基于macvlan的容器网络模型中，通过TC可以在容器内网卡上实现入口流量限速(导入流量到ifb)和出口流量限速。

3. 容器网络QoS策略宜在service层管理和维护，策略执行由macvlan plugin调用底层TC命令来实现，具体与service层的交互模型、架构模型和与macvlan plugin的通信方式在概念设计文档中详细说明。

## Reference

* neutron qos 

    http://specs.openstack.org/openstack/neutron-specs/specs/liberty/qos-api-extension.html

* ovs qos
    
    http://docs.openvswitch.org/en/latest/faq/qos/

* contive netplugin

    https://github.com/contiv/netplugin

* TC howto
    
    http://www.tldp.org/HOWTO/html_single/Traffic-Control-HOWTO/

    http://lartc.org/howto/

* TC man page
    
    https://linux.die.net/man/8/tc

* ustack floating ip qos
    
    https://docs.ustack.com/unp/src/funcs/fip_qos.html

* TC ingress and ifb

    http://blog.csdn.net/dog250/article/details/40680765

* Linux network namespace

    http://blog.scottlowe.org/2013/09/04/introducing-linux-network-namespaces/

End
