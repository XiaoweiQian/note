---
layout: post
title:  Openstack 部署总结
comments: false
category: openstack
---

1. 安装前准备
    * OS： Centos7.1
    * 修改节点名称
    ```c
    # vi /etc/hostname
    ```
    * 修改hosts
    ```c
    # vi /etc/hosts
    ```
    * 禁用NetworkManager
    ```c
    # systemctl disable NetworkManager
    # systemctl stop NetworkManager
    ```
    * 禁用SELINUX
    ```c
    # vi /etc/selinux/config 
     设置SELINUX=disabled
    开启IP转发
    # vi /etc/sysctl.conf
    添加如下内容：
    net.ipv4.ip_forward=1
    net.ipv4.conf.all.rp_filter=0
    net.ipv4.conf.default.rp_filter=0
    # sysctl -p
    ```
    * 设置防火墙
    ```c
    # firewall-cmd --set-default-zone=trusted
    ```
    * 设置censtos7和kilo本地源
    ```c
    # cd /etc/yum.repos.d/
    # rm -rf CentOS-*
    # curl -o kilo.repo http://192.168.1.10/kilo.repo
    # yum clean all
    # yum makecache
    ```
    * 更新系统
    ```c
    # yum -y update
    ```
    * 重启系统
    ```c
    # reboot
    ```
    * 删除系统升级产生的源文件
    ```c
    # cd /etc/yum.repos.d/
    # rm -rf CentOS-*
    ```
2. 基础服务安装
    * ntp时间同步
    ```c
    安装
    # yum -y install ntp
    配置ntp
    # vi /etc/ntp.conf
    注释所有server，新增ntp服务器地址
    server 192.168.1.10
    注意：如果要把本机设置为ntp源服务器需要注释所有server配置，再增加一条：
    broadcast 193.160.31.255 autokey
    启动ntp
    # systemctl enable ntpd
    # systemctl start ntpd
    ```

    * mariadb
    ```c
    安装
    # yum -y install mariadb-galera-server MySQL-python
    配置my.cnf
    # vi /etc/my.cnf
    增加内容如下：
    [mysqld]
    default-storage-engine = innodb
    innodb_file_per_table
    collation-server = utf8_general_ci
    init-connect = 'SET NAMES utf8'
    character-set-server = utf8
    启动mariadb
    # systemctl enable mariadb
    # systemctl start mariadb
    ```

    * rabbitmq
    ```c
    安装
    # yum -y install rabbitmq-server
    启动
    # systemctl enable rabbitmq-server
    # systemctl start rabbitmq-server
    增加openstack用户
    # rabbitmqctl add_user openstack RABBIT_PASS
    设置openstack用户权限
    # rabbitmqctl set_permissions openstack ".*" ".*" ".*"
    ```
3. openstack服务安装
    * controller节点需要安装keystone、nova、glance、cinder、neutron、dashboard服务，
    compute节点需要安装neutron和nova服务
    安装步骤参照官方手册，链接如下：
    http://docs.openstack.org/kilo/install-guide/install/yum/content/
    安装注意事项：
    ```
    (1) 由于前面1，2步骤已经做了安装前配置,因此官方文档中前面系统准备配置可以跳过，直接从keystone开始安装。
    (2) 安装keystone时，由于网络原因下面这个步骤可能会不能正确执行
    Copy the WSGI components from the upstream repository into this directory:
    #curl http://git.openstack.org/cgit/openstack/keystone/plain/httpd/
    keystone.py?h=stable/kilo \| tee /var/www/cgi-bin/keystone/main /var/www/cgi-bin/keystone/admin
    解决办法：
    先从http://git.openstack.org/cgit/openstack/keystone/plain/httpd/keystone.py?h=stable/kilo
    下载keystone.py文件，然后重命名成main和admin这两个文件，再复制到/var/www/cgi-bin/keystone目录
    
    (3) 安装cinder时，/usr/share/cinder/cinder-dist.conf配置文件第一行有错误
    解决办法：
    把logdir改为log_dir
   ```
4. PKI 配置
   controller节点 配置PKI步骤如下：

   * 修改keystone.conf
   ```c
   certfile = /etc/keystone/ssl/certs/signing_cert.pem
   keyfile = /etc/keystone/ssl/private/signing_key.pem
   ca_certs = /etc/keystone/ssl/certs/ca.pem
   ca_key = /etc/keystone/ssl/private/cakey.pem
   key_size = 2048
   valid_days = 3650
   cert_subject = /C=US/ST=Unset/L=Unset/O=Unset/CN=192.168.31.23
   ```
   
   * 生成证书文件
   ```c
   # keystone-manage pki_setup --keystone-user keystone --keystone-group keystone
   # chown -R keystone:keystone /etc/keystone/*
   拷贝/etc/keystone/ssl 目录到controller-standby 节点，并授权
   # chown -R keystone:keystone /etc/keystone/*
   ```

   * 重起controller节点所有openstack服务
   ```c
   # openstack-service restart
   ```

5. rabbitmq集群模式研究：
    采用openstack官方配置，使用rabbit_hosts 来指定多个rabbitmq节点，这种模式下，所有服务只连接rabbit_hosts中第一个节点，当第一个节点失去连接后才自动连接第二节点，这导致第一个节点rabbitmq负载过大，容易出现连接丢失和vm状态异常等错误。
    
    合理的rabbitmq集群模式：
    不配置rabbit_hosts，使用rabbit_host 指定到haproxy服务器，通过haproxy的负载均衡功能，把连接自动分配到各个rabbitmq节点上。

    * rabbitmq 使用三节点模式，cmp09新安装rabbitmq并开启web管理模块
    ```c
      # yum install rabbitmq-server
      # rabbitmq-plugins enable rabbitmq_management
      # rabbitmqctl set_user_tags username administrator
    ```
    
    * haproxy 配置负载均衡
    ```c
    listen rabbitmq_cluster
      bind :5670
      mode tcp
      option tcpka
      balance roundrobin
      option tcplog
      server controller 193.160.31.20:5672 check inter 2000 rise 2 fall 5
    ```
    
    * 修改nova,neutron,glance,cinder 中相关配置文件
    ```c
      注释rabbit_hosts配置项
      增加如下配置项：
      rabbit_host=193.160.31.23
      rabbit_port=5670
    ```

6. compute 热迁移相关配置
    * 需要配置计算节点无密码SSH
    新增加的计算节点，需要配置nova用户SSH无密码访问步骤如下：
    ```
    a.允许nova用户访问
    修改/etc/passwd
    nova:x:162:162:OpenStack Nova Daemons:/var/lib/nova:/sbin/nologin
    =》nova:x:162:162:OpenStack Nova Daemons:/var/lib/nova:/bin/bash
    
    b.生成nova用户ssh key
    su nova
    ssh-keygen -t rsa
    
    c.拷贝/var/lib/nova/.ssh/id_rsa.pub中内容到各个计算节点的/var/lib/nova/.ssh/authorized_keys中
    ```
    * Libvirtd配置
    ```c
    /etc/libvirt/qemu.conf中修改如下配置
    security_driver = "none"
    dynamic_ownership = 0
    /etc/libvirt/libvirtd.conf中修改如下配置
    listen_tls = 0
    listen_tcp = 1
    listen_addr = ipaddr （此处填写存储网络地址，因为迁移对带宽要求较大）
    auth_tcp = "none"
    ```
