# K8S集群elasticsearch-fluentbit-kibana部署方案

## fluent-bit [中文文档](https://hulining.gitbook.io/fluentbit/) | [英文文档](https://docs.fluentbit.io/manual/)

Fluent Bit 建议作为 DaemonSet 部署，这样就可以在 Kubernetes 集群的每个节点上使用它。首先，请使用以下命令来创建名称空间，服务帐号和角色设置(namespace, serviceaccount, role)

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
kubectl exec $(kubectl get pods -n elastic | grep elasticsearch-client | sed -n 1p | awk '{print $1}') \
    -n logging \
    -- bin/elasticsearch-setup-passwords auto -b
```

### 修改密码
```
curl -XPUT -u elastic:${password} 'http://localhost:9200/_xpack/security/user/elastic/_password' -d '{ "password" : "new_passwd" }'
```

### 忘记管理员密码的重置方案
1. 创建一个管理员账户
```
bin/elasticsearch-users useradd admin -p admin123 -r superuser
```
2. 使用创建的管理员密码重置
```
curl -u admin -XPUT 'http://localhost:9200/_xpack/security/user/elastic/_password?pretty' -H 'Content-Type: application/json' -d '{"password" : "new_password"}' 
```

### 保存密码到Secret对象中
```
kubectl create secret generic elasticsearch-pw-elastic \
    -n logging \
    --from-literal password=${生成的密码}
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
