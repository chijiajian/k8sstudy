# 提供CA和生成TLS证书 #
## CA ##

cfssl print-defaults config > ca-config.json

编辑ca-config.json文件如下：

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


创建CA认证签名请求

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

生成CA证书和私钥
cfssl gencert -initca ca-csr.json | cfssljson -bare ca

生成

ca.pem
ca-key.pem

# 客户端和服务器证书 #
Admin Client证书
admin-csr.json

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

生成admin客户端证书和密钥

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin

生成：
admin.pem
admin-key.pem

Kubelet客户端证书

worker节点证书配置文件脚本

node-csr.sh

for instance in svrxjk8sworker01 svrxjk8sworker02 svrxjk8sworker03; do
cat > ${instance}-csr.json <<EOF
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

结果：

svrxjk8sworker01-csr.json
svrxjk8sworker02-csr.json
svrxjk8sworker03-csr.json


cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=svrxjk8sworker01,10.66.0.71 \  
  -profile=kubernetes \
  svrxjk8sworker01-csr.json | cfssljson -bare svrxjk8sworker01

生成worker节点密钥

生成
svrxjk8sworker01.pem
svrxjk8sworker01-key.pem

依次再生成svrxjk8sworker02,svrxjk8sworker03 的密钥

kube-proxy客户端证书
kube-proxy签名请求
kube-proxy-csr.json

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


生成kube-proxy证书和密钥

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-proxy-csr.json | cfssljson -bare kube-proxy

生成


kube-proxy.pem
kube-proxy-key.pem

Kubernetes API Server 证书
Kubernetes API Server签名请求

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


kubernetes-csr.json

生成 Kubernetes API Server证书和密钥

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=192.168.0.1,10.66.0.67,10.66.0.68,10.66.0.69,10.66.0.75,127.0.0.1,kubernetes.default \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes

192.168.0.1为K8S_SERVICE_IP，10.66.0.67,68,69为Master节点IP，10.66.0.75为负载均衡IP
kubernetes.pem
kubernetes-key.pem


分发证书
scp ca.pem svrxjk8sworker01.pem svrxjk8sworker01-key.pem 到 svrxjk8sworker01 节点
依次分发到svrxjk8sworker02，svrxjk8sworker03节点

scp  ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem 到SvrXJK8sMaster01,SvrXJK8sMaster02,SvrXJK8sMaster03控制节点