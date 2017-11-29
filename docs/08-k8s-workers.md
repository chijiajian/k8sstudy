# 配置和启动kubernetes Worker 节点 #


启用3个kubernetes worker节点， 需要安装kubelet和kube-proxy

## docker安装和配置 ##

通过官方脚本进行安装。

    $ curl -fsSL get.docker.com -o get-docker.sh
    $ sudo sh get-docker.sh

## 配置docker ##

### docker.service配置 ###
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
ExecStart=/usr/bin/dockerd $DOCKER_NETWORK_OPTIONS
ExecReload=/bin/kill -s HUP $MAINPID
ExecStartPost=/sbin/iptables -A FORWARD -s 0.0.0.0/0 -j ACCEPT  
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

> ExecStart=/usr/bin/dockerd $DOCKER_NETWORK_OPTIONS 此处添加的环境变量来修改网络配置，网络使用flannel插件所生成的网段信息。
> 
> ExecStartPost=/sbin/iptables -A FORWARD -s 0.0.0.0/0 -j ACCEPT   docker 从 1.13 版本开始，将 iptables FORWARD chain的默认策略设置为DROP，从而导致 ping 其它 Node 上的 Pod IP 失败，遇到这种情况时，需要手动设置策略为 ACCEPT


### docker环境变量和代理 ###

创建目录/etc/systemd/system/docker.service.d 创建http-proxy.conf和40-flannel.conf文件，http-proxy.conf指明代理服务器地址，用于内网docker节点无法访问Internet的情况，40-flannel.conf文件指明 flannel插件生成的环境变量路径。

    mkdir -p /etc/systemd/system/docker.service.d
    

    cat http-proxy.conf

    [Service]
    Environment="HTTP_PROXY=http://10.0.193.33:1080/"

    cat 40-flannel.conf 
    
    [Unit]
    Requires=flanneld.service
    After=flanneld.service
    [Service]
    EnvironmentFile=/run/flannel/docker


为了加快下载docker镜像的速度，也可以使用阿里云作为镜像加速器，通过编辑/etc/docker/daemon.json进行。具体请参考：[阿里云镜像加速](https://yq.aliyun.com/articles/29941)


### docker服务启动 ###

    $ sudo systemctl daemon-reload
    $ sudo systemctl enable docker
    $ sudo systemctl start docker

同时检查确保docker0的网段和flannel1.1网段在同一个地址段


<pre>
<code>
3: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN 
    link/ether 96:c0:f4:23:51:a9 brd ff:ff:ff:ff:ff:ff
    inet 192.168.52.0/32 scope global flannel.1
       valid_lft forever preferred_lft forever
    inet6 fe80::94c0:f4ff:fe23:51a9/64 scope link 
       valid_lft forever preferred_lft forever
4: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP 
    link/ether 02:42:92:47:a6:38 brd ff:ff:ff:ff:ff:ff
    inet 192.168.52.1/24 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:92ff:fe47:a638/64 scope link 
       valid_lft forever preferred_lft forever
</pre>
</code>


## worker节点需要安装和配置kubelet和kube-proxy ##

<pre>
<code>
curl -x $http_proxy -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -x $http_proxy -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/$workers
其中workers为kube-proxy和kubelet

sudo mkdir -p \
  /etc/cni/net.d \
  /opt/cni/bin \   #如果后续用到其他网络插件的默认路径
  /var/lib/kubelet \
  /var/lib/kube-proxy \
  /var/lib/kubernetes \
  /var/run/kubernetes

chmod +x  kube-proxy kubelet
cp  kube-proxy kubelet /usr/local/bin/

创建cat /etc/cni/net.d/10-flannel.conf
{
  "name": "podnet",
  "type": "flannel",
  "delegate": {
    "isDefaultGateway": true
  }
}
</pre>
</code>

## 配置kubelet ##

    cp ${HOSTNAME}-key.pem ${HOSTNAME}.pem /var/lib/kubelet/
    cp ${HOSTNAME}.kubeconfig /var/lib/kubelet/kubeconfig
    cp ca.pem /var/lib/kubernetes/

创建kubelet.service服务:
<pre>
<code>
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
ExecStart=/usr/local/bin/kubelet \
  --allow-privileged=true \
  --anonymous-auth=false \
  --authorization-mode=Webhook \
  --client-ca-file=/var/lib/kubernetes/ca.pem \
  --cluster-dns=192.168.0.10 \
  --cluster-domain=cluster.local \
  --container-runtime=docker \
  --image-pull-progress-deadline=2m \
  --kubeconfig=/var/lib/kubelet/kubeconfig \
  --pod-cidr=192.168.0.0 \
  --register-node=true \
  --require-kubeconfig \
  --runtime-request-timeout=15m \
  --tls-cert-file=/var/lib/kubelet/svrxjk8sworker02.pem \
  --tls-private-key-file=/var/lib/kubelet/svrxjk8sworker02-key.pem \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
</pre>
</code>

> 默认的hostname为worker机器名，DNS服务器要能解析。也可以用  --hostname-override=worker.domain.com  域名或IP的格式，如果修改相关证书都要有对应



## 配置kubernetes proxy##


<pre>
<code>
cp kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig

cat /etc/systemd/system/kube-proxy.service 
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \
  --cluster-cidr=192.168.0.0/16 \
  --kubeconfig=/var/lib/kube-proxy/kubeconfig \
  --proxy-mode=iptables \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target

cp kubelet.service kube-proxy.service /etc/systemd/system/
systemctl daemon-reload
systemctl enable  kubelet kube-proxy
systemctl status  kubelet kube-proxy
</pre>
</code>

## 检查 ##
<pre>
<code>
[root@SvrXJK8sMaster01 config]# kubectl get nodes
NAME               STATUS    ROLES     AGE       VERSION
10.66.0.71         Ready     <none>    1d        v1.8.0   #这个节点就是用--hostname-override修改过的
svrxjk8sworker02   Ready     <none>    2d        v1.8.0
svrxjk8sworker03   Ready     <none>    2d        v1.8.0
</pre>
</code>


下一步：[配置kubectl进行远程访问和管理](09-kubectl.md)


 


