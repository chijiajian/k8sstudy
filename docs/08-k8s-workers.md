# 配置和启动kubernetes Worker 节点 #

worker节点需要安装和配置kubelet和kube-proxy


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

## 配置kubelet ##
cp ${HOSTNAME}-key.pem ${HOSTNAME}.pem /var/lib/kubelet/
cp ${HOSTNAME}.kubeconfig /var/lib/kubelet/kubeconfig
cp ca.pem /var/lib/kubernetes/

创建kubelet.service服务
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

默认的hostname为worker机器名，DNS服务器要能解析。也可以用  --hostname-override=worker.domain.com  域名或IP的格式，如果修改相关证书都要有对应

## 配置kubernetes proxy
 ##

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

检查
[root@SvrXJK8sMaster01 config]# kubectl get nodes
NAME               STATUS    ROLES     AGE       VERSION
10.66.0.71         Ready     <none>    1d        v1.8.0   #这个节点就是用--hostname-override修改过的
svrxjk8sworker02   Ready     <none>    2d        v1.8.0
svrxjk8sworker03   Ready     <none>    2d        v1.8.0



 


