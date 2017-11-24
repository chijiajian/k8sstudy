# 提供CA和生成TLS证书 #
使用 cfssl生成kubernetes各组件所需的TLS证书，提供etcd,kube-apiserver,kubelet和kube-proxy的通讯加密
## CA ##

cfssl print-defaults config > ca-config.json

编辑ca-config.json文件如下：

<pre>
<code>
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
</code>
</pre>   




创建CA认证签名请求:
<pre>
<code>
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "Shanghai",
      "O": "Kubernetes",
      "OU": "easthope",
      "ST": "Shanghai"
    }
  ]
}
</code>
</pre> 

生成CA证书和私钥:

cfssl gencert -initca ca-csr.json | cfssljson -bare ca

> 生成以下文件：

ca.pem

ca-key.pem

# 客户端和服务器证书 #
### Admin Client证书 ###
cat admin-csr.json

<pre>
<code>
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "Shanghai",
      "O": "system:masters",
      "OU": "Kubernetes easthope",
      "ST": "Shanghai"
    }
  ]
}
</code>
</pre> 


生成admin客户端证书和密钥:
<pre>
<code>
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin
</code>
</pre> 


> 生成以下文件：

admin.pem

admin-key.pem

### Kubelet客户端证书 ###

worker节点证书配置文件脚本:

<pre>
<code>
cat node-csr.sh

for instance in svrxjk8sworker01 svrxjk8sworker02 svrxjk8sworker03; do
cat &gt; ${instance}-csr.json &lt;&lt;EOF
{
  "CN": "system:node:${instance}",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "Shanghai",
      "O": "system:nodes",
      "OU": "Kubernetes easthope",
      "ST": "Shanghai"
    }
  ]
}
EOF
done
</pre>
</code>

> 生成以下文件：

svrxjk8sworker01-csr.json

svrxjk8sworker02-csr.json

svrxjk8sworker03-csr.json




生成worker节点密钥:
<pre>
<code>
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=svrxjk8sworker01,10.66.0.71 \  
  -profile=kubernetes \
  svrxjk8sworker01-csr.json | cfssljson -bare svrxjk8sworker01
</pre>
</code>



> 生成以下文件：
svrxjk8sworker01.pem

svrxjk8sworker01-key.pem

依次生成svrxjk8sworker02,svrxjk8sworker03 的密钥

### kube-proxy客户端证书 ###
kube-proxy签名请求:

cat kube-proxy-csr.json
<pre>
<code>
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "Shanghai",
      "O": "system:node-proxier",
      "OU": "Kubernetes easthope",
      "ST": "Shanghai"
    }
  ]
}
</pre>
</code>

生成kube-proxy证书和密钥:
<pre>
<code>
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-proxy-csr.json | cfssljson -bare kube-proxy
</pre>
</code>


> 生成以下文件：

kube-proxy.pem

kube-proxy-key.pem

### Kubernetes API Server 证书: ###
Kubernetes API Server签名请求:

cat kubernetes-csr.json

<pre>
<code>
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "Shanghai",
      "O": "Kubernetes",
      "OU": "Kubernetes easthope",
      "ST": "Shanghai"
    }
  ]
}
</pre>
</code>



生成 Kubernetes API Server证书和密钥:
<pre>
<code>
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=192.168.0.1,10.66.0.67,10.66.0.68,10.66.0.69,10.66.0.75,127.0.0.1,kubernetes.default \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes

</pre>
</code>

> 192.168.0.1为K8S_SERVICE_IP，10.66.0.67,68,69为Master节点IP，10.66.0.75为负载均衡IP



> 生成以下文件：

kubernetes.pem

kubernetes-key.pem


## 分发证书 ##
scp ca.pem svrxjk8sworker01.pem svrxjk8sworker01-key.pem 到 svrxjk8sworker01 节点

依次分发对应证书到svrxjk8sworker02，svrxjk8sworker03节点

scp  ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem 到SvrXJK8sMaster01,SvrXJK8sMaster02,SvrXJK8sMaster03控制节点



下一步：[生成kubernetes 配置文件](04-kubernetes-configuration-files.md)