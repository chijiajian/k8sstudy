# RBAC授权 #
## RBAC API 对象 ##
对kubernetes资源进行增、删、改、查CRUD的操作
资源有

- pod
- PersistentVolumes
- ConfigMaps
- Deployments
- Nodes
- Secrets
- Namespaces

对资源的操作有

- create
- get
- delete
- list
- update
- edit
- watch

管理kubernetes中的RBAC，需要以下元素

- Rules: 操作规则(verbs)
- Roles和ClusterRoles：Role关联单一namespace，ClusterRole是cluster级别的他的规则可以跨多个namespace，例如nodes资源
- Subjects：三个类型的subjects
	- User Accounts:global的，集群外用户帐号，不关联kubernetes集群中API Object资源对象
	- Service Accounts：集群内服务帐号，通过API进行授权的
	- Groups：关联多个用户的组

- RoleBindings和ClusterRoleBindings：绑定subject到role。RoleBinding在namespace内生效对应规则，ClusterRoleBinding在左右namespace生效对应规则

## 例子1： 创建开发用户和开发组，限定在某一个namespace里访问 ##

用户帐号信息
- username: employee-dev01
- Group: dev01

假定用户在dev的namespace下可以访问所有部署权限

### 第一步： 创建dev namespace ###

    kubectl create ns dev

### 第二步：创建用户证书 ###

- 创建私钥

     openssl genrsa -out employee-dev01.key 2048

- 创建证书签名请求。确保subj中的用户名CN字段和组字段O正确。

    openssl req -new -key employee-dev01.key -out employee-dev01.csr -subj "/CN=employee-dev01/O=dev01" 

- 生成证书

    openssl x509 -req -in employee-dev01.csr -CA /root/k8stool/kube-ssl/config/ca.pem -CAkey /root/k8stool/kube-ssl/config/ca-key.pem -CAcreateserial -out employee-dev01.crt -days 365

- 使用以上帐号添加新的上下文环境

    kubectl config set-credentials employee-dev01 --client-certificate=/root/rbac/employee-dev01.crt --client-key=/root/rbac/employee-dev01.key 
    kubectl config set-context dev-context --cluster=kubernetes-easthope --namespace=dev --user=employee-dev01


### 第三步：创建可以管理部署的Role ###

<pre>
<code>
cat role-deployment-devns.yaml

 kind: Role
  apiVersion: rbac.authorization.k8s.io/v1beta1
  metadata:
    namespace: dev
    name: deployment-manager
  rules:
  - apiGroups: ["", "extensions", "apps"]
    resources: ["deployments", "replicasets", "pods"]
    verbs: ["*"] 
</code>
</pre>

### 第四步：绑定Role到用户 ###

<pre>
<code>
cat rolebinding-deployment-mamager.yaml 

  kind: RoleBinding
  apiVersion: rbac.authorization.k8s.io/v1beta1
  metadata:
    name: deployment-manager-binding
    namespace: dev
  subjects:
  - kind: User  #也可以修改为Group
    name: employee-dev01
    apiGroup: ""
  roleRef:
    kind: Role
    name: deployment-manager
    apiGroup: ""
</code>
</pre>

执行第三步和第四步的yaml文件  kubectl create -f *

### 第五步：测试RBAC规则 ###

    kubectl --context=dev-context auth can-i get pods --namespace dev
    yes
	kubectl --context=dev-context auth can-i create deployment  --namespace dev
	yes
    
    kubectl --context=dev-context auth can-i get pods --namespace default
    no
    kubectl --context=dev-context auth can-i create deployment  --namespace kube-system
    no



