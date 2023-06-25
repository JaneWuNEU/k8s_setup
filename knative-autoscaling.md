knative auto-scaling setups

# 配置并行度

concurrency 控制的是每个replica可以同时处理的requests的数目，同样可以做per-revision的设置和global的设置

    # container-concurrency specifies the maximum number
    # of requests the Container can handle at once, and requests
    # above this threshold are queued.  Setting a value of zero
    # disables this throttling and lets through as many requests as
    # the pod receives.

```
kubectl patch configmap/config-defaults \
  --namespace knative-serving \
  --type merge \
  --patch '{"data":{"container-concurrency":"1"}}'
```

# 配置resource-size

```控制ksvc最大的concurrent pods数目
kubectl patch configmap/config-autoscaler \
  --namespace knative-serving \
  --type merge \
  --patch '{"data":{"max-scale":6}}'

kubectl patch configmap/config-autoscaler \
  --namespace knative-serving \
  --type merge \
  --patch '{"data":{"scale-down-delay":"0s"}}'#设置保活时间，默认是0s

kubectl patch configmap/config-defaults \
  --namespace knative-serving \
  --type merge \
  --patch '{"data":{"revision-cpu-limit":"1000m"}}'
  
kubectl patch configmap/config-defaults \
  --namespace knative-serving \
  --type merge \
  --patch '{"data":{"revision-memory-limit":"500M"}}'
```

# 配置执行位置

```限制pod执行创建在特定node上
kubectl patch configmap/config-features \
   --namespace knative-serving \
   --type merge \
   --patch '{"data":{"kubernetes.podspec-nodeselector":"allowed"}}'
```

```
kubectl edit ksvc $service-name -n $target-namespace -o yaml
#在yaml文件中加入如下内容
spec:
  template:
    spec:
      nodeSelector:
        kubernetes.io/hostname: slave0
```

# 配置DNS

修改kourier的类型，否则kourier的external-ip一直是pending状态

```修改kourier的类型，否则kourier的external-ip一直是pending状态
kubectl patch svc kourier -n kourier-system -p '{"spec": {"type": "NodePort"}}'
```

# 查看kourier-internel的ip

```查看kourier-internel的ip
kubectl get svc -A
```

# 配置/etct/hosts

```
sudo vim /etc/hosts
$kourier-internel ip  $url of service
#比如：
#10.110.72.44  net-intensive.default.svc.cluster.local 
#10.110.72.44  cpu-intensive.default.svc.cluster.local 
sudo reboot
```

# 运行测试代码

## 测试代码

```
from locust import HttpUser, task
import logging
import time
import json
import uuid
class HelloWorldUser(HttpUser):
    @task
    def hello_world(self):
        start = int(round(time.time() * 1000))
        data = {"start":start}
        #http://cpu-intensive.default.svc.cluster.local
        with self.client.get("http://cpu-intensive.default.svc.cluster.local",catch_response=True,json=data) as response:
            # the keys in response text includes createTime,receiveTime,returnTime, and execTime
            # which means the time when the request is created by clients,
            # received, returned, and executed by servers.
            # Note that execTime = returnTime - receiveTime
            try:
                end = int(round(time.time() * 1000))
                if response.status_code == 200:
                    data['status'] = response.status_code
                    tmp = json.loads(response.text.strip())
                    data['exc'] = tmp['execTime']
                    data['e2e'] = end - start
                    data['non-exc'] = data['e2e'] - data['exc']
                    data['end'] = end
                    data['receive'] = tmp['receiveTime']
                    data['return'] = tmp['returnTime']
                    data['userId'] = self.userId
            except Exception as e:
                print(e)
        logging.info(data)
        print(data)
    def on_start(self):
        self.userId = str(uuid.uuid4())


```

## 测试命令

不存储logs信息

```
locust -f ./load-cpu-intensive.py --headless --user 1 -t 1m -H http://cpu-intensive.default.svc.cluster.local
```

存储logs信息

```
locust -f ./load-cpu-intensive.py --headless --user 1 -t 1m \
-H http://cpu-intensive.default.svc.cluster.local 
--logfile $log-file-path
```

# 注意事项

1. 如果发现配置的超参数不生效时，尝试delete service重新部署

2. func deploy出错不意味着deploy不成功，需要执行kn service list查看具体的部署情况

3. 


