# Prerequisites #

## 集群机器 ##
- 10.66.0.67
- 10.66.0.68
- 10.66.0.69
- 10.66.0.71
- 10.66.0.72
- 10.66.0.73
- 10.66.0.75

## 机器说明 ##
- 67,68,69为etcd集群、Kubernetes master集群
- 71,72,73为kubernetes worker节点
- 75使用haproxy作为kubernetes api服务器的loadbalancer

## 环境参数 ##
POD_NETWORK=192.168.0.0/16

SERVICE_IP_RANGE=192.168.0.0/24   不能和POD_NETWORK地址重叠，也不能和已存在的网络架构地址重叠

K8S_SERVICE_IP=192.168.0.1  Kubernetes API 服务的虚拟IP

DNS_SERVICE_IP=192.168.0.10 此地址必须为SERVICE_IP_RANGE的地址范围

## 其他环境设置 ##
关闭防火墙

    systemctl stop firewalld.service
    systemctl disable firewalld.service

关闭Selinux

    cat /etc/selinux/config
    SELINUX=disabled

修改系统配置
创建/etc/sysctl.d/k8s.conf文件，添加如下内容：

    net.bridge.bridge-nf-call-ip6tables= 1
    net.bridge.bridge-nf-call-iptables= 1

执行sysctl -p /etc/sysctl.d/k8s.conf

关闭swap

    swapoff -a
     /etc/fstab注释掉swap的挂载


> kubernetes 1.8开始要求关闭系统的swap，也可以修改--fail-swap-on=fals

## 关于GFW问题 ##
大量文章写道在xx云服务商那搞一个VPC，然后安装docker
在xx云服务商的开启shadowsocket服务
在PC上下一个客户端，并提供代理服务。本文使用的代理地址为http://10.0.193.33:1080


## CentOS 7文件系统问题 ##
如果是默认安装CentOS 7默认使用的是xfs
Docker所使用的Overlayfs和Overlay2文件系统驱动，这两项驱动依赖overlayfs文件系统，Docker使用Overlayfs存储image的，Overlayfs代码（内核一部分）检查d_type特性是否开启。

### 检查docker info ###
>注意docke info输出显示警告： the backing xfs filesystem is formatted without d_type support...

<pre>
<code>
docker info

Containers: 15
 Running: 12
 Paused: 0
 Stopped: 3
Images: 11
Server Version: 17.10.0-ce
Storage Driver: overlay
 Backing Filesystem: xfs    #后端文件系统
 Supports d_type: false     #d_type没有开启
Logging Driver: json-file
Cgroup Driver: cgroupfs
Plugins:
 Volume: local
 Network: bridge host macvlan null overlay
 Log: awslogs fluentd gcplogs gelf journald json-file logentries splunk syslog
Swarm: inactive
Runtimes: runc
Default Runtime: runc
Init Binary: docker-init
containerd version: 06b9cb35161009dcb7123345749fef02f7cea8e0
runc version: 0351df1c5a66838d0c392b4ac4cf9450de844e2d
init version: 949e6fa
Security Options:
 seccomp
  Profile: default
Kernel Version: 3.10.0-327.el7.x86_64
Operating System: CentOS Linux 7 (Core)
OSType: linux
Architecture: x86_64
CPUs: 4
Total Memory: 15.26GiB
Name: svrxjk8sworker01
ID: JZMX:YGYD:5EFQ:IQZS:3EDU:2YJH:WVTQ:FHJG:UWVZ:KJ3Y:7NI6:PHWR
Docker Root Dir: /var/lib/docker
Debug Mode (client): false
Debug Mode (server): false
HTTP Proxy: http://10.0.193.33:1080/
Registry: https://index.docker.io/v1/
Experimental: false
Insecure Registries:
 127.0.0.0/8
Live Restore Enabled: false

WARNING: overlay: the backing xfs filesystem is formatted without d_type support, which leads to incorrect behavior.
         Reformat the filesystem with ftype=1 to enable d_type support.
         Running without d_type support will not be supported in future releases.


</code>
</pre> 


### 检查xfs文件系统 ###
<pre>
<code>
xfs_info /
meta-data=/dev/mapper/centos-root isize=256    agcount=4, agsize=3276800 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=0        finobt=0
data     =                       bsize=4096   blocks=13107200, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=0   #注意看这里的ftype=0意味着d_type是关闭的，很不幸。
log      =internal               bsize=4096   blocks=6400, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
</code>
</pre> 

### 开启d_type属性 ###
> 很不幸，没有什么配置开关可以设置开启d_type的属性，请在安装OS的时候，文件系统就要注意此选项。用以下三步解决

1. 备份Docker相关数据
2. 重建或增加存储新建带d_type=1的文件系统
3. 还原Docker数据


mkfs.xfs -n ftype=1 /path/to/your/device 
本文增加了存储，mkfs.xfs -n ftype=1 /dev/xvdb 

把/dev/xvdb 挂载到/docker/data

### 变更docker存储目录 ###
1. 修改/etc/docker/daemon.json去修改存储目录
2. 使用软连接方式改变存储目录


	 
    
    - 	停止docker 
    - 	备份/var/lib/docker
    - 	迁移/valib/docker到新目录 
    - 	新建软链接

<pre>
<code>  
systemctl stop docker
mv /var/lib/docker /docker/data
ln -s /docker/data /var/lib/docker
systemctl start docker
</code>
</pre>

### 检查docker是否支持d_type ###

    docker info

> 输出

    Storage Driver: overlay
     Backing Filesystem: xfs
     Supports d_type: true
    Logging Driver: json-file
    Cgroup Driver: cgroupfs
    ...
    Docker Root Dir: /docker/data



要了解d_type详细描述，请参考：[https://linuxer.pro/2017/03/what-is-d_type-and-why-docker-overlayfs-need-it/](https://linuxer.pro/2017/03/what-is-d_type-and-why-docker-overlayfs-need-it/)

[下一步：客户端工具安装](02-clinet-tools.md)


