---
layout: post
title:  "Istio安装第一部分之k8s安装"
date:   2018-12-14 09:00:13
categories:
  - 微服务
permalink: /archivers/istio-install-part1
---

# k8s集群安装

本文主要参考
[How to Install a Kubernetes Docker Cluster on CentOS 7](https://www.howtoforge.com/tutorial/centos-kubernetes-docker-cluster/)

增加了下面两个配置

* 配置科学上网的proxy，以便直接从google下载相关软件包
* 限定docker版本

相关软件版本

* CentOS : 7.4
* Docker : 18.06
* kubernetes : 1.13.0

机器规划

* k8s-master : 192.168.1.100
* node001 : 192.168.1.101
* node002 : 192.168.1.102

## 步骤一 kubernetes安装

依次在三台机器执行下面的动作

### 配置主机


使用vi修改/etc/hosts文件

```shell
vi /etc/hosts
```

将下面的配置粘贴进去

```shell
192.168.1.100 k8s-master
192.168.1.101 node001
192.168.1.102 node002
```

保存并退出

### 禁用SELinux

```shell
setenforce 0
sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
```

### 开启br_netfilter内核模块


```shell
modprobe br_netfilter
echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables
```

### 停用swap

```shell
swapoff -a
```

然后修改/etc/fstab文件

```shell
vi /etc/fstab
```

将/dev/mapper/centos-swap一行用#注释掉，如下图

```shell
#/dev/mapper/centos-swap swap                    swap    defaults        0 0
```

### 安装Docker CE

首先安装依赖包

```shell
yum install -y yum-utils device-mapper-persistent-data lvm2
```

添加Docker仓库并安装18.06版本的Docker

```shell
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum install -y docker-ce-18.06.0.ce-3.el7
```

### 配置代理

安装kubernetes的时候，需要通过google获取相关的安装包，由于google被墙的原因，需要配置可以科学上网的代理

我在宿主机(192.168.1.2)用Burp Suit开启了一个proxy (配置ssl past though), proxy为http://192.168.1.2:8080

#### 配置系统代理

修改/etc/profile

```shell
vi /etc/profile
```

在最后加上代理

```shell
export http_proxy=http://192.168.1.2:8080
export https_proxy=http://192.168.1.2:8080
export no_proxy="localhost,127.0.0.1,192.168.1.100,192.168.1.101,192.168.1.102"
```

执行下面命令使代理生效

```shell
source /etc/profile
```

#### 配置Docker代理

创建proxy目录和文件，参考[Control Docker with systemd](https://docs.docker.com/config/daemon/systemd/#runtime-directory-and-storage-driver) HTTP/HTTPS proxy一节

```shell
mkdir -p /etc/systemd/system/docker.service.d
vi /etc/systemd/system/docker.service.d/http-proxy.conf
```

添加下面配置

```shell
[Service]
Environment="HTTP_PROXY=http://192.168.1.2:8080" "HTTPS_PROXY=http://192.168.1.2:8080" "NO_PROXY=localhost,127.0.0.1"
```

重新加载配置并且重启Docker

```shell
systemctl daemon-reload
systemctl restart docker
```

检查配置是否已经加载

```shell
systemctl show --property=Environment docker
```

### 安装kubernets

添加kubernetes仓库

```shell
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
        https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
```

安装kubernetes的包

```shell
yum install -y kubelet kubeadm kubectl
```

重启

```shell
reboot
```

重启登录系统并启动相关服务

```shell
systemctl start docker && systemctl enable docker
systemctl start kubelet && systemctl enable kubelet
```

### 修改cgroup-driver

执行下面命令

```shell
sed -i 's/cgroup-driver=systemd/cgroup-driver=cgroupfs/g' /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
systemctl daemon-reload
systemctl restart kubelet
```

### 禁用并停止firewalld

```shell
systemctl disable firewalld
systemctl stop firewalld
```

## 步骤二 初始化kubernetes集群

在k8s-master节点

```shell
kubeadm init --apiserver-advertise-address=192.168.1.100 --pod-network-cidr=10.244.0.0/16
```

出现下面文字表示初始化完成

```shell
You can now join any number of machines by running the following on each node
as root:

  kubeadm join 192.168.1.100:6443 --token vbdkrx.u1x0x5cuqmg5z2lj --discovery-token-ca-cert-hash sha256:5b3a37227b1b4cd98f08fe19429927b3e394815667f2568347f580af37e978e6
```

执行下面命令

```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

添加flannel

```shell
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

删除apiserver自动配置的proxy，不然后续kubernetes服务间的访问会出问题

```shell
vi /etc/kubernetes/manifests/kube-apiserver.yaml
```

如下面，把env和下面的proxy项给注释掉

```shell
#    env:
#    - name: http_proxy
#      value: http://192.168.1.2:8080
#    - name: https_proxy
#      value: http://192.168.1.2:8080
#    - name: no_proxy
#      value: localhost,127.0.0.1,192.168.1.100,192.168.1.101,192.168.1.102
```

## 步骤三 将node001和node002加入集群

分别在node001和node002执行下面命令

```shell
kubeadm join 192.168.1.100:6443 --token vbdkrx.u1x0x5cuqmg5z2lj --discovery-token-ca-cert-hash sha256:5b3a37227b1b4cd98f08fe19429927b3e394815667f2568347f580af37e978e6
```

## 步骤四 检查

确认所有POD都是running状态

```shell
kubectl get pods -n kube-system
```

看到如下输出

```shell
NAME                                 READY   STATUS    RESTARTS   AGE
coredns-86c58d9df4-277l9             1/1     Running   0          27m
coredns-86c58d9df4-rxzm2             1/1     Running   0          27m
etcd-k8s-master                      1/1     Running   0          26m
kube-apiserver-k8s-master            1/1     Running   0          16m
kube-controller-manager-k8s-master   1/1     Running   1          26m
kube-flannel-ds-amd64-7znqn          1/1     Running   0          13m
kube-flannel-ds-amd64-q92pc          1/1     Running   0          13m
kube-flannel-ds-amd64-w8hvw          1/1     Running   0          13m
kube-proxy-bx6vm                     1/1     Running   0          27m
kube-proxy-mjs5p                     1/1     Running   0          15m
kube-proxy-rztg5                     1/1     Running   0          14m
kube-scheduler-k8s-master            1/1     Running   1          26m
```

确认所有nodes都是Ready状态

```shell
kubectl get nodes
```

看到如下输出

```shell
NAME         STATUS   ROLES    AGE   VERSION
k8s-master   Ready    master   26m   v1.13.0
node001      Ready    <none>   13m   v1.13.0
node002      Ready    <none>   13m   v1.13.0
```
