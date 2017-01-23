---
date: 2016-01-05
layout: post
title: KVM 环境搭建ceph集群
categories: 技术
tags: ceph kvm openstack
---

## 环境要求

    * OS：Centos7.1
    * QEMU: 根据qemu2.5源码重新编译打包RPM包，编译时需要开启RBD支持
    * CEPH: 搭建9.2.0版本地源
    * 配置NTP服务器，broadcast 192.168.1.255 autokey
    * 关闭SELINUX
    * 设置防火墙zone为trusted

## 环境准备
    
    * KVM虚拟机配置
    
    创建三台KVM虚拟机，一台当CEPH monitor，另外两台当CEPH STORAGE，IP和磁盘分配如下：
    ```
    CEPH monitor:
        hostname:mon1 
            ip:192.168.1.10
            disk: vda 20G
    
    CEPH STORAGE1:
    	hostname:storage1 
            ip:192.168.1.11
            disk: vda 20G,vdb 20G,vdc 50G,vdd 50G,vde 50G
    
    CEPH STORAGE2:
    	hostname:storage2
            ip:192.168.1.12
            disk: vda 20G,vdb 20G,vdc 50G,vdd 50G,vde 50G
    ```

    * libvirt 新增disk

    ```sh
    # 创建快设备镜像
    qemu-img create -f qcow2 /kvm/storage1-vdb.qcow2 20G

    # 挂载快设备镜像到VM
    virsh  attach-disk storage1 /kvm/storage1-vdb.qcow2 vdb --subdriver qcow2 --cache none
    
    # 编辑VM xml文件,新增disk描述信息，保证重起后能自动挂载新增的disk
    virsh edit storage1
    <disk type='file' device='disk'>
    　　<driver name='qemu' type='qcow2' cache='none'/>
    　　<source file='/kvm/storage1-vdb.qcow2'/>
    　　<target dev='vdb' bus='virtio'/>
    </disk>
    ```

    * CEPH本地源搭建

    (1)下载CEPH 9.2.0 代码到本地源服务器，release.asc文件也需要下载。
    (2)在三个结点上新增ceph.repo 文件，格式如下：
    
    ```
    [ceph]
    name=Ceph packages for $basearch
    baseurl=http://192.168.1.10:8080/ceph/$basearch
    enabled=1
    gpgcheck=1
    priority=1
    type=rpm-md
    gpgkey=http://192.168.1.10:8080/ceph/release.asc
    
    [ceph-noarch]
    name=Ceph noarch packages
    baseurl=http://192.168.1.10:8080/ceph/noarch
    enabled=1
    gpgcheck=1
    priority=1
    type=rpm-md
    gpgkey=http://192.168.1.10:8080/ceph/release.asc
    
    [ceph-source]
    name=Ceph source packages
    baseurl=http://192.168.1.10:8080/ceph/SRPMS
    enabled=0
    gpgcheck=1
    type=rpm-md
    gpgkey=http://192.168.1.10:8080/ceph/release.asc
    priority=1
    ```

    * 设置SSH密钥

    在Ceph monitor节点生成 ssh 密钥并把密钥复制到每个Ceph集群节点。
    
    ```sh
    ssh-keygen
    ssh-copy-id root@storage1
    ssh-copy-id root@storage2
    ```

    * 配置 PID 数目

    需要为每个节点配置PID数目的值，默认情况下值为32768，需要改为4194303
    
    ```c
    vi /etc/sysctl.conf
    kernel.pid_max=4194303
    sysctl -p
    ```

## 安装CEPH

    * monitor 安装ceph-deploy,并生成配置文件

    ```sh
    yum install ceph-deploy
    mkdir ~/ceph-cluster
    cd ~/ceph-cluster
    ceph-deploy new storage 
    ```

    * 修改集群配置文件ceph.conf
    
    ```sh
    vi ceph.conf
    [global]
    fsid = 200ce52d-6b00-4b91-a5f2-8178535e8f28
    mon_initial_members = mon1        #monitor节点hostname
    mon_host = 192.168.1.10           #monitor节点IP
    auth_cluster_required = cephx
    auth_service_required = cephx
    auth_client_required = cephx
    filestore_xattr_use_omap = true
    osd_pool_default_size = 2         #osd副本数
    cluster_network = 192.168.1.0/24  #指定存储网络
    
    [mon]
    mon_clock_drift_allowed = 2       #注定OSD允许延迟为2秒
    
    ```

    * 使用ceph-deploy 部署各个节点
    
    ```sh
    ceph-deploy install --repo-url http://192.168.1.10:8080/ceph --gpg-url http://192.168.1.10:8080/ceph/release.asc mon1 storage1 sorage2
    ceph-deploy mon create-initial
    ```
    
    * 在storage1 和 storage2 节点增加OSD
    
    ```sh
    # 执行prepare命令,storage1 上vdb作为日志盘，其它vdc,vdd,vde 创建osd
    ceph-deploy osd prepare storage1:vdc:/dev/vdb
    ceph-deploy osd prepare storage1:vdd:/dev/vdb
    ceph-deploy osd prepare storage1:vde:/dev/vdb
    
    # 执行activate命令
    ceph-deploy osd activate storage1:/dev/vdc1:/dev/vdb1
    ceph-deploy osd activate storage1:/dev/vdd1:/dev/vdb2
    ceph-deploy osd activate storage1:/dev/vde1:/dev/vdb3
    
    # storage2 创建OSD 参照以上命令，最后执行以下命令把配置文件拷贝到各个节点
    ceph-deploy --overwrite-conf admin mon1 storage1 storage2
    
    # 查看集群状态，若为HEALTH_OK 则配置成功
    ceph status
    ceph osd tree
    ```

## openstack配置使用Ceph 块存储

    * 创建pool
    
    ```sh
    ceph osd pool create volumes 128
    ceph osd pool create images 128
    ceph osd pool create backups 128
    ceph osd pool create vms 128
    ```
    
    * 拷贝ceph.conf 到所有opensack host
    
    ```sh
    ssh {your-openstack-server} tee /etc/ceph/ceph.conf </etc/ceph/ceph.conf
    ```
    
    * 安装ceph client
    
    ```sh
    # glance-api node， 安装librbd
    yum install python-rbd
    # nova-compute、cinder-backup、cinder-volume node，安装ceph
    yum install ceph
    ```
    
    * oepnstack中配置ceph权限认证
    在mon1 节点生成如下三个认证文件

    ```sh
    ceph auth get-or-create client.cinder mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=volumes, allow rwx pool=vms, allow rx pool=images'
    ceph auth get-or-create client.glance mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=images'
    ceph auth get-or-create client.cinder-backup mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=backups'
    
    # 拷贝这三个文件到openstack相应的节点中
    ceph auth get-or-create client.glance | ssh {your-glance-api-server} tee /etc/ceph/ceph.client.glance.keyring

    # ssh {your-glance-api-server} chown glance:glance /etc/ceph/ceph.client.glance.keyring
    ceph auth get-or-create client.cinder | ssh {your-volume-server} tee /etc/ceph/ceph.client.cinder.keyring

    # ssh {your-cinder-volume-server} chown cinder:cinder /etc/ceph/ceph.client.cinder.keyring
    ceph auth get-or-create client.cinder-backup | ssh {your-cinder-backup-server} tee /etc/ceph/ceph.client.cinder-backup.keyring

    # ssh {your-cinder-backup-server} chown cinder:cinder /etc/ceph/ceph.client.cinder-backup.keyring
    
    # 计算节点需要配置cinder认证
    ceph auth get-or-create client.cinder | ssh {your-nova-compute-server} tee /etc/ceph/ceph.client.cinder.keyring
    
    # 在一台计算节点中配置libvirt认证
    ceph auth get-key client.cinder | ssh {your-compute-node} tee client.cinder.key

    # 生成key
    uuidgen
    457eb676-33da-42ec-9a8c-9293d545c337
    cat > secret.xml <<EOF
    <secret ephemeral='no' private='no'>
      <uuid>457eb676-33da-42ec-9a8c-9293d545c337</uuid>
      <usage type='ceph'>
        <name>client.cinder secret</name>
      </usage>
    </secret>
    EOF
    virsh secret-define --file secret.xml
    Secret 457eb676-33da-42ec-9a8c-9293d545c337 created
    virsh secret-set-value --secret 457eb676-33da-42ec-9a8c-9293d545c337 --base64 $(cat client.cinder.key)
    
    # 把secret.xml 和 client.cinder.key 文件拷贝到其他所有计算节点中，执行
    virsh secret-define --file secret.xml
    virsh secret-set-value --secret 457eb676-33da-42ec-9a8c-9293d545c337 --base64 $(cat client.cinder.key)
    ```
    
    * oepnstack中配置文件修改
    
    (1) glance 节点
    ```sh
    vi /etc/glance/glance-api.conf
    [DEFAULT]
    show_image_direct_url = True
    
    [glance_store]
    default_store=rbd
    stores = rbd
    rbd_store_pool = images
    rbd_store_user = glance
    rbd_store_ceph_conf = /etc/ceph/ceph.conf
    rbd_store_chunk_size = 8
    
    [paste_deploy]
    flavor = keystone
    
    # glance 中image推荐作如下修改
        hw_scsi_model=virtio-scsi: add the virtio-scsi controller and get better performance and support for discard operation
        hw_disk_bus=scsi: connect every cinder block devices to that controller
        hw_qemu_guest_agent=yes: enable the QEMU guest agent
        os_require_quiesce=yes: send fs-freeze/thaw calls through the QEMU guest agent
    
    (2) cinder节点
    
    ```sh
    [DEFAULT]
    enabled_backends = volumes
    glance_host=193.160.31.45
    glance_api_version=2
     
    [volumes]
    volume_backend_name=volumes
    volume_driver = cinder.volume.drivers.rbd.RBDDriver
    rbd_pool = volumes
    rbd_ceph_conf = /etc/ceph/ceph.conf
    rbd_flatten_volume_from_snapshot = false
    rbd_max_clone_depth = 5
    rbd_store_chunk_size = 4
    rados_connect_timeout = -1
    rbd_user = cinder
    rbd_secret_uuid = 457eb676-33da-42ec-9a8c-9293d545c337
    ```

    (3) compute 节点

    所有计算节点ceph.conf 文件中增加如下配置

    ```sh
    [client]
    rbd_cache = true
    rbd_cache_writethrough_until_flush = true
    log_file = /var/log/qemu/qemu-guest-$pid.log
    rbd_concurrent_management_ops = 20
    mkdir -p /var/run/ceph/guests/ /var/log/qemu/
    chown qemu:libvirtd /var/run/ceph/guests /var/log/qemu/
    
    # 所有计算节点nova.conf 文件中增加如下配置
    [libvirt]
    images_type = rbd
    images_rbd_pool = vms
    images_rbd_ceph_conf = /etc/ceph/ceph.conf
    rbd_user = cinder
    rbd_secret_uuid = 457eb676-33da-42ec-9a8c-9293d545c337
    disk_cachemodes="network=none"
    inject_password = false
    inject_key = false
    inject_partition = -2
    virt_type = qemu
    #hw_disk_discard = unmap
    live_migration_flag="VIR_MIGRATE_UNDEFINE_SOURCE,VIR_MIGRATE_PEER2PEER,VIR_MIGRATE_LIVE,VIR_MIGRATE_PERSIST_DEST,VIR_MIGRATE_TUNNELLED"
    
    ```

    * 重起所有节点openstack服务

End
