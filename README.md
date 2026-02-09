# k8s monitoring basics

## Goal

Setup simple monitoring with `prometheus` and `grafana` for a small cluster.

## Prerequisites

A `k8s` cluster and `helm` installed and configured (a KUBECONFIG environment variable pointing to the kubeconfig of the cluster should be enough).

## What we have

For the beginning, let's add a testing application in our cluster. The deployment could look like:
```sh
apiVersion: apps/v1
kind: Deployment
metadata:
  name: testapp
  namespace: testapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: testapp
  template:
    metadata:
      labels:
        app: testapp
    spec:
      containers:
        - name: testapp
          image: hashicorp/http-echo:latest
          args:
            - "-text=Test App is Working!"
          ports:
            - name: http
              containerPort: 5678
```
, the service could look like:
```sh
apiVersion: v1
kind: Service
metadata:
  name: testapp-service
  namespace: testapp
  labels:
    app: testapp
spec:
  type: LoadBalancer
  selector:
    app: testapp
  ports:
    - name: http
      port: 80
      targetPort: 5678
```
, and the namespace would be:
```sh
apiVersion: v1
kind: Namespace
metadata:
  name: testapp
```
Note that you will need to label your service, so that `prometheus` can find it for scraping.

If they are all found in a directory named *testapp*, we can add all of them to the cluster with the follwing command:
```sh
kubectl apply -f testapp
```

## Adding prometheus and grafana

Let's add the `helm` repositories:
```sh
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

Now we can create a monitoring namespace yaml file:
```sh
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
```

Let's create the values file for the `prometheus` and `grafana` `helm` chart:
```sh
grafana:
  enabled: true
  adminPassword: "admin123"
  service:
    type: NodePort
    nodePort: 30000
  
prometheus:
  enabled: true
  service:
    type: NodePort
    nodePort: 30090
  prometheusSpec:
    serviceMonitorSelector: {}
    podMonitorSelector: {}
    serviceMonitorSelectorNilUsesHelmValues: false
    podMonitorSelectorNilUsesHelmValues: false
  
kube-state-metrics:
  enabled: true
  
prometheus-node-exporter:
  enabled: true
```
Most of the settings are self-explanatory, except the last *prometheusSpec* batch, which are configured to allow scraping of all namespaces and all service and pod monitors. You might want to change this in some way depending to security needs. I assume you will change the password later :).

Now we can add the namespace to the cluster:
```sh
kubectl apply -f ./monitoring/namespace.yaml
```

The next step is to install the `helm` chart. But first we want to know what version to use. Let's call:
```sh
helm search repo prometheus-community/kube-prometheus-stack --versions
```
to find out what versions we can use. This will output something like:
```sh
NAME                                      	CHART VERSION	APP VERSION	DESCRIPTION
prometheus-community/kube-prometheus-stack	81.5.0       	v0.88.1    	kube-prometheus-stack collects Kubernetes manif...
prometheus-community/kube-prometheus-stack	81.4.3       	v0.88.1    	kube-prometheus-stack collects Kubernetes manif...
prometheus-community/kube-prometheus-stack	81.4.2       	v0.88.1    	kube-prometheus-stack collects Kubernetes manif...
prometheus-community/kube-prometheus-stack	81.4.1       	v0.88.1    	kube-prometheus-stack collects Kubernetes manif...
prometheus-community/kube-prometheus-stack	81.4.0       	v0.88.1    	kube-prometheus-stack collects Kubernetes manif...
...
```

Now we can install the `helm` chart:
```sh
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --version 81.4.3 \
  --values monitoring/values.yaml \
  --wait
```

You can uninstall this with:
```sh
helm uninstall prometheus -n monitoring
```
if you want to clean it up later.

## Cluster monitor

First of all, to check if service discovery is working, we can call:
```sh
kubectl exec -n monitoring statefulset/prometheus-prometheus-kube-prometheus-prometheus \
  -- wget -q -O- "http://localhost:9090/api/v1/targets" \
  | jq '.data.activeTargets[] | {job: .labels.job, health: .health}' | head -12
```
, which should return something like:
```sh
{
  "job": "prometheus-grafana",
  "health": "up"
}
{
  "job": "prometheus-kube-prometheus-alertmanager",
  "health": "up"
}
{
  "job": "prometheus-kube-prometheus-alertmanager",
  "health": "up"
}
```

To check if `grafana` is accessible, we can call:
```sh
kubectl exec -n monitoring deployment/prometheus-grafana -- wget -q -O- --timeout=2 http://localhost:3000/api/health
```
, which should return something like:
```sh
{
  "database": "ok",
  "version": "12.3.1",
  "commit": "3a1c80ca7ce612f309fdc99338dd3c5e486339be"
}
```

To check if `prometheus` is scraping *kubelet*, we can can the following command:
```sh
kubectl exec -n monitoring statefulset/prometheus-prometheus-kube-prometheus-prometheus \
  -- wget -q -O- "http://localhost:9090/api/v1/query?query=up{job=\"kubelet\"}" \
  | jq '.data.result[] | {job: .metric.job, instance: .metric.instance, up: .value[1]}'
```
, which should show something like:
```sh
...
  "job": "kubelet",
  "instance": "172.19.0.2:10250",
  "up": "1"
...
```

To check if some node metrics are being collected, we can call:
```sh
kubectl exec -n monitoring statefulset/prometheus-prometheus-kube-prometheus-prometheus \
  -- wget -q -O- "http://localhost:9090/api/v1/query?query=node_cpu_seconds_total" | jq '.data.result[0:1]'
```
, for example; this will query *node_cpu_seconds_total* metrics, so we expect something like:
```sh
...
    "metric": {
      "__name__": "node_cpu_seconds_total",
      "container": "node-exporter",
      "cpu": "0",
      "endpoint": "http-metrics",
...
```

To check metrics for our *testapp* container, we can call:
```sh
kubectl exec -n monitoring statefulset/prometheus-prometheus-kube-prometheus-prometheus \
  -- wget -q -O- "http://localhost:9090/api/v1/query?query=container_memory_usage_bytes{namespace=\"testapp\",container=\"testapp\"}" \
  | jq '.data.result[]'
```
Note that this queries for *container_memory_usage_bytes*, but as long as it returns something, it means that it is working. The result should be something like:
```sh
...
  "metric": {
    "__name__": "container_memory_usage_bytes",
    "container": "testapp",
    "endpoint": "https-metrics",
...
```

## Monitor our current testing application

To monitor our *testapp* we need to create a *ServiceMonitor* `k8s` resource for it:
```sh
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: testapp-monitor
  namespace: testapp
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: testapp
  endpoints:
  - targetPort: 5678
    interval: 15s
    path: /
```
, and apply it:
```sh
kubectl apply -f ./monitoring/testapp-monitor.yaml
```

This will not work in the end, because our *testapp* does not have a monitoring endpoint, and the path we are trying to monitor returns *Test App is Working!*, which will no be able to be interpreted by `prometheus`.

By calling:
```sh
kubectl exec -n monitoring statefulset/prometheus-prometheus-kube-prometheus-prometheus -- wget -q -O- "http://localhost:9090/api/v1/targets" | jq '.data.activeTargets[] | select(.labels.job | contains("testapp"))'
```
, we enter into a `prometheus` pod and list scraping results. This will show something like:
```sh
{
...
  "labels": {
    "container": "testapp",
    "endpoint": "5678",
    "instance": "10.244.0.65:5678",
    "job": "testapp-service",
    "namespace": "testapp",
    "pod": "testapp-f7679c884-c5zbn",
    "service": "testapp-service"
  },
...
  "lastError": "unsupported character in float while parsing: \"Test App\"",
  "lastScrape": "2026-02-07T18:09:33.666063882Z",
  "lastScrapeDuration": 0.001640375,
  "health": "down",
  "scrapeInterval": "15s",
  "scrapeTimeout": "10s"
}
```
I removed some content to keep the output smaller, but we are interested int he following: *"job": "testapp-service"*, *"lastScrape": "2026-02-07T18:09:33.666063882Z"*, *"lastScrapeDuration": 0.001640375* and finally the error *"lastError": "unsupported character in float while parsing: \"Test App\""*, which was expected as I explained previously.

## Monitor a meaningful testing application

To properly play around with `prometheus`, let's add a second testing application that actually returns metrics. We can do so by using another image. Let's add the deployment:
```sh
apiVersion: apps/v1
kind: Deployment
metadata:
  name: testapp-metrics
  namespace: testapp-metrics
spec:
  replicas: 1
  selector:
    matchLabels:
      app: testapp-metrics
  template:
    metadata:
      labels:
        app: testapp-metrics
    spec:
      containers:
      - name: app
        image: quay.io/brancz/prometheus-example-app:v0.3.0
        ports:
        - name: http
          containerPort: 8080
```
, a service for it:
```sh
apiVersion: v1
kind: Service
metadata:
  name: testapp-metrics-service
  namespace: testapp-metrics
  labels:
    app: testapp-metrics
spec:
  type: LoadBalancer
  selector:
    app: testapp-metrics
  ports:
    - name: http
      port: 80
      targetPort: 8080
```
, and a namespace:
```sh
apiVersion: v1
kind: Namespace
metadata:
  name: testapp-metrics
```
We can apply all of these by calling:
```sh
kubectl apply -f testapp-metrics
```

To find out how the metrics look like, we can port forward the service:
```sh
kubectl port-forward -n testapp-metrics svc/testapp-metrics-service 8081:80
```
, and view the results in the browser at *http://localhost:8081/metrics*:
```sh
# HELP version Version information about this binary
# TYPE version gauge
version{version="v0.3.0"} 1
```
, so we see a metric about the version.

Let's create a new  *ServiceMonitor* `k8s` resource for our new testing application:
```sh
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: testapp-metrics-monitor
  namespace: testapp-metrics
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: testapp-metrics
  endpoints:
  - port: http
    interval: 15s
    path: /metrics
```
, and apply it:
```sh
kubectl apply -f ./monitoring/testapp-metrics-monitor.yaml
```

Now that we know that our new application has a metric called version, we can query it in `prometheus` by calling:
```sh
kubectl exec -n monitoring statefulset/prometheus-prometheus-kube-prometheus-prometheus \
  -- wget -q -O- "http://localhost:9090/api/v1/query?query=version" | jq .
```
, which should return something like:
```sh
  "status": "success",
  "data": {
    "resultType": "vector",
    "result": [
      {
        "metric": {
          "__name__": "version",
          "container": "app",
          "endpoint": "http",
          "instance": "10.244.0.83:8080",
          "job": "testapp-service",
          "namespace": "testapp-metrics",
          "pod": "testapp-metrics-5ccbfb5f46-xnkw4",
          "service": "testapp-service",
          "version": "v0.3.0"
        },
        "value": [
          1770655231.334,
          "1"
```
Note that we recognize a *version* metric with a value of 1.

Let's do another test to find out what metrics prometheus finds in the *testapp-metrics* namespace:
```sh
kubectl exec -n monitoring statefulset/prometheus-prometheus-kube-prometheus-prometheus \
  -- wget -q -O- "http://localhost:9090/api/v1/query?query=up{namespace=\"testapp-metrics\"}" | jq .
```
We expect something similar as before. The output should look like:
```sh
  "status": "success",
  "data": {
    "resultType": "vector",
    "result": [
      {
        "metric": {
          "__name__": "up",
          "container": "app",
          "endpoint": "http",
          "instance": "10.244.0.83:8080",
          "job": "testapp-service",
          "namespace": "testapp-metrics",
          "pod": "testapp-metrics-5ccbfb5f46-xnkw4",
          "service": "testapp-service"
        },
        "value": [
          1770655377.710,
          "1"
```

## Grafana

To connect to `grafana`, we must first configure a port forward:
```sh
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80
```
Now we can access `grafana` in our browser at *http://localhost:3000*. The login data is *admin:admin123*.

In the left panel, we can navigate to *Dashboards* to view any of the default dashboards:
```sh
Alertmanager / Overview
CoreDNS
etcd
Grafana Overview
Kubernetes / API server
Kubernetes / Compute Resources / Multi-Cluster
Kubernetes / Compute Resources / Cluster
Kubernetes / Compute Resources / Namespace (Pods)
Kubernetes / Compute Resources / Namespace (Workloads)
Kubernetes / Compute Resources / Node (Pods)
Kubernetes / Compute Resources / Pod
Kubernetes / Compute Resources / Workload
Kubernetes / Controller Manager
Kubernetes / Kubelet
Kubernetes / Networking / Cluster
Kubernetes / Networking / Namespace
Kubernetes / Networking / Namespace (Workload)
Kubernetes / Networking / Pod
Kubernetes / Networking / Workload
Kubernetes / Persistent Volumes
Kubernetes / Proxy
Kubernetes / Scheduler
Node Exporter / AIX
Node Exporter / Nodes
Node Exporter / USE Method / Cluster
Node Exporter / USE Method / Node
Prometheus / Overview
```
Feel free to explore any of them.
While you can create new dashboards via the UI, I recommend to create them as yaml files and apply them via `kubectl`. Let's create the yaml file first:
```sh
apiVersion: v1
kind: ConfigMap
metadata:
  name: testapp-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "true"
data:
  testapp-dashboard.json: |
    {
      "title": "TestApp-Metrics Dashboard",
      "panels": [
        {
          "title": "CPU Usage",
          "type": "graph",
          "targets": [{
            "expr": "rate(container_cpu_usage_seconds_total{namespace=\"testapp-metrics\",container=\"app\"}[5m])",
            "legendFormat": "{{pod}}"
          }]
        },
        {
          "title": "Memory Usage",
          "type": "graph", 
          "targets": [{
            "expr": "container_memory_usage_bytes{namespace=\"testapp-metrics\",container=\"app\"}",
            "legendFormat": "{{pod}}"
          }]
        },
        {
          "title": "Version",
          "type": "stat",
          "targets": [{
            "expr": "version{namespace=\"testapp-metrics\"}",
            "legendFormat": "v{{version}}"
          }]
        }
      ]
    }
```
In this code snippet we want to create a dashboard named *TestApp-Metrics Dashboard* with 3 panels: *CPU Usage*, *Memory Usage* and *Version*. The *Version* panel monitors our custom version parameter. We created a *ConfigMap* `k8s` resource in the *monitoring* namespace with a *grafana_dashboard* label. A `grafana` sidecar container watches for these *ConfigMaps*, look for the json and copy it in the `grafana` dashboard directory. Afterwards the dashboard is loaded automatically and will appear in `grafana` UI.

Let me prove that there is a `grafana` sidecar. First of all, let's call:
```sh
kubectl get pods -n monitoring -l app.kubernetes.io/name=grafana -o jsonpath='{.items[*].spec.containers[*].name}'
```
, which should output something like:
```sh
grafana-sc-dashboard grafana-sc-datasources grafana
```
By calling:
```sh
kubectl get pods -n monitoring -l app.kubernetes.io/name=grafana \
  -o jsonpath='{range .items[0].spec.containers[*]}{.name}{" -> "}{.ports[*].containerPort}{"\n"}{end}'
```
, we get something like:
```sh
grafana-sc-dashboard ->
grafana-sc-datasources ->
grafana -> 3000 9094 9094 6060
```
, which shows us clearly that `grafana` is the main container and the other 2 are sidecar containers.

Now let me prove that our dashboard was copied in some *dashboards* directory. The correct container is obviously *grafana-sc-dashboard*. Let's run:
```sh
kubectl exec -n monitoring -it $(kubectl get pod -n monitoring -l app.kubernetes.io/name=grafana -o name) -c grafana-sc-dashboard -- ls -la /tmp/dashboards/
```
, which will output something like:
```sh
...
-rw-r--r--    1 472      472           9724 Feb  7 18:24 scheduler.json
-rw-r--r--    1 472      472            715 Feb  9 17:19 testapp-dashboard.json
-rw-r--r--    1 472      472          11018 Feb  7 18:24 workload-total.json
```
, where we clearly see our *testapp-dashboard.json*.

Let's apply the dashboard:
```sh
kubectl apply -f ./monitoring/testapp-metrics-dashboard.yaml
```
You can now find the newly created *TestApp-Metrics Dashboard* in `grafana` UI.

## Cleanup

To clean everything up, we can call
```sh
kubectl delete -f ./testapp-metrics
kubectl delete -f ./testapp
helm uninstall prometheus -n monitoring
kubectl delete ns monitoring
```
