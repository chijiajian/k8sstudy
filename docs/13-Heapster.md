# 部署heapster插件 #
默认安装的dashboard没有统计CPU和内存用量的图表，使用heapster插件来直观显示各组件的资源消耗情况。

heapster项目地址：[https://github.com/kubernetes/heapster](https://github.com/kubernetes/heapster)

根据https://github.com/kubernetes/heapster/tree/master/deploy/kube-config/influxdb 官方文档来部署后端为influxdb的heapster。

其中此文档应添加ClusterRoleBinding
<pre>
<code>
cat heapster-rbac.yaml 

kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: heapster
subjects:
  - kind: ServiceAccount
    name: heapster
    namespace: kube-system
roleRef:
  kind: ClusterRole
  name: system:heapster
  apiGroup: rbac.authorization.k8s.io

</code>
</pre> 



## 增加grafana-ingress，以便通过域名进行访问 ##
<pre>
<code>
cat grafana-ing.yaml 
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: grafana
  namespace: kube-system
spec:
  rules:
  - host: grafana.domain.local
    http:
      paths:
      - backend:
          serviceName: monitoring-grafana
          servicePort: 80
        path: /

</code>
</pre> 

下一步：[部署VMWare harbor私有仓库](14-harbor.md)