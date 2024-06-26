# Kubernetes v1.20创建PVC报错

## 故障描述
PVC 显示创建不成功：kubectl get pvc -n efk 显示 Pending，这是由于版本太高导致的。k8sv1.20 以上版本默认禁止使用 selfLink。(selfLink：通过 API 访问资源自身的 URL，例如一个 Pod 的 link 可能是 /api/v1/namespaces/ns36aa8455/pods/sc-cluster-test-1-6bc58d44d6-r8hld)。

## 故障解决
``` shell
$ vi /etc/kubernetes/manifests/kube-apiserver.yaml
apiVersion: v1
···
    - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
    - --feature-gates=RemoveSelfLink=false # 添加这个配置
重启下kube-apiserver.yaml

# 如果是二进制安装的 k8s，执行 systemctl restart kube-apiserver
# 如果是 kubeadm 安装的 k8s
$ ps aux|grep kube-apiserver
$ kill -9 [Pid]
$ kubectl apply -f /etc/kubernetes/manifests/kube-apiserver.yaml
...
$ kubectl get pvc	# 查看 pvc 显示 Bound
NAME     STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
my-pvc   Bound    pvc-ae9f6d4b-fc4c-4e19-8854-7bfa259a3a04   1Gi        RWX            example-nfs    13m
```
