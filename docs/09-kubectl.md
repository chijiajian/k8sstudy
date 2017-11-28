# 配置kubectl进行远程访问和管理 #

## 创建kubectl kubeconfig文件 ##

admin用户认证的kubeconfig文件
<pre>
<code>

kubectl config set-cluster kubernetes-easthope \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443

kubectl config set-credentials admin \
  --client-certificate=admin.pem \
  --client-key=admin-key.pem
kubectl config set-context kubernetes-easthope \
  --cluster=kubernetes-the-hard-way \
  --user=admin
kubectl config use-context kubernetes-easthope

</pre>
</code>


> ${KUBERNETES_PUBLIC_ADDRESS}为负载均衡器的IP地址: 10.66.0.75


## 验证： ##

检查健康状态
kubectl get componentstatuses

> 输出

<pre>
<code>
NAME                 STATUS    MESSAGE              ERROR
scheduler            Healthy   ok                   
controller-manager   Healthy   ok                   
etcd-0               Healthy   {"health": "true"}   
etcd-2               Healthy   {"health": "true"}   
etcd-1               Healthy   {"health": "true"} 
</pre>
</code>



下一步：[安装DNS  add-on](10-dns-addon.md)