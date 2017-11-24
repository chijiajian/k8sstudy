# 生成kubernetes 配置文件 #

# kubelet kubernetes 配置文件 #

kubeconfigconfig.sh
for instance in svrxjk8sworker01 svrxjk8sworker02 svrxjk8sworker03; do
  kubectl config set-cluster kubernetes-easthope \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://10.66.0.75:6443 \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-credentials system:node:${instance} \
    --client-certificate=${instance}.pem \
    --client-key=${instance}-key.pem \
    --embed-certs=true \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-easthope \
    --user=system:node:${instance} \
    --kubeconfig=${instance}.kubeconfig

  kubectl config use-context default --kubeconfig=${instance}.kubeconfig
done

 执行kubeconfigconfig.sh脚本获得

svrxjk8sworker01.kubeconfig
svrxjk8sworker02.kubeconfig
svrxjk8sworker03.kubeconfig

kube-proxy kubernetes 配置文件


kubectl config set-cluster kubernetes-easthope \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://https://10.66.0.75:6443 \
  --kubeconfig=kube-proxy.kubeconfig
kubectl config set-credentials kube-proxy \
  --client-certificate=kube-proxy.pem \
  --client-key=kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig
kubectl config set-context default \
  --cluster=kubernetes-easthope \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig
kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig

分发kubernetes配置文件

scp svrxjk8sworker01.kubeconfig kube-proxy.kubeconfig 到svrxjk8sworker01上
依次分发对应文件到svrxjk8sworker02，svrxjk8sworker03

生成数据加密配置文件和密钥

加密秘要
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)

加密配置文件
encryption-config.yaml
cat > encryption-config.yaml <<EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF

scp encryption-config.yaml加密文件到每个控制节点SvrXJK8sMaster01，SvrXJK8sMaster02，SvrXJK8sMaster03

