---
title: Detecting Anomalous Activities in Your Kubernetes Applications Using Falco
date: 2019-05-2 14:26:00
layout: post
tags:
  - Falco
  - Kubernetes
---
## Falco
Falco is the behavioral activity monitor designed to detect anomalous activity in your applications. Falco is powered by sysdig’s system call capture methodology, it lets you continuously monitor containers, applications, hosts, and network activities all in single place, from single source of data, with single set of rules.Falco is also emerged as as a sandbox level project at CNCF's Umbrella. 

### Deploying Falco to Kubernetes with RBAC enabled.

- First clone the git repository and get into the proper directory.

```command
git clone https://github.com/falcosecurity/falco.git
cd falco/integrations/k8s-using-daemonset/
```

-  Create a Service Account for Falco, as well as the Cluster Role Bindings to grant the appropriate permissions to the `falco-account` Service Account. 

```command
cat <<EOF> k8s-with-rbac/falco-account.yaml

apiVersion: v1
kind: ServiceAccount
metadata:
  name: falco-account
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: falco-cluster-role-binding
  namespace: default
subjects:
  - kind: ServiceAccount
    name: falco-account
    namespace: default
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
  
EOF
```

- Apply above configuration which will create Service Account and Cluster Role Binding for it.

```command
kubectl create -f k8s-with-rbac/falco-account.yaml
```

- In order to create the ConfigMap you'll need to first need to copy the required configuration from their location in this GitHub repo to the `k8s-with-rbac/falco-config/` 

```command
mkdir -p k8s-with-rbac/falco-config/
cp ../../falco.yaml k8s-with-rbac/falco-config/
cp ../../rules/falco_rules.* k8s-with-rbac/falco-config/
```
The `falco.yaml` file contains configuration of falco. While `falco_rules.yaml` contains rules and alerts for falco. Any custom rules for your environment can be added to into the `falco_rules.local.yaml` file and they will be picked up by Falco at start time.

- If you want to send Falco alerts to a `Slack channel`, You have to modify the `k8s-with-rbac/falco-config/falco.yaml` file to point to your Slack webhook. Add the below to the bottom of the falco.yaml config file you just copied to enable Slack messages.

```yaml
program_output:
  enabled: true
  keep_alive: false
  program: "jq '{text: .output}' | curl -d @- -X POST https://hooks.slack.com/services/see_your_slack_team/apps_settings_for/a_webhook_url"
```
You will also need to enable JSON output. Find the `json_output: false` setting in the `falco.yaml` file and change it to  `json_output: true`.  

- Create the ConfigMap of these rules files located at `k8s-with-rbac/falco-config/`.

```command
kubectl create configmap falco-config --from-file=k8s-with-rbac/falco-config
```

- Deploy Falco Daemonset. So on each node one pod will be scheduled and it will keep watching for the suspicious event on each node.

```command
kubectl create -f k8s-without-rbac/falco-daemonset.yaml
```

### Verifying the installation.

- List the pods.

```command
kubectl get pods
```
```
NAME          READY   STATUS    RESTARTS   AGE
falco-gnr52   1/1     Running   0          21s
```

- Get the logs of the pods.

```command
kubectl logs falco-gnr52
```
```
depmod...

DKMS: install completed.
* Trying to load a dkms falco-probe, if present
falco-probe found and loaded in dkms
Wed Jan  9 08:43:52 2019: Falco initialized with configuration file /etc/falco/falco.yaml
Wed Jan  9 08:43:52 2019: Loading rules from file /etc/falco/falco_rules.yaml:
Wed Jan  9 08:43:52 2019: Loading rules from file /etc/falco/falco_rules.local.yaml:
Wed Jan  9 08:43:52 2019: Starting internal webserver, listening on port 8765
```

- Exec into the pod and come out of it.

```command
kubectl exec falco-dnxsm bash
```

- Check the logs of the pod.

```command
kubectl logs falco-dnxsm
```
```
* Setting up /usr/src links from host
* Unloading falco-probe, if present
* Running dkms install for falco

Kernel preparation unnecessary for this kernel.  Skipping...

Building module:
cleaning build area...
make -j2 KERNELRELEASE=4.4.0-141-generic -C /lib/modules/4.4.0-141-generic/build M=/var/lib/dkms/falco/0.13.0/build...
cleaning build area...

DKMS: build completed.

falco-probe.ko:
Running module version sanity check.
 - Original module
   - No original module exists within this kernel
 - Installation
   - Installing to /lib/modules/4.4.0-141-generic/kernel/extra/
mkdir: cannot create directory '/lib/modules/4.4.0-141-generic/kernel/extra': Read-only file system
cp: cannot create regular file '/lib/modules/4.4.0-141-generic/kernel/extra/falco-probe.ko': No such file or directory

depmod...

DKMS: install completed.
* Trying to load a dkms falco-probe, if present
falco-probe found and loaded in dkms
Wed Jan  9 08:57:28 2019: Falco initialized with configuration file /etc/falco/falco.yaml
Wed Jan  9 08:57:28 2019: Loading rules from file /etc/falco/falco_rules.yaml:
Wed Jan  9 08:57:28 2019: Loading rules from file /etc/falco/falco_rules.local.yaml:
Wed Jan  9 08:57:29 2019: Starting internal webserver, listening on port 8765
{"output":"08:58:36.335246440: Notice A shell was spawned in a container with an attached terminal (user=root k8s.pod=falco-dnxsm container=bb1a36fcb1d5 shell=bash parent=<NA> cmdline=bash  terminal=34816)","priority":"Notice","rule":"Terminal shell in container","time":"2019-01-09T08:58:36.335246440Z", "output_fields": {"container.id":"bb1a36fcb1d5","evt.time":1547024316335246440,"k8s.pod.name":"falco-dnxsm","proc.cmdline":"bash ","proc.name":"bash","proc.pname":null,"proc.tty":34816,"user.name":"root"}}
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   210    0     2  100   208      3    365 --:--:-- --:--:-- --:--:--   369
```

You will see that you are also getting messages on teh slack channels too.


### Clean Up.

```command
kubectl delete cm falco-config
kubectl delete ds falco
kubectl delete  sa falco-account
kubectl delete clusterrolebinding falco-cluster-role-binding
```
