# 部署traefik ingress controller #

traefik项目地址：[https://github.com/containous/traefik](https://github.com/containous/traefik)

本文根据traefik在kubernetes中的部署文档进行实验。

https://docs.traefik.io/user-guide/kubernetes/

## 部署traefik ClusterRole和ClusterRoleBinding ##

    kubectl create -f https://raw.githubusercontent.com/containous/traefik/master/examples/k8s/traefik-rbac.yaml

## 使用DaemonSet部署Trafik ##

    kubectl create -f https://raw.githubusercontent.com/containous/traefik/master/examples/k8s/traefik-ds.yaml

> 使用deployment和 daemonset部署yaml文件没有什么大的不同，在deployment部署中建议添加

      nodeSelector:
        role: traefit


同时给需要做反向代理的入口node节点打上对应的label ，使用
kubectl get nodes --show-labels进行验证

## 部署trafik管理UI ##

 kubectl create -f https://raw.githubusercontent.com/containous/traefik/master/examples/k8s/ui.yaml

> 根据实际情况修改rules

<pre>
<code>
  rules:
  - host: traefik.domain.local   #修改为DNS能解析到的DOMAIN就可以通过此domain进行UI访问了。
 
</code>
</pre> 

下一步：[部署Heapster add-on](13-Heapster.md)