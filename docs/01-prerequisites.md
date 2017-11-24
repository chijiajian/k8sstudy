# Prerequisites #

## 集群机器 ##
- 10.66.0.67
- 10.66.0.68
- 10.66.0.69
- 10.66.0.71
- 10.66.0.72
- 10.66.0.73
- 10.66.0.75

## 机器说明 ##
- 67,68,69为etcd集群、Kubernetes master集群
- 71,72,73为kubernetes worker节点
- 75使用haproxy作为kubernetes api服务器的loadbalancer

## 环境参数 ##
POD_NETWORK=192.168.0.0/16

SERVICE_IP_RANGE=192.168.0.0/24  不能和POD_NETWORK地址重叠，也不能和已存在的网络架构地址重叠

K8S_SERVICE_IP=192.168.0.1  Kubernetes API 服务的虚拟IP

DNS_SERVICE_IP=192.168.0.10 此地址必须为SERVICE_IP_RANGE的地址范围

## 关于GFW问题 ##
在xx云服务商的开启shadowsocket服务
在PC上下一个客户端，并提供代理服务。本文使用的代理地址为http://10.0.193.33:1080


