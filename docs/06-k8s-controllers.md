# 配置和启动kubernetes控制节点 #

本实验配置3台kubernetes控制节点和配置高可用性。本文使用haproxy开放Kubernetes API Server对应端口6443作为负载均衡，各云供应商或IaaS私有云提供了LB，也可开放对应端口进行负载均衡。例如阿里云的ELB。


 控制节点需要kube-apiserver,kube-controller-manager,kube-scheduler 

## 下载和安装 ##

    curl -x $http_proxy -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -x $http_proxy -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/$controlplane
    
    其中$controlplane 为kube-apiserver，kube-controller-manager，kube-scheduler
    
    chmod +x kube-apiserver kube-controller-manager kube-scheduler
    
    cp  kube-apiserver kube-controller-manager kube-scheduler  /usr/local/bin/

## 配置kubernetes API Server ##

    mkdir -p /var/lib/kubernetes/
    cp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem encryption-config.yaml /var/lib/kubernetes/

<pre>
<code>
cat kube-apiserver.service 

[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \
  --admission-control=Initializers,NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \
  --advertise-address=10.66.0.67 \
  --allow-privileged=true \
  --apiserver-count=3 \
  --audit-log-maxage=30 \
  --audit-log-maxbackup=3 \
  --audit-log-maxsize=100 \
  --audit-log-path=/var/log/audit.log \
  --authorization-mode=Node,RBAC \
  --bind-address=0.0.0.0 \
  --client-ca-file=/var/lib/kubernetes/ca.pem \
  --enable-swagger-ui=true \
  --etcd-cafile=/var/lib/kubernetes/ca.pem \
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \
  --etcd-servers=https://10.66.0.67:2379,https://10.66.0.68:2379,https://10.66.0.69:2379 \
  --event-ttl=1h \
  --experimental-encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \
  --insecure-bind-address=127.0.0.1 \
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \
  --kubelet-https=true \
  --runtime-config=api/all \
  --service-account-key-file=/var/lib/kubernetes/ca-key.pem \
  --service-cluster-ip-range=192.168.0.0/24 \
  --service-node-port-range=30000-32767 \
  --tls-ca-file=/var/lib/kubernetes/ca.pem \
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
</pre>
</code>



## 配置 Kubernetes Controller Manager ##
<pre>
<code>
cat kube-controller-manager.service

[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \
  --address=0.0.0.0 \
  --cluster-cidr=192.168.0.0/16 \
  --cluster-name=kubernetes \
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \
  --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \
  --allocate-node-cidrs=true \
  --leader-elect=true \
  --master=http://127.0.0.1:8080 \
  --root-ca-file=/var/lib/kubernetes/ca.pem \
  --service-account-private-key-file=/var/lib/kubernetes/ca-key.pem \
  --service-cluster-ip-range=192.168.0.0/24 \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
</pre>
</code>


## 配置 Kubernetes Scheduler ##
<pre>
<code>
cat kube-scheduler.service

[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \
  --leader-elect=true \
  --master=http://127.0.0.1:8080 \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
</pre>
</code>


    cp kube-apiserver.service kube-scheduler.service kube-controller-manager.service /etc/systemd/system/
    systemctl daemon-reload
    systemctl enable kube-apiserver kube-controller-manager kube-scheduler
    systemctl start kube-apiserver kube-controller-manager kube-scheduler

## 验证 ##

<pre>
<code>
[root@SvrXJK8sMaster01 config]# kubectl get componentstatuses
NAME STATUSMESSAGE  ERROR
controller-manager   Healthy   ok   
schedulerHealthy   ok   
etcd-0   Healthy   {"health": "true"}   
etcd-2   Healthy   {"health": "true"}   
etcd-1   Healthy   {"health": "true"} 
</pre>
</code>

> 三台Controller节点都进行验证

## kubelet RBAC授权 ##


<pre>
<code>
cat &gt;&gt;EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
EOF
</pre>
</code>

<pre>
<code>
cat &gt;&gt;EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF
</pre>
</code>


## 10.66.0.75 haproxy配置 ##
<pre>
<code>
frontend  k8sapi *:6443
    mode        tcp
    default_backend             k8s

backend k8s
    mode        tcp
    balance     roundrobin
    server  xjk8smaster01 10.66.0.67:6443 check
    server  xjk8smaster02 10.66.0.68:6443 check
    server  xjk8smaster03 10.66.0.69:6443 check
</pre>
</code>

## 验证 ##
<pre>
<code>
curl --cacert ca.pem https://10.66.0.75:6443/version

[root@SvrXJK8sMaster01 config]# curl --cacert ca.pem https://10.66.0.75:6443/version
{
  "major": "1",
  "minor": "8",
  "gitVersion": "v1.8.0",
  "gitCommit": "6e937839ac04a38cac63e6a7a306c5d035fe7b0a",
  "gitTreeState": "clean",
  "buildDate": "2017-09-28T22:46:41Z",
  "goVersion": "go1.8.3",
  "compiler": "gc",
  "platform": "linux/amd64"
}
</pre>
</code>


下一步：[部署flannel插件](07-flannel.md)
