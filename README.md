# 日志收集系统-ELK

## 流程
![](/ELK.jpg)

## fluent-bit

Fluent Bit 必须作为 `DaemonSet` 部署，这样就可以在 `Kubernetes` 集群的每个节点上使用它。首先，请使用以下命令来创建名称空间，服务帐号和角色设置(`namespace`, `serviceaccount`, `role``)

### 部署
```
kubectl create namespace logging
kubectl create -f https://raw.githubusercontent.com/fluent/fluent-bit-kubernetes-logging/master/fluent-bit-service-account.yaml
kubectl create -f https://raw.githubusercontent.com/fluent/fluent-bit-kubernetes-logging/master/fluent-bit-role.yaml
kubectl create -f https://raw.githubusercontent.com/fluent/fluent-bit-kubernetes-logging/master/fluent-bit-role-binding.yaml
```

创建ConfigMap以及DaemonSet
```
kubectl create -f fluentbit/deployments/k8s.yaml
```

## elasticsearch
日志持久化组件，以及查询组件。

### 部署
#### 主节点
```
kubectl apply -f elasticsearch/elasticsearch-master.deployment.yaml
```

#### 数据节点
```
kubectl apply -f elasticsearch/elasticsearch-data.statefulset.yaml
```

#### 客户端节点
```
kubectl apply -f elasticsearch/elasticsearch-client.deployment.yaml
```

### 查看状态
```
kubectl get pods -n logging -l app=elasticsearch
```

### 生成密码
我们启用了 xpack 安全模块来保护我们的集群，所以我们需要一个初始化的密码。我们可以执行如下所示的命令，在客户端节点容器内运行 bin/elasticsearch-setup-passwords 命令来生成默认的用户名和密码
```
kubectl exec $(kubectl get pods -n logging | grep elasticsearch-client | sed -n 1p | awk '{print $1}') \
    -n logging \
    -- bin/elasticsearch-setup-passwords auto -b
```
保存密码到Secret对象中
```
kubectl create secret generic elasticsearch-pw-elastic \
    -n logging \
    --from-literal password=${生成的密码}
```
修改密码
```
curl -H "Content-Type:application/json" -XPOST -u elastic 'http://localhost:9200/_xpack/security/user/elastic/_password' -d '{ "password" : "new password" }'
```

验证密码
```
curl -u elastic 'http://localhost:9200/_xpack/security/_authenticate?pretty'
```

## logstash
日志的过滤聚合组件。采集kafka日志消息队列，做日志的过滤聚合，并输出到es集群中去。

### 部署
```
kubectl apply -f logstash/logstash.deployment.yaml
```

## metricbeat
### 部署
服务metric指标收集的强大利器，可以采集kafka，logstash，kube-metric，system，docker等状态指标，并且es，kibana交互，可以展示非常友好的Dashboard。  
当你想要收集如kafka、logstash类似的deployment服务的时候，metricbeat可以选择deployment方式部署。  
当你想要收集node节点指标时，那么metricbeat需要以daemonset方式部署。  

```
kubectl apply -f metricbeat/metricbeat.deployment.yaml
```

## kibana
### 部署
```
kubectl apply -f kibana/kibana.deployment.yaml
```
### 查看状态
```
kubectl get svc kibana -n logging
```

## 可能存在的瓶颈
- elasticsearch磁盘IO瓶颈，可采用固态盘硬件，热节点。（需注意硬件隔离）
- logstash消费速度慢，可增加实例，注意consumer thread、pipeline thread的合理配置

##  todo

- 日志采集组件选用filebeat，ELK Stack可以非常友好的集成。
- elasticsearch采用集群方式部署，可横向扩展
- logstash多节点部署。需要合理配置consumer thread以保证最大的消费速度
