---
layout: kubenetes
title: 内网安装k8s
date: 2023-10-25 11:20:31
tags:
---
```

一、文件准备：
docker-20.10.17
cri-dockerd-0.3.1
kubernetes_1.26.1
kube-flannel


二、docker 安装
1.修改yum.repos.d CentOS文件移动到备份文件夹
mkdir /etc/yum.repos.d/bak
mv /etc/yum.repos.d/CentOS-Linux-* /etc/yum.repos.d/bak
cp -r /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/bak

2.修改CentOS-Base.repo内容
[base]
name=CentOS-8.5.2111 - Base - mirrors.aliyun.com
baseurl=http://mirrors.aliyun.com/centos-vault/8.5.2111/BaseOS/$basearch/os/
        http://mirrors.aliyuncs.com/centos-vault/8.5.2111/BaseOS/$basearch/os/
        http://mirrors.cloud.aliyuncs.com/centos-vault/8.5.2111/BaseOS/$basearch/os/
gpgcheck=0
gpgkey=http://mirrors.aliyun.com/centos/RPM-GPG-KEY-CentOS-Official
 
#additional packages that may be useful
[extras]
name=CentOS-8.5.2111 - Extras - mirrors.aliyun.com
baseurl=http://mirrors.aliyun.com/centos-vault/8.5.2111/extras/$basearch/os/
        http://mirrors.aliyuncs.com/centos-vault/8.5.2111/extras/$basearch/os/
        http://mirrors.cloud.aliyuncs.com/centos-vault/8.5.2111/extras/$basearch/os/
gpgcheck=0
gpgkey=http://mirrors.aliyun.com/centos/RPM-GPG-KEY-CentOS-Official
 
#additional packages that extend functionality of existing packages
[centosplus]
name=CentOS-8.5.2111 - Plus - mirrors.aliyun.com
baseurl=http://mirrors.aliyun.com/centos-vault/8.5.2111/centosplus/$basearch/os/
        http://mirrors.aliyuncs.com/centos-vault/8.5.2111/centosplus/$basearch/os/
        http://mirrors.cloud.aliyuncs.com/centos-vault/8.5.2111/centosplus/$basearch/os/
gpgcheck=0
enabled=0
gpgkey=http://mirrors.aliyun.com/centos/RPM-GPG-KEY-CentOS-Official
 
[PowerTools]
name=CentOS-8.5.2111 - PowerTools - mirrors.aliyun.com
baseurl=http://mirrors.aliyun.com/centos-vault/8.5.2111/PowerTools/$basearch/os/
        http://mirrors.aliyuncs.com/centos-vault/8.5.2111/PowerTools/$basearch/os/
        http://mirrors.cloud.aliyuncs.com/centos-vault/8.5.2111/PowerTools/$basearch/os/
gpgcheck=0
enabled=0
gpgkey=http://mirrors.aliyun.com/centos/RPM-GPG-KEY-CentOS-Official


[AppStream]
name=CentOS-8.5.2111 - AppStream - mirrors.aliyun.com
baseurl=http://mirrors.aliyun.com/centos-vault/8.5.2111/AppStream/$basearch/os/
        http://mirrors.aliyuncs.com/centos-vault/8.5.2111/AppStream/$basearch/os/
        http://mirrors.cloud.aliyuncs.com/centos-vault/8.5.2111/AppStream/$basearch/os/
gpgcheck=0
gpgkey=http://mirrors.aliyun.com/centos/RPM-GPG-KEY-CentOS-Official


3.安装yum工具包
yum -y install yum-utils

4.添加docker源
yum-config-manager \
    --add-repo \
    http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
	
5.安装docker
yum localinstall /alfiedata/app/docker-20.10.17/*.rpm
6.新增 /etc/docker/daemon.json文件
vim /etc/docker/daemon.json
{
  "builder": {
    "gc": {
      "defaultKeepStorage": "20GB",
      "enabled": true
    }
  },
  "experimental": false,
  "features": {
    "buildkit": true
  },
  "registry-mirrors": [
    "https://bsxh6h1a.mirror.aliyuncs.com"
  ],
  "insecure-registries":[
    "电脑ip:5000"
  ],
  "exec-opts": ["native.cgroupdriver=systemd"]
}
7.创建代理文件
mkdir /etc/systemd/system/docker.service.d
vim /etc/systemd/system/docker.service.d/http-proxy.conf
#写入内容
[Service]
Environment="HTTP_PROXY=http://192.168.168.135:8888"
Environment="HTTPS_PROXY=http://192.168.168.135:8888"


8.设置开机自启动
systemctl start docker
systemctl enable docker.service




三、安装kubernetes

1.修改hosts配置别名
vim /etc/hosts
192.168.168.11 alfie-itkf-1
192.168.168.12 alfie-itkf-2
192.168.168.13 alfie-itkf-3
192.168.168.14 alfie-itkf-4
192.168.168.15 alfie-itkf-5

2.关闭正在使用的虚拟内存
swapoff -a
3.注释掉swap行，防止重启后使用虚拟内存
vim /etc/fstab
# /dev/mapper/centos-swap swap                    swap    defaults        0 0
# 查看是否成功关闭，swap是否都为0
free -m

4.配置网桥
vim /etc/sysctl.conf
# kubernetes config
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1

5.刷新配置
sysctl -p
# 如果提示文件缺失，初始化一下
modprobe br_netfilter

6.修改docker cgroups策略
vim /etc/docker/daemon.json
# 确认/etc/docker/daemon.json参数，不同则修改
"exec-opts": ["native.cgroupdriver=systemd"]

7.修改了上面就刷新配置并重启docker
systemctl daemon-reload && systemctl restart docker

8.安装kubernetes
yum localinstall /alfiedata/app/kubernetes_1.26.1/*.rpm
# 如果提示conflicting requests加上 --skip-broken 后面会自动安装依赖
yum localinstall --skip-broken /alfiedata/app/kubernetes_1.26.1/*.rpm

9.设置kubernetes开机自启动
systemctl enable kubelet.service

10.查看kubernetes需要安装哪些镜像
kubeadm config images list

11.根据结果将镜像pull下来
docker pull registry.k8s.io/kube-apiserver:v1.26.1
docker pull registry.k8s.io/kube-controller-manager:v1.26.1
docker pull registry.k8s.io/kube-scheduler:v1.26.1
docker pull registry.k8s.io/kube-proxy:v1.26.1
docker pull registry.k8s.io/pause:3.9
docker pull registry.k8s.io/etcd:3.5.6-0
docker pull registry.k8s.io/coredns/coredns:v1.9.3
#目前用的阿里加速服，如果有镜像pull不下来换到科大，记得换回来，科大速度慢
vim /etc/docker/daemon.json
科大镜像：https://docker.mirrors.ustc.edu.cn/

# 安装cri-dockerd
12.将解压后的文件复制到/usr/bin/
tar -xvf cri-dockerd-0.3.1.amd64.tgz
cp /alfiedata/app/cri-dockerd-0.3.1/cri-dockerd/cri-dockerd /usr/bin/

13.创建cri-dockerd启动文件
cat <<"EOF" > /usr/lib/systemd/system/cri-docker.service
[Unit]
Description=CRI Interface for Docker Application Container Engine
Documentation=https://docs.mirantis.com
After=network-online.target firewalld.service docker.service
Wants=network-online.target
Requires=cri-docker.socket
[Service]
Type=notify
ExecStart=/usr/bin/cri-dockerd --network-plugin=cni --pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.7
ExecReload=/bin/kill -s HUP $MAINPID
TimeoutSec=0
RestartSec=2
Restart=always
StartLimitBurst=3
StartLimitInterval=60s
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
TasksMax=infinity
Delegate=yes
KillMode=process
[Install]
WantedBy=multi-user.target
EOF

14.创建socket文件
cat <<"EOF" > /usr/lib/systemd/system/cri-docker.socket
[Unit]
Description=CRI Docker Socket for the API
PartOf=cri-docker.service
[Socket]
ListenStream=%t/cri-dockerd.sock
SocketMode=0660
SocketUser=root
SocketGroup=docker
[Install]
WantedBy=sockets.target
EOF

15.启动cri-docked并设置开机自启动
systemctl daemon-reload
systemctl enable cri-docker --now
# 查看是否启动成功
systemctl is-active cri-docker

16. 刷新配置
kubeadm reset

17.初始化kubernetes
# master执行，节点不用执行！！！！！！！！！！！！！！！！！！！！！
kubeadm init --kubernetes-version=v1.26.1     --apiserver-advertise-address=192.168.168.30     --pod-network-cidr=10.244.0.0/16
--image-repository=registry.aliyuncs.com/google_containers

18.创建kubernetes配置文件
# 先用root账号确认/etc/sudoers是否有写的权限
ll /etc/sudoers
# 没有就修改权限
chmod u+w /etc/sudoers
# 给应用账号赋予权限，在root	ALL=(ALL) 	ALL下面加一行
webapp2021 ALL=(ALL)       ALL
# 切换到应用账号执行
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

19.node执行加入master
kubeadm join 192.168.168.15:6443 --token pbqtxu.c0gzunury7v62th3 --discovery-token-ca-cert-hash sha256:a10a664d31ad048158cbda20274a4c14cc378deb2b6c2a7e93475cb64a1438cb --cri-socket=unix:///var/run/cri-dockerd.sock

20.安装kube-flannel插件
kubectl create -f /alfiedata/app/kube-flannel/kube-flannel.yml
```

为普通用户赋权

```
新增用户组 docker
sudo groupadd docker

添加成员到用户组
sudo usermod -aG docker $USER

newgrp docker
```

机器打通k8s网络

```
telepresence connect --kubeconfig=D:\tool\telepresence\conf\config
```
