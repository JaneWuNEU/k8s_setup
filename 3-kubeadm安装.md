### 1. 在 master 上安装

下面直接给出对应的命令：

```shell
kubeadm init  --apiserver-advertise-address=192.168.1.53 --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12 --cri-socket=unix:///run/containerd/containerd.sock
```

```切换到普通用户执行以下命令
 mkdir -p $HOME/.kube #切换到普通用户执行这三条命令
 sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
 sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

在~/.bashrc内加入环境变量```export KUBECONFIG=$HOME/.kube/config ```

下面这句话是给出的提示内容：
![[Pasted image 20230415195843.png]]

```shell
kubeadm join 192.168.66.90:6443 --token fukl4m.1pglrbj5czpi9f9z \
        --discovery-token-ca-cert-hash sha256:5742b8e07a20a6a9282f285578ffa64facfbe915efdcd6931ad650f19ae49e9b 
```

### 2. 在 slave 上安装：

```shell
kubeadm join 192.168.66.90:6443 --token fukl4m.1pglrbj5czpi9f9z \
        --discovery-token-ca-cert-hash sha256:5742b8e07a20a6a9282f285578ffa64facfbe915efdcd6931ad650f19ae49e9b 
```

之后可以在 master 上通过命令查看安装情况

```shell
kubectl get node 
```

### 3. 安装网络插件

在 master 上安装网络插件，可以选择：

1. calico
2. flannel
3. cilium

本次直接选择 flannel，flannel 的地址 [Flannel](https://github.com/flannel-io/flannel)：

```shell
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

最后 k8s 安装完成：
![[Pasted image 20230415200153.png]]

### 附录

这个过程中可以通过如下命令查看镜像的状态：

```shell
crictl image list

ctr -n k8s.io images ls 
```
