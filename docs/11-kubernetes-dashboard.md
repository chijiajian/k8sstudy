# 部署dashboard #

官方项目地址：[https://github.com/kubernetes/dashboard](https://github.com/kubernetes/dashboard)


## 下载安装yaml文件 ##

    curl -x $https_proxy -OL https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
    
    kubectl create -f kubernetes-dashboard.yaml 


> 输出

    secret "kubernetes-dashboard-certs" created
    serviceaccount "kubernetes-dashboard" created
    role "kubernetes-dashboard-minimal" created
    rolebinding "kubernetes-dashboard-minimal" created
    deployment "kubernetes-dashboard" created
    service "kubernetes-dashboard" created


> 报错：

    Error from server: Get https://svrxjk8sworker03:10250/containerLogs/kube-system/kubernetes-dashboard-d575c6f74-bmtbv/kubernetes-dashboard?follow=true: dial tcp: lookup svrxjk8sworker03 on 10.0.1.5:53: server misbehaving

> 提示：

之前提示过在部署workers kubelet时候可以修改 --hostname-override=worker.domain.com  域名或IP的格式，确保DNS能正确解析。

## 访问： ##

    kubectl proxy --address=0.0.0.0 --port=8001 --accept-hosts='^*$'

在生产环境请注意修改合适的--accept-hosts,如使用本文所示的'^*$'会有安全风险。

访问dashboard官方给定的是默认的最小RBAC权限，默认的RBAC权限不足访问对应资源内容



## 关于dashboard的访问权限wiki ##

[https://github.com/kubernetes/dashboard/wiki/Access-control](https://github.com/kubernetes/dashboard/wiki/Access-control)

官方说明：授予admin权限给Dashboard的Service Account会有安全风险，本文暂用此方案展现dashboard。

<pre>
<code>
cat dashboard-admin.yaml 
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard
  labels:
    k8s-app: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard
  namespace: kube-system

</code>
</pre> 

