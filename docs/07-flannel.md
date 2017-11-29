# 部署flannel插件 #

## 下载 ##
    curl -OL https://github.com/coreos/flannel/releases/download/v0.9.1/flannel-v0.9.1-linux-amd64.tar.gz
    
    tar -xzvf flannel-v0.9.1-linux-amd64.tar.gz -C flannel
    cp flannel/flanenld /usr/local/bin/flanneld

## 创建flanneld.service ##

<pre>
<code>
[Unit]
Description=Flanneld overlay address etcd agent
After=network.target
After=network-online.target
Wants=network-online.target
After=etcd.service
Before=docker.service

[Service]
Type=notify
ExecStart=/usr/local/bin/flanneld \
  -etcd-cafile=/etc/etcd/ca.pem \
  -etcd-certfile=/etc/etcd/kubernetes.pem \
  -etcd-keyfile=/etc/etcd/kubernetes-key.pem \
  -etcd-endpoints=https://10.66.0.67:2379,https://10.66.0.68:2379,https://10.66.0.69:2379 \
ExecStartPost=/etc/flannel/mk-docker-opts.sh -k DOCKER_NETWORK_OPTIONS -d /run/flannel/docker
Restart=on-failure

[Install]
WantedBy=multi-user.target
RequiredBy=docker.service

</pre>
</code>

## pod网段写入etcd ##
<pre>
<code>
etcdctl --cert-file=kubernetes.pem \
        --key-file=kubernetes-key.pem 
        --ca-file=ca.pem \
        set /coreos.com/network/config \
        {"Network":"192.168.0.0/16", "Backend":{"Type":"vxlan"}}
</pre>
</code>

    cp flanneld.service /etc/systemd/system/
    systemctl daemon-reload
    systemctl enable flanneld
    systemctl start flanneld

## 验证 ##
<pre>
<code>
[root@SvrXJK8sMaster01 config]# etcdctl --cert-file=kubernetes.pem --key-file=kubernetes-key.pem --ca-file=ca.pem get /coreos.com/network/config
{"Network":"192.168.0.0/16", "Backend":{"Type":"vxlan"}}

[root@SvrXJK8sMaster01 config]# etcdctl --cert-file=kubernetes.pem --key-file=kubernetes-key.pem --ca-file=ca.pem ls /coreos.com/network/subnets
/coreos.com/network/subnets/192.168.52.0-24
/coreos.com/network/subnets/192.168.28.0-24
/coreos.com/network/subnets/192.168.97.0-24
/coreos.com/network/subnets/192.168.48.0-24
/coreos.com/network/subnets/192.168.96.0-24
/coreos.com/network/subnets/192.168.70.0-24

[root@SvrXJK8sMaster01 config]# etcdctl --cert-file=kubernetes.pem --key-file=kubernetes-key.pem --ca-file=ca.pem get /coreos.com/network/subnets/192.168.52.0-24
{"PublicIP":"10.66.0.71","BackendType":"vxlan","BackendData":{"VtepMAC":"7e:cb:66:c9:c6:77"}}

[root@SvrXJK8sMaster01 config]# ip a s flannel.1
4: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN 
    link/ether ea:ef:6f:ee:e3:ac brd ff:ff:ff:ff:ff:ff
    inet 192.168.48.0/32 scope global flannel.1
       valid_lft forever preferred_lft forever
    inet6 fe80::e8ef:6fff:feee:e3ac/64 scope link 
       valid_lft forever preferred_lft forever

各节点互ping flannel.1地址，要OK
</pre>
</code>

## 安装docker-ce ##

    $ curl -fsSL get.docker.com -o get-docker.sh
    $ sudo sh get-docker.sh

docker配置: 

<pre>
<code>
mkdir -p /etc/systemd/system/docker.service.d
下载google相关镜像需要科学上网，配置http_proxy
cat  /etc/systemd/system/docker.service.d/http-proxy.conf 
[Service]
Environment="HTTP_PROXY=http://10.0.193.33:1080/"
</pre>
</code>

创建使用flannel 的环境变量配置文件:

<pre>
<code>
cat /etc/systemd/system/docker.service.d/40-flannel.conf
[Unit]
Requires=flanneld.service
After=flanneld.service
[Service]
EnvironmentFile=/run/flannel/docker

/run/flannel/docker为mk-docker-opts.sh生成的相关环境变量



cat /run/flannel/docker
DOCKER_OPT_BIP="--bip=192.168.52.1/24"
DOCKER_OPT_IPMASQ="--ip-masq=true"
DOCKER_OPT_MTU="--mtu=1450"
DOCKER_NETWORK_OPTIONS=" --bip=192.168.52.1/24 --ip-masq=true --mtu=1450"

</pre>
</code>

修改启动docker参数的环境变量:
<pre>
<code>
cat /usr/lib/systemd/system/docker.service
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service
Wants=network-online.target

[Service]
Type=notify
# the default is not to use systemd for cgroups because the delegate issues still
# exists and systemd currently does not support the cgroup feature set required
# for containers run by docker
ExecStart=/usr/bin/dockerd $DOCKER_NETWORK_OPTIONS  #添加此环境变量
ExecReload=/bin/kill -s HUP $MAINPID
ExecStartPost=/sbin/iptables -A FORWARD -s 0.0.0.0/0 -j ACCEPT #修改默认iptable规则，默认为DROP
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
# Uncomment TasksMax if your systemd version supports it.
# Only systemd 226 and above support this version.
#TasksMax=infinity
TimeoutStartSec=0
# set delegate yes so that systemd does not reset the cgroups of docker containers
Delegate=yes
# kill only the docker process, not all processes in the cgroup
KillMode=process
# restart the docker process if it exits prematurely
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s

[Install]
WantedBy=multi-user.target
</pre>
</code>

    systemctl daemon-reload
    systemctl enable docker
    systemctl start docker

检查docker服务和网络:
<pre>
<code>
docker version
docker info
ip a
3: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN 
    link/ether 7a:54:93:80:3b:f3 brd ff:ff:ff:ff:ff:ff
    inet 192.168.97.0/32 scope global flannel.1
       valid_lft forever preferred_lft forever
    inet6 fe80::7854:93ff:fe80:3bf3/64 scope link 
       valid_lft forever preferred_lft forever
4: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP 
    link/ether 02:42:03:69:d8:35 brd ff:ff:ff:ff:ff:ff
    inet 192.168.97.1/24 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:3ff:fe69:d835/64 scope link 
       valid_lft forever preferred_lft forever

docker0和flannel.1在同一个网段中，这两是桥接，其他节点能互ping 这两接口的地址
</pre>
</code>


下一步：[配置和启动kubernetes Worker节点](08-k8s-workers.md)



















