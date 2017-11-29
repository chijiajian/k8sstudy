# 配置和启动etcd集群 #

Kubernetes组件是无状态的，集群状态是保存在etcd中。本实验启用3个节点作为etcd集群，并通过安全远程访问加强安全。

etcd部署在SvrXJK8sMaster01，SvrXJK8sMaster02，SvrXJK8sMaster03上，也可以拿3台主机独立部署

## 下载安装 ##

    curl -OL "https://github.com/coreos/etcd/releases/download/v3.2.8/etcd-v3.2.8-linux-amd64.tar.gz"
    
    tar -xvf etcd-v3.2.8-linux-amd64.tar.gz
    cp etcd-v3.2.8-linux-amd64/etcd* /usr/local/bin/

## etcd配置 ##

    mkdir -p /etc/etcd /var/lib/etcd
    cp ca.pem kubernetes-key.pem kubernetes.pem /etc/etcd/

## 创建etcd.service ##
要确保etcd的名称是集群中唯一的，建议使用hostname作为各etcd节点名称。
<pre>
<code>

[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
ExecStart=/usr/local/bin/etcd \
  --name=SvrXJK8sMaster01  \
  --cert-file=/etc/etcd/kubernetes.pem \
  --key-file=/etc/etcd/kubernetes-key.pem \
  --peer-cert-file=/etc/etcd/kubernetes.pem \
  --peer-key-file=/etc/etcd/kubernetes-key.pem \
  --trusted-ca-file=/etc/etcd/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ca.pem \
  --peer-client-cert-auth \
  --client-cert-auth \
  --initial-advertise-peer-urls=https://10.66.0.67:2380 \
  --listen-peer-urls=https://10.66.0.67:2380 \
  --listen-client-urls=https://10.66.0.67:2379,http://127.0.0.1:2379 \
  --advertise-client-urls=https://10.66.0.67:2379 \
  --initial-cluster-token etcdcluster \
  --initial-cluster SvrXJK8sMaster01=https://10.66.0.67:2380,SvrXJK8sMaster02=https://10.66.0.68:2380,SvrXJK8sMaster03=https://10.66.0.69:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
</pre>
</code>


依次生成SvrXJK8sMaster02,SvrXJK8sMaster03 的etcd.service

    cp etcd.service /etc/systemd/system/
    systemctl daemon-reload
    systemctl enable etcd
    systemctl start etcd

## 检查 ##
    
    ETCDCTL_API=3 etcdctl member list
    
    [root@SvrXJK8sMaster01 config]# etcdctl --cert-file=kubernetes.pem --key-file=kubernetes-key.pem --ca-file=ca.pem member list
    a9b850345993890: name=SvrXJK8sMaster02 peerURLs=https://10.66.0.68:2380 clientURLs=https://10.66.0.68:2379 isLeader=false
    bf8c6a606a699629: name=SvrXJK8sMaster01 peerURLs=https://10.66.0.67:2380 clientURLs=https://10.66.0.67:2379 isLeader=false
    c23f3e1a72e36f0c: name=SvrXJK8sMaster03 peerURLs=https://10.66.0.69:2380 clientURLs=https://10.66.0.69:2379 isLeader=true
    
    [root@SvrXJK8sMaster01 config]# etcdctl --cert-file=kubernetes.pem --key-file=kubernetes-key.pem --ca-file=ca.pem cluster-health
    member a9b850345993890 is healthy: got healthy result from https://10.66.0.68:2379
    member bf8c6a606a699629 is healthy: got healthy result from https://10.66.0.67:2379
    member c23f3e1a72e36f0c is healthy: got healthy result from https://10.66.0.69:2379
    cluster is healthy



下一步：[配置和启动k8s 控制节点](06-k8s-controllers.md)