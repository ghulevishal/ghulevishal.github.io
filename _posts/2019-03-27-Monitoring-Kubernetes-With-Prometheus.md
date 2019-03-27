---
title: Monitoring Kubernetes with Prometheus
date: 2019-03-27 00:00:00
layout: post
tags:
  - Prometheus
---





# Monitoring with Prometheus

## Slides
<iframe src="https://docs.google.com/presentation/d/e/2PACX-1vTeh-3aSDikw7SdOMBKuyD0y4rmLfIrMHXRc_qs3fUYfv601wkC1_IFcuP2-1pfIgjRYfnPJQqNb6Tg/embed?start=false&loop=false&delayms=3000" frameborder="0" width="960" height="569" allowfullscreen="true" mozallowfullscreen="true" webkitallowfullscreen="true"></iframe>



### History and Introduction.

- Google introduced the Borg system. The Borg system is “a cluster manager that runs hundreds of thousands of jobs, from many thousands of different applications, across a number of clusters each with up to tens of thousands of machines.
- Google also developed the monitoring system for the Google Borg which is called as Borgmon. Borgmon is a real-time–focused time series monitoring system that uses that data to identify issues and alert on them. 
- Prometheus is inspired by the Google's Borgmon. When `Matt T. Proud` ex-Google SRE  joins the SoundCloud company, He developed the Prometheus. 
- Similar to the Borgmon, Prometheus was designed to provide near real-time introspection monitoring of cloud infrastructure and container-based microservices, services, and applications.
- Prometheus is written in Go, open source, and licensed under the Apache 2.0 license. It is incubated under the Cloud Native Computing Foundation
- Prometheus provides the powerful query language which extract the required information by processing the recent data.
 
### Architecture and working of Prometheus.

- Prometheus generally work with technique of scraping or pulling the data from the applications. The time-series data of application is often exposed as HTTP endpoint via exporter( client-libraries or proxies).
- Sometime time-series data cannot be pulled if the application or targets in such a condition Prometheus may have the `Push gateway` to receive small volumes of data.
-  An endpoint usually corresponds to a single process, host, service, or application. To scrape data from an endpoint, Prometheus defines configuration(how to connect endpoint, authentications required to connect endpoints etc) called a target. 
- Groups of targets with the same role are called as Job.
- Time series data is stored locally on the Prometheus server or it can be shipped to external storage.
- Prometheus queries the time-series data and it will also aggregate time-series data. With aggregation it can create new time-series data from existing time-series data.
- Prometheus doesn't come with inbuilt alerting system, you have to configure external alertmanager server for sending you the alert about monitoring.
- PromQL is prometheus in built query language which process time-series data and extract out the required information.
- For faster querying, we should use faster data storage(SSDs), so data can processed faster.
- We can run prometheus server as HA mode.
-  Prometheus collects time series data. To handle this data it has a multi-dimensional time series data model. The time series data model combines time series names and key/value pairs called labels; these labels provide the dimensions. Each time series is uniquely identified by the combination of time series name and any assigned labels.


![](https://prometheus.io/assets/architecture.png)

 


## Prerequisites : Helm must be installed and tiller pod must be running in your kubernetes cluster.


### Install Helm binaries.

- Download Helm installation script. Change the permission of the script and execute the script using following commands.

```command
curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get > get_helm.sh
chmod 700 get_helm.sh
./get_helm.sh
```

- Lets create Service account `tiller` and RBAC rule for `tiller` service account.

```command
cat rbac_helm.yaml
```
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system

  - kind: User
    name: "admin"
    apiGroup: rbac.authorization.k8s.io

  - kind: User
    name: "kubelet"
    apiGroup: rbac.authorization.k8s.io

  - kind: Group
    name: system:serviceaccounts
    apiGroup: rbac.authorization.k8s.io

```

- Deploy above configuration.

```command
kubectl apply -f rbac_helm.yaml 
```

- Initialize the Helm.

```command
helm init --service-account tiller 
```

- Verify tiller pod is running in the `kube-system` namespace.

```command
kubectl --namespace kube-system get pods | grep tiller
```


## Demo.

- Install the Prometheus with Helm chart.

```command
helm install --name prometheus --set server.service.type=NodePort stable/prometheus
```

- Get the list of the Pods.

```command
kubectl get pods
```
```output
NAME                                             READY   STATUS    RESTARTS   AGE
prometheus-alertmanager-7c8d6b6754-mgtr2         2/2     Running   0          3m
prometheus-kube-state-metrics-74d5c694c7-j88h6   1/1     Running   0          3m
prometheus-node-exporter-rl5jj                   1/1     Running   0          3m
prometheus-pushgateway-d5fdc4f5b-7pzv6           1/1     Running   0          3m
prometheus-server-c946c7f8-b5fmt                 2/2     Running   0          3m
```

- Get the list of the services.

```command
kubectl get svc
```
```output
NAME                            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes                      ClusterIP   10.96.0.1       <none>        443/TCP        1h
prometheus-alertmanager         ClusterIP   10.98.205.151   <none>        80/TCP         18s
prometheus-kube-state-metrics   ClusterIP   None            <none>        80/TCP         18s
prometheus-node-exporter        ClusterIP   None            <none>        9100/TCP       18s
prometheus-pushgateway          ClusterIP   10.110.53.233   <none>        9091/TCP       17s
prometheus-server               NodePort    10.96.194.14    <none>        80:30469/TCP   17s
```

Now try to access the Prometheus UI by using NodePort shown above. In Prometheus UI if go to the `Status`-> `Rule`, You see `No rules defined`. Similarly, in Prometheus UI if go to `Alert` you can see `No alerting rules defined`.

Let's configure `Record Rules` and `Alert Rules`.

- Get the  [values.yaml](https://raw.githubusercontent.com/helm/charts/master/stable/prometheus/values.yaml) and modifiy as below.

- In `values.yaml` find the section of `alertmanagerFiles` and update as below.

```yaml
.
.
.
.
## alertmanager ConfigMap entries
##
alertmanagerFiles:
  alertmanager.yml:
    global: {}
    global:
      smtp_smarthost: 'smtp.gmail.com:587'
      smtp_from: 'sender@cloudyuga.guru'
    templates:
    - '/etc/alertmanager/template/*.tmpl'
    route:
      receiver: email
    receivers:
    - name: 'email'
      email_configs:
      - to: 'reciever@cloudyuga.guru'
        from: "sender@cloudyuga.guru"
        smarthost: smtp.gmail.com:587
        auth_username: "sender@cloudyuga.guru"
        auth_identity: "sender@cloudyuga.guru"
        auth_password: "************"
.
.
.
.
.
.
```

Update SMTP Host, the Sender and reciever Email-ID. Update authentication for sender. 

- Configure the `Alert Rule` in `values.yaml`, Find a `serverFiles:` section and update with your alert rules in `alerts` section. For example take a look at the following configuration

```yaml
.
.
.
.
## Prometheus server ConfigMap entries
##
serverFiles:

  ## Alerts configuration
  ## Ref: https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/
  alerts: 
    groups:
      - name: k8s_alerts
        rules:
        - alert: MoreThan30Deployments
          expr: count(kube_deployment_created) >= 30
          for: 2m
          labels:
            severity: critical
          annotations:
            summary: Hey Admin!!!!! More Than 30 Deployments are running in cluster
      - name: node_alerts
        rules:
        - alert: HighNodeCPU
          expr: instance:node_cpu:avg_rate5m > 20
          for: 10s
          labels:
            severity: warning
          annotations:
            summary: High Node CPU of {{ humanize $value}}% for 1 hour
.
.
.
.
.
```

- Configure the `Record Rule` in `values.yaml`, Find a `serverFiles:` section and update with your record rules in `rules` section. For example it will look like the following configuration

```
.
.
.
serverFiles:

  ## Alerts configuration
  ## Ref: https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/
  alerts: 
    groups:
      - name: k8s_alerts
        rules:
        - alert: MoreThan30Deployments
          expr: count(kube_deployment_created) >= 30
          for: 2m
          labels:
            severity: critical
          annotations:
            summary: Hey Admin!!!!! More Than 30 Deployments are running in cluster
      - name: node_alerts
        rules:
        - alert: HighNodeCPU
          expr: instance:node_cpu:avg_rate5m > 20
          for: 10s
          labels:
            severity: warning
          annotations:
            summary: High Node CPU of {{ humanize $value}}% for 1 hour

  rules: 
    groups:
      - name: kubernetes_rules
        rules:
        - record: apiserver_latency_seconds:quantile
          expr: histogram_quantile(0.99, rate(apiserver_request_latencies_bucket[5m])) / 1e+06
          labels:
            quantile: "0.99"
        - record: apiserver_latency_seconds:quantile
          expr: histogram_quantile(0.9, rate(apiserver_request_latencies_bucket[5m])) / 1e+06
          labels:
            quantile: "0.9"
        - record: apiserver_latency_seconds:quantile
          expr: histogram_quantile(0.5, rate(apiserver_request_latencies_bucket[5m])) / 1e+06
          labels:
            quantile: "0.5"
      - name: node_rules
        rules:
        - record: instance:node_cpu:avg_rate5m
          expr: 100 - avg (irate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance) * 100
        - record: instance:node_memory_usage:percentage
          expr: (node_memory_MemTotal_bytes - (node_memory_MemFree + node_memory_Cached_bytes + node_memory_Buffers_bytes)) / node_memory_MemTotal_bytes * 100
        - record: instance:root:node_filesystem_usage:percentage
          expr: (node_filesystem_size_bytes{mountpoint="/"} - node_filesystem_free_bytes{mountpoint="/"}) / node_filesystem_size_bytes{mountpoint="/"}* 100
      - name: k8s_rules
        rules:
        - record: k8s:service:count
          expr: count(kube_service_created)
        - record: k8s:pod:count
          expr: count(kube_pod_created)
        - record: k8s:deploy:count
          expr: count(kube_deployment_created)
        - record: k8s:clusterrole:count
          expr: etcd_object_counts{job="kubernetes-apiservers",resource="clusterroles.rbac.authorization.k8s.io"}
        - record: k8s:serviceaccount:count
          expr: etcd_object_counts{job="kubernetes-apiservers",resource="serviceaccounts"}	

  prometheus.yml:
    rule_files:
      - /etc/config/rules
      - /etc/config/alerts

    scrape_configs:
      - job_name: prometheus
        static_configs:
          - targets:
            - localhost:9090
.
.
.
.
        
```

Once you update the `values.yaml` Lets now upgrade the Helm chart.

- Upgrade the Helm release.

```command
helm upgrade prometheus --set server.service.type=NodePort stable/prometheus -f configs/values.yaml
```

Once you upgrade the Helm release. Access the Prometheus UI. In Prometheus UI if go to the `Status`-> `Rule`, You see new rules we have configure earlier are present there. Similarly, in Prometheus UI if you go to `Alert`, you can see that Alert rules are present there.

## Clean UP.

- Remove the Helm release and all its components.

```
helm del --purge prometheus
```

