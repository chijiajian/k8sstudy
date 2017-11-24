# 客户端工具安装 #
需要安装[csffl](https://github.com/cloudflare/cfssl),[cfssljson](https://github.com/cloudflare/cfssl)和[kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl), 代理地址：http_proxy=http://10.0.193.33:1080

## 安装CFSSL ##
cfssl和cfssljson来生成PKI和TLS证书

    curl -x $http_proxy -OL https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
    curl -x $http_proxy -OL https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
    
    chmod +x cfssl_linux-amd64 cfssljson_linux-amd64
    
    cp cfssl_linux-amd64 /usr/local/bin/cfssl
    cp cfssljson_linux-amd64 /usr/local/bin/cfssljson

## 验证 ##
cfssl version 1.2.0或以上
cfssl version

输出

Version: 1.2.0
Revision: dev
Runtime: go1.6

## 安装kubectl  ##
kubectl 工具提供和kubernetes API服务器进行交互。

    curl -x $http_proxy -OL https://storage.googleapis.com/kubernetes-release/release/$(curl -x $http_proxy -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
    
    chmod +x kubectl
    
    cp kubectl /usr/local/bin/

## 验证 ##
kubectl version --client

输出：

Client Version: version.Info{Major:"1", Minor:"8", GitVersion:"v1.8.0", GitCommit:"6e937839ac04a38cac63e6a7a306c5d035fe7b0a", GitTreeState:"clean", BuildDate:"2017-09-28T22:57:57Z", GoVersion:"go1.8.3", Compiler:"gc", Platform:"linux/amd64"}


下一步：[提供CA和生成TLS证书](03-certificate.md)

