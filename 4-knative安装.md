# 安装步骤

## knative serving

(安装教程)[https://knative.dev/docs/install/yaml-install/serving/install-serving-with-yaml], <https://editor.csdn.net/md/?articleId=127933377>

```
#首先要保证k8s以及kubectl已经成功安装
#安装cosign
wget "https://github.com/sigstore/cosign/releases/download/v2.0.0/cosign-linux-amd64"
sudo mv cosign-linux-amd64 /usr/local/bin/cosign
chmod +x /usr/local/bin/cosign
#安装jq
sudo apt-get install jq

#注意：以下验证步骤可省略（验证的内容是检验下载的包是否安全）
# download the yaml file, this example uses the serving manifest
curl -fsSLO https://github.com/knative/serving/releases/download/knative-v1.9.0/serving-core.yaml
cat serving-core.yaml | grep 'gcr.io/' | awk '{print $2}' > images.txt
input=images.txt
while IFS= read -r image
do
  COSIGN_EXPERIMENTAL=1 cosign verify -o text "$image" | jq
done < "$input"

#Install the required custom resources by running the command （没有被墙，可以正常执行）
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.10.1/serving-crds.yaml

#Install the core components of Knative Serving by running the command（被墙，需要特殊处理）
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.10.1/serving-core.yaml
```

### 安装&验证network layer

```
#kubectl apply -f https://github.com/knative/net-kourier/releases/download/knative-v1.10.0/kourier.yaml（被墙，需要特殊处理）

kubectl patch configmap/config-network \
  --namespace knative-serving \
  --type merge \
  --patch '{"data":{"ingress-class":"kourier.ingress.networking.knative.dev"}}'

kubectl --namespace kourier-system get service kourier
```

### 配置DNS

安装文档里的安装指南如下，但是因为国外无法访问sslip.io，下边的配置方法是无效的。

Knative provides a Kubernetes Job called `default-domain` that configures Knative Serving to use [sslip.io](http://sslip.io/) as the default DNS suffix.

```
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.10.1/serving-default-domain.yaml
```

有效的配置方式：1. 修改kourier的类型，2.将service和gateway的映射关系添加到/etc/hosts里。原理在于VM在进行域名解析时会先查询本地的/etc/hosts，如果没有找到才会查询DNS server。

```
kubectl patch svc kourier -n kourier-system -p '{"spec": {"type": "NodePort"}}'### enable autoscaling
```

在/etc/hosts内添加如下内容：

> 10.107.180.32    hello-example.default.example.com

### 配置弹性伸缩工具

Knative also supports the use of the Kubernetes Horizontal Pod Autoscaler (HPA) for driving autoscaling decisions.

```
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.10.1/serving-hpa.yaml
```
