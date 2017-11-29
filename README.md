# Kubernetes 手动部署 #

本文档为k8s在CentOS 7中手动部署kubernetes集群的步骤。

## 集群信息 ##

- [kubernetes](https://github.com/kubernetes/kubernetes) 1.8.4
- [检查最新稳定版本](https://storage.googleapis.com/kubernetes-release/release/stable.txt): curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt
- [flannel](https://github.com/coreos/flannel/) v0.9.1
- [etcd](https://github.com/coreos/etcd) v3.2.8

## 实验步骤 ##

- [环境准备](docs/01-prerequisites.md)
- [客户端工具安装](docs/02-clinet-tools.md)
- [CA和TLS证书](docs/03-certificate.md)
- [生成kubernetes 认证配置文件和数据加密配置和密钥](docs/04-kubernetes-configuration-files.md)
- [配置和启动 etcd集群](docs/05-etcd.md)
- [配置和启动kubernetes control组件](docs/06-k8s-controllers.md)
- [配置flannel插件](docs/07-flannel.md)
- [配置和启动kubernetes worker节点](docs/08-k8s-workers.md)
- [配置kuberctl](docs/09-kubectl.md)
- [部署DNS插件](docs/10-dns-addon.md)
- [部署kubernetes dashboard](docs/11-kubernetes-dashboard.md)
- [部署traefik ingress controller](docs/12-traefik-ingress.md)
- [部署Heapster插件](docs/13-Heapster.md)




