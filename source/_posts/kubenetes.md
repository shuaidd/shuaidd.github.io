---
layout: kubenetes
title: kubenetes
date: 2023-10-24 14:16:59
tags:
---

查看pod 允许情况
```
kubectl get pod -o wide
 
kubeadm init --kubernetes-version=v1.26.1     --apiserver-advertise-address=192.168.245.130  --pod-network-cidr=10.244.0.0/16 --image-repository=registry.aliyuncs.com/google_containers
```
重新生成join token

```
kubeadm token create --print-join-command
```

创建dashboard ui 登陆token

```
kubectl -n kubernetes-dashboard create token admin-user --duration 10m
```

进入pod内部

```
kubectl exec -it nginx-674dc8d944-srxrz -- /bin/sh
```

拷贝授权文件到节点机器

```
scp $HOME/.kube/config root@192.168.245.133:$HOME/.kube/config
```

查看pod标签

```
kubectl get pods --show-labels
```

deployment

```
更新
kubectl edit deployment/nginx-deployment

查看
kubectl describe deployments
kubectl describe deployment nginx-deployment

扩缩容
kubectl scale -n default deployment nginx-deployment --replicas=3

基于CPU利用率的自动扩缩容
kubectl autoscale deployment/nginx-deployment --min=10 --max=15 --cpu-percent=80

查看上线状态
kubectl rollout status deployment/nginx-deployment

更新镜像
kubectl set image deployment.v1.apps/nginx-deployment nginx=nginx:1.16.1
或
kubectl set image deployment/nginx-deployment nginx=nginx:1.16.1

查看历史修订版本
kubectl rollout history deployment/nginx-deployment

查看具体版本的修订信息
kubectl rollout history deployment/nginx-deployment --revision=2

回滚到上一个版本
kubectl rollout undo deployment/nginx-deployment

回滚到指定版本
kubectl rollout undo deployment/nginx-deployment --to-revision=2

暂停上线
kubectl rollout pause deployment/nginx-deployment
恢复上线
kubectl rollout resume deployment/nginx-deployment
```
