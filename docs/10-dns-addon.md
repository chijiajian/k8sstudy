# 部署DNS插件 #

kubernetes的服务发现有使用环境变量和DNS两种方式，本章节部署DNS插件来提供kubernetes的服务发现。

## DNS Cluster插件脚本##

kubectl create -f kube-dns.yaml

## 验证： ##
kubectl get pods --namespace=kube-system | grep kube-dns

> 输出

<pre>
<code>
kube-dns-7797cb8758-66sz9        3/3       Running   0          9m
kube-dns-7797cb8758-z7zh5        3/3       Running   0          9m
</code>
</pre>

 跑一个dns tool的pod进行测试
    kubectl run -it --rm --restart=Never --image=infoblox/dnstools:latest dnstools
<pre>
<code>
dnstools# nslookup kubernetes
Server:         192.168.0.10
Address:        192.168.0.10#53

Non-authoritative answer:
Name:   kubernetes.default.svc.cluster.local
Address: 192.168.0.1
</code>
</pre>


 下一步：[部署kubernetes dashboard](11-kubernetes-dashboard.md)