# Kubernetes 手动部署 #

本文档为k8s在CentOS 7中手动部署kubernetes集群的步骤

## 集群信息 ##

- kubernetes 1.8.4
- 检查最新稳定版本: curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt
- flannel v0.9.1
- etcd v3.2.8

## 实验 ##

- 环境准备
- 客户端工具安装
- CA和TLS证书
- 生成kubernetes 认证配置文件和数据加密配置和密钥
- 配置和启动 etcd集群
- 配置和启动kubernetes control组件
- 配置和启动kubernetes worker节点
- 配置kuberctl

curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl


