关于这部分内容可以参考官方文档[runtime setup](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)

### 1. 在线安装

#### 1.1 在线安装过程

首先安装基础软件：

```shell
apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates
```

然后配置软件源：

```shell
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add - 
add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
```

然后安装 `containerd`

```shell
apt update
apt install -y containerd.io
```

可以用命令查看安装的命令：

```shell
dpkg -L containerd.io
```

命令的验证：

```shell
ctr image ls
```

#### 1.2 定制containerd

定制化`containerd`属性：

```shell
containerd config default | tee /etc/containerd/config.toml
```

修改 `/etc/containerd/config.toml` 的内容主要是修改 `k8s.io` 的镜像地址以及修改 `runC` 的`SystemdCgroup=true`

```toml
sandbox_image = "registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.6"
#修改k8s的镜像地址，否则在后续使用kubeadm安装k8s时会服务拉取镜像
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    BinaryName = ""
    CriuImagePath = ""
    CriuPath = ""
    CriuWorkPath = ""
    IoGid = 0
    IoUid = 0
    NoNewKeyring = false
    NoPivotRoot = false
    Root = ""
    ShimCgroup = ""
    SystemdCgroup = true
```

然后重启该服务

```shell
systemctl restart containerd.service
```

然后进一步定制化k8s所提供的工具 `crictl` ，修改 `/etc/crictl.yaml`

```yaml
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
```

### 2. 离线安装

### 3. runC部署
