# 使用 Fluentd 记录日志

此任务将展示如何配置 Istio 创建自定义日志条目并且发送给 [Fluentd](https://www.fluentd.org/) 守护进程。Fluentd 是一个开源的日志收集器，
支持多种[数据输出](https://www.fluentd.org/dataoutputs)并且有一个可插拔架构。
[Elasticsearch](https://www.elastic.co/products/elasticsearch)是一个流行的后端日志记录程序，
[Kibana](https://www.elastic.co/products/kibana) 用于查看。在任务结束后，
一个新的日志流将被加载发送日志到示例 Fluentd/Elasticsearch/Kibana 栈。

在任务中，将使用 [BookInfo](../../guides/bookinfo.md) 示例应用程序作为示例应用程序。

## 在开始之前
* [安装 Istio](../../setup/) 到您的集群并且部署一个应用程序。这个任务假定 Mixer 是
以默认配置设置的(`--configDefaultNamespace=istio-system`)。
如果您使用不同的值，则更新此任务中的配置和命令以匹配对应的值。

## 安装 Fluentd

在您的群集中，您可能已经有一个 Fluentd DaemonSet 运行，
就像 add-on 中[这里](https://kubernetes.io/docs/tasks/debug-application-cluster/logging-elasticsearch-kibana/)
和[这里](https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/fluentd-elasticsearch)的描述,
或者特定于您的集群提供者的东西。这可能配置为将日志发送到 Elasticsearch 系统或其它日志记录提供程序。

您可以使用这些 Fluentd 守护进程或您已经设置的任何其他Fluentd守护进程，只要Fluentd守护进程正在侦听转发的日志, 并且
Istio 的 Mixer 可以连接Fluentd守护进程。为了让 Mixer 连接到正在运行的 Fluentd 守护进程, 您可能需要为 Fluentd 添加
[service](https://kubernetes.io/docs/concepts/services-networking/service/). 
监听转发日志的 Fluentd 配置是:

```
<source>
  type forward
</source>
```

将 Mixer 连接到所有可能 Fluentd 配置的完整细节超出了此任务的范围。

### Fluentd, Elasticsearch, Kibana 栈示例

为了这个任务的准备，您可以部署提供的示例栈。 
该栈包括 Fluentd，Elasticsearch 和 Kibana 在一个非生产集合 [Services](https://kubernetes.io/docs/concepts/services-networking/service/) 和 [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
在一个新的叫做`logging`的 
[Namespace](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) 中.

将下面的内容保存为 `logging-stack.yaml`.

```
# Logging Namespace. All below are a part of this namespace.
apiVersion: v1
kind: Namespace
metadata:
  name: logging
---
# Elasticsearch Service
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch
  namespace: logging
  labels:
    app: elasticsearch
spec:
  ports:
  - port: 9200
    protocol: TCP
    targetPort: db
  selector:
    app: elasticsearch
---
# Elasticsearch Deployment
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: elasticsearch
  namespace: logging
  labels:
    app: elasticsearch
  annotations:
    sidecar.istio.io/inject: "false"
spec:
  template:
    metadata:
      labels:
        app: elasticsearch
    spec:
      containers:
      - image: docker.elastic.co/elasticsearch/elasticsearch-oss:6.1.1
        name: elasticsearch
        resources:
          # need more cpu upon initialization, therefore burstable class
          limits:
            cpu: 1000m
          requests:
            cpu: 100m
        env:
          - name: discovery.type
            value: single-node
        ports:
        - containerPort: 9200
          name: db
          protocol: TCP
        - containerPort: 9300
          name: transport
          protocol: TCP
        volumeMounts:
        - name: elasticsearch
          mountPath: /data
      volumes:
      - name: elasticsearch
        emptyDir: {}
---
# Fluentd Service
apiVersion: v1
kind: Service
metadata:
  name: fluentd-es
  namespace: logging
  labels:
    app: fluentd-es
spec:
  ports:
  - name: fluentd-tcp
    port: 24224
    protocol: TCP
    targetPort: 24224
  - name: fluentd-udp
    port: 24224
    protocol: UDP
    targetPort: 24224
  selector:
    app: fluentd-es
---
# Fluentd Deployment
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: fluentd-es
  namespace: logging
  labels:
    app: fluentd-es
  annotations:
    sidecar.istio.io/inject: "false"
spec:
  template:
    metadata:
      labels:
        app: fluentd-es
    spec:
      containers:
      - name: fluentd-es
        image: gcr.io/google-containers/fluentd-elasticsearch:v2.0.1
        env:
        - name: FLUENTD_ARGS
          value: --no-supervisor -q
        resources:
          limits:
            memory: 500Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: config-volume
          mountPath: /etc/fluent/config.d
      terminationGracePeriodSeconds: 30
      volumes:
      - name: config-volume
        configMap:
          name: fluentd-es-config
---
# Fluentd ConfigMap, contains config files.
kind: ConfigMap
apiVersion: v1
data:
  forward.input.conf: |-
    # Takes the messages sent over TCP
    <source>
      type forward
    </source>
  output.conf: |-
    <match **>
       type elasticsearch
       log_level info
       include_tag_key true
       host elasticsearch
       port 9200
       logstash_format true
       # Set the chunk limits.
       buffer_chunk_limit 2M
       buffer_queue_limit 8
       flush_interval 5s
       # Never wait longer than 5 minutes between retries.
       max_retry_wait 30
       # Disable the limit on the number of retries (retry forever).
       disable_retry_limit
       # Use multiple threads for processing.
       num_threads 2
    </match>
metadata:
  name: fluentd-es-config
  namespace: logging
---
# Kibana Service
apiVersion: v1
kind: Service
metadata:
  name: kibana
  namespace: logging
  labels:
    app: kibana
spec:
  ports:
  - port: 5601
    protocol: TCP
    targetPort: ui
  selector:
    app: kibana
---
# Kibana Deployment
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: kibana
  namespace: logging
  labels:
    app: kibana
  annotations:
    sidecar.istio.io/inject: "false"
spec:
  template:
    metadata:
      labels:
        app: kibana
    spec:
      containers:
      - name: kibana
        image: docker.elastic.co/kibana/kibana-oss:6.1.1
        resources:
          # need more cpu upon initialization, therefore burstable class
          limits:
            cpu: 1000m
          requests:
            cpu: 100m
        env:
          - name: ELASTICSEARCH_URL
            value: http://elasticsearch:9200
        ports:
        - containerPort: 5601
          name: ui
          protocol: TCP
---
```

创建资源:

```bash
kubectl apply -f logging-stack.yaml
```

你应该看到以下内容:

```
namespace "logging" created
service "elasticsearch" created
deployment "elasticsearch" created
service "fluentd-es" created
deployment "fluentd-es" created
configmap "fluentd-es-config" created
service "kibana" created
deployment "kibana" created
```

## 配置 Istio

现在有一个正在运行的 Fluentd 守护进程，请使用新的日志类型配置 Istio，
并将这些日志发送到监听守护进程。
创建一个新的 YAML 文件来保存日志流的配置，Istio 将自动生成并收集。

将下面的内容保存为 `fluentd-istio.yaml`:

```
# Configuration for logentry instances
apiVersion: "config.istio.io/v1alpha2"
kind: logentry
metadata:
  name: newlog
  namespace: istio-system
spec:
  severity: '"info"'
  timestamp: request.time
  variables:
    source: source.labels["app"] | source.service | "unknown"
    user: source.user | "unknown"
    destination: destination.labels["app"] | destination.service | "unknown"
    responseCode: response.code | 0
    responseSize: response.size | 0
    latency: response.duration | "0ms"
  monitored_resource_type: '"UNSPECIFIED"'
---
# Configuration for a fluentd handler
apiVersion: "config.istio.io/v1alpha2"
kind: fluentd
metadata:
  name: handler
  namespace: istio-system
spec:
  address: "fluentd-es.logging:24224"
---
# Rule to send logentry instances to the fluentd handler
apiVersion: "config.istio.io/v1alpha2"
kind: rule
metadata:
  name: newlogtofluentd
  namespace: istio-system
spec:
  match: "true" # match for all requests
  actions:
   - handler: handler.fluentd
     instances:
     - newlog.logentry
---
```

创建资源:

```bash
istioctl create -f fluentd-istio.yaml
```

预期的输出类似于:
```
Created config logentry/istio-system/newlog at revision 22374
Created config fluentd/istio-system/handler at revision 22375
Created config rule/istio-system/newlogtofluentd at revision 22376
```

请注意在处理程序配置中 `address: "fluentd-es.logging:24224"` 行指向我们设置的Fluentd守护进程示例栈。

## 查看新的日志

1. 将流量发送到示例应用程序。

   对于 [BookInfo](../../guides/bookinfo.md#determining-the-ingress-ip-and-port)
   示例, 在浏览器中访问 `http://$GATEWAY_URL/productpage`
   或发送以下命令:

   ```bash
   curl http://$GATEWAY_URL/productpage
   ```

1. 在 Kubernetes 环境中, 通过以下命令为 Kibana 建立端口转发:

   ```bash
   kubectl -n logging port-forward $(kubectl -n logging get pod -l app=kibana -o jsonpath='{.items[0].metadata.name}') 5601:5601
   ```

    退出运行命令。完成访问 Kibana UI 时输入 Ctrl-C 退出。

1. 导航到 [Kibana UI](http://localhost:5601/) 并点击 右上角的 "Set up index patterns"。

1. 使用 `*` 作为索引模式, 并单击 "Next step."。

1. 选择 `@timestamp` 作为时间筛选字段名称，然后单击 "Create index pattern"。

1. 现在在左侧的菜单上点击 "Discover"，并开始检索生成的日志。

## 清理

* 删除新的遥测配置:

  ```bash
  istioctl delete -f fluentd-istio.yaml
  ```

* 删除 Fluentd, Elasticsearch, Kibana 示例栈:

  ```bash
  kubectl delete -f logging-stack.yaml
  ```

* 如果您不打算探索任何后续任务, 参考
  [BookInfo cleanup](../../guides/bookinfo.md#cleanup) 关闭应用程序的说明。

## 进一步阅读

* [收集指标和日志](../../tasks/telemetry/metrics-logs.md) 有关日志配置的详细说明。

* 学习更多关于 [Mixer](../../concepts/policy-and-control/mixer.md)
  和 [Mixer 配置](../../concepts/policy-and-control/mixer-config.md)。

* 查看完整 [属性词汇](../../reference/config/mixer/attribute-vocabulary.md)。

* 阅读参考指南 [编写配置](../../reference/writing-config.md)。