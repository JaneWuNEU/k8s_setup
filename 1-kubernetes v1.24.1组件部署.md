### 1. Kubernetes v1.24.1的特点

在kubernetes中删除了对于docker的原生支持具体可以参考 [[2-CRI容器运行标准]]

### 2. Kubernetes v1.24.1 组件的部署

其中Kubernetes的架构图：
![[Pasted image 20221207161928.png|500]]

其中容器运行时的架构图：
![[Pasted image 20221207162727.png|500]]
还是再来解释一下：

1. low level：直接在操作系统内核中去创建容器
2. high level：提供给用户的操作接口

其中Kubernetes管理容器的方式如下所示：
![[Pasted image 20221207162833.png|500]]
其中我们可以看到containerd与runC之间还有一层，这层可以在[[2-CRI容器运行标准]]看到其中还需要一个 `containerd-shim` 来操作OCI标准的容器。

下面是具体的kubelet如何创建容器的过程，随着时间的改变：
![[Pasted image 20221207163521.png|500]]

在这个图中：

1. `dockershim cri-docker` 是docker企业维护的
2. `cri-containerd` 是谷歌维护用来调用docker所贡献的containerd项目
3. `cri-o`是谷歌用来维护完全取代docker的方式

因此此时Kubernetes一共有三种部署方式：

1. containerd：默认情况下，Kubernetes采用该方式部署
2. Docker：Kubernetes在默认情况下废除了对docker的支持，需要借助 `cri-docker` 插件来实现
3. CRI-O：在创建集群的时候需要借助`cri-o`来实现

### 3. 基本环境部署

#### 3.1 设置网络以及主机名

首先需要设置镜头的IP地址，具体可以查看 [[1-ubuntu定制网络]]
在本次实践中，其基本信息：

```text
192.168.66.100 slave0 
192.168.66.101 slave1 
192.168.66.90 master
```

然后设置主机名：

```shell
sudo hostnamectl set-hostname <NodeName>
```

#### 3.2 设置交换分区

我们需要禁止交换分区：

```shell
# 注释掉有 swap的那一行
vim /etc/fstab
mount -a

swapoff -a

free -g
```

#### 3.3 网络内核模版

首先添加新的网络模块：

```shell
tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF
```

然后加载模块组件：

```shell
modprobe overlay
modprobe br_netfilter

# 验证
modinfo br_netfilter
modinfo overlay
```

#### 3.4 调整数据包的转发

设置数据转发方式：

```shell
tee /etc/sysctl.d/kubernetes.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
```

配置生效：

```shell
sysctl --system
```

#### 3.5 配置软件源

具体可以参考如下方式：
在ubuntu上使用阿里源安装：

```shell
apt update && apt install -y apt-transport-https
apt update && apt install -y apt-transport-https
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add - 
cat << EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
```

>  其中画 kubernetes-xenial 不需要修改

下载工具：

```shell
apt-get update
apt-get install -y kubelet kubeadm kubectl

# 同时限制软件更新
sudo apt-mark hold kubelet kubeadm kubectl
kubelet set on hold.
kubeadm set on hold.
kubectl set on hold.
```

> 如果需要指定安装 [[4-下载指定版本的应用]]
> `apt install kubelet=1.22.16-00`
> `apt install kubeadm=1.22.16-00`
> `apt install kubectl=1.22.16-00`

可以验证一下：

```shell
crictl images
WARN[0000] image connect using default endpoints: [unix:///var/run/dockershim.sock unix:///run/containerd/containerd.sock unix:///run/crio/crio.sock unix:///var/run/cri-dockerd.sock]
```

### 4.命令说明

我们可以查看对应的 `kubelet` 的参数：

```shell
ls /etc/systemd/system/kubelet.service.d/10-kubeadm.conf 
vim /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
```

读取该文件可以得到`kubelet`的对应参数：

```shell
# Note: This dropin only works with kubeadm and kubelet v1.11+
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
# This is a file that "kubeadm init" and "kubeadm join" generates at runtime, populating the KUBELET_KUBEADM_ARGS variable dynamically
EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env
# This is a file that the user can use for overrides of the kubelet args as a last resort. Preferably, the user should use
# the .NodeRegistration.KubeletExtraArgs object in the configuration files instead. KUBELET_EXTRA_ARGS should be sourced from this file.
EnvironmentFile=-/etc/default/kubelet
ExecStart=
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS
```
