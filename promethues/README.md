### 参考文档

```
http://www.servicemesher.com/blog/prometheus-operator-manual/
https://github.com/coreos/prometheus-operator
https://github.com/coreos/kube-prometheus
```

### 背景环境

- kubernetes集群1.16版本，纯二进制版本打造，参考[k8s1.16集群部署]([https://github.com/anzejiang/kubernetes/edit/master/ansible%E8%87%AA%E5%8A%A8%E5%8C%96%E9%83%A8%E7%BD%B2/README.md](https://github.com/anzejiang/kubernetes/edit/master/ansible自动化部署/README.md))



### 监控原理

Prometheus读取Metrcs，读取etcd或者api的都行

查看etcd的metrics输出信息
```
[root@elasticsearch01 yaml]# curl --cacert /opt/etcd/ssl/ca.pem --cert /opt/etcd/ssl/server.pem --key /opt/etcd/ssl/server-key.pem https://192.168.122.215:2379/metrics
```
查看kube-apiserver的metrics信息
```
[root@elasticsearch01 yaml]#  kubectl get --raw /metrics
```

### 实施部署

注意：This will be the last release supporting Kubernetes 1.13 and before. The next release is going to support Kubernetes 1.14+ only.
后续版本只支持k8s1.14+，所以后续要下载release版本，目前只有一个版本所以可以直接git clone

**1.下载原代码**

```
[root@elasticsearch01 opt]# git clone https://github.com/anzejiang/kubernetes.git
```

**2.查看原配置文件**

```
[root@elasticsearch01 opt]# cd kubernetes/promethues/
[root@elasticsearch01 promethues]# ll
总用量 56
drwxr-xr-x 2 root root  4096 1月  13 16:03 adapter
drwxr-xr-x 2 root root  4096 1月  13 16:05 alertmanager
drwxr-xr-x 2 root root  4096 1月  13 18:38 grafana
drwxr-xr-x 2 root root  4096 1月  14 16:13 ingress-yaml
-rw-r--r-- 1 root root 17010 1月  14 16:22 kubernetes集群全栈监控报警方案kube-prometheus.md
drwxr-xr-x 2 root root  4096 1月  13 16:05 kube-state-metrics
drwxr-xr-x 2 root root  4096 1月  13 16:05 node-exporter
drwxr-xr-x 2 root root  4096 1月  13 16:03 operator
drwxr-xr-x 2 root root  4096 1月  13 16:05 prometheus
drwxr-xr-x 2 root root  4096 1月  13 16:05 serviceMonitor
```


**3.新建目录重新梳理下**

```
[root@elasticsearch01 promethues]# ls -lh
总用量 56K
drwxr-xr-x 2 root root 4.0K 1月  13 16:03 adapter
drwxr-xr-x 2 root root 4.0K 1月  13 16:05 alertmanager
drwxr-xr-x 2 root root 4.0K 1月  13 18:38 grafana
drwxr-xr-x 2 root root 4.0K 1月  14 16:13 ingress-yaml
-rw-r--r-- 1 root root  17K 1月  14 16:22 kubernetes集群全栈监控报警方案kube-prometheus.md
drwxr-xr-x 2 root root 4.0K 1月  13 16:05 kube-state-metrics
drwxr-xr-x 2 root root 4.0K 1月  13 16:05 node-exporter
drwxr-xr-x 2 root root 4.0K 1月  13 16:03 operator
drwxr-xr-x 2 root root 4.0K 1月  13 16:05 prometheus
drwxr-xr-x 2 root root 4.0K 1月  13 16:05 serviceMonitor

```

**4.部署前注意问题**
a.镜像问题
其中k8s.gcr.io/addon-resizer:1.8.4镜像下载不了，需要借助阿里云中转下，其他镜像默认都能下载，如遇到不能下载的也需要中转下再tag到自己私有镜像库
```
[root@VM_8_24_centos ~]# docker pull registry.cn-beijing.aliyuncs.com/minminmsn/addon-resizer:1.8.4
1.8.4: Pulling from minminmsn/addon-resizer
90e01955edcd: Pull complete 
ab19a0d489ff: Pull complete 
Digest: sha256:455eb18aa7a658db4f21c1f2b901c6a274afa7db4b73f4402a26fe9b3993c205
Status: Downloaded newer image for registry.cn-beijing.aliyuncs.com/minminmsn/addon-resizer:1.8.4

[root@VM_8_24_centos ~]# docker tag registry.cn-beijing.aliyuncs.com/minminmsn/addon-resizer:1.8.4 core-harbor.minminmsn.com/public/addon-resizer:1.8.4 
[root@VM_8_24_centos ~]# docker push core-harbor.minminmsn.com/public/addon-resizer:1.8.4 
The push refers to repository [core-harbor.minminmsn.com/public/addon-resizer]
cd05ae2f58b4: Pushed 
8a788232037e: Pushed 
1.8.4: digest: sha256:455eb18aa7a658db4f21c1f2b901c6a274afa7db4b73f4402a26fe9b3993c205 size: 738
```


b.访问问题
grafana，prometheus，alermanager等如果不想使用ingres方式访问就需要使用nodeport方式，否则对外不好访问
nodeport方式需在service配置文件，如grafana/grafana-service.yaml 添加type: NodePort,如果要指定node对外端口，需要加配nodePort: 33000，具体可以看配置文件
ingress方式也需要配置文件，ingress配置文件见最后访问配置文件，ingress部署参考[k8s集群部署ingress](https://github.com/minminmsn/k8s1.13/blob/master/ingress-nginx/kubernetes1.13.1%E9%83%A8%E7%BD%B2ingress-nginx%E5%B9%B6%E9%85%8D%E7%BD%AEhttps%E8%BD%AC%E5%8F%91dashboard.md)


**5.应用部署**
```
[root@elasticsearch01 promethues]# kubectl apply -f .
namespace/monitoring created

[root@elasticsearch01 promethues]# kubectl apply -f operator/
customresourcedefinition.apiextensions.k8s.io/alertmanagers.monitoring.coreos.com created
customresourcedefinition.apiextensions.k8s.io/prometheuses.monitoring.coreos.com created
customresourcedefinition.apiextensions.k8s.io/prometheusrules.monitoring.coreos.com created
customresourcedefinition.apiextensions.k8s.io/servicemonitors.monitoring.coreos.com created
clusterrole.rbac.authorization.k8s.io/prometheus-operator created
clusterrolebinding.rbac.authorization.k8s.io/prometheus-operator created
deployment.apps/prometheus-operator created
service/prometheus-operator created
serviceaccount/prometheus-operator created
[root@elasticsearch01 promethues]# kubectl -n monitoring get pod
NAME                                   READY   STATUS    RESTARTS   AGE
prometheus-operator-7cb68545c6-z2kjn   1/1     Running   0          41s


[root@elasticsearch01 promethues]# kubectl apply -f adapter/
apiservice.apiregistration.k8s.io/v1beta1.metrics.k8s.io created
clusterrole.rbac.authorization.k8s.io/prometheus-adapter created
clusterrole.rbac.authorization.k8s.io/system:aggregated-metrics-reader created
clusterrolebinding.rbac.authorization.k8s.io/prometheus-adapter created
clusterrolebinding.rbac.authorization.k8s.io/resource-metrics:system:auth-delegator created
clusterrole.rbac.authorization.k8s.io/resource-metrics-server-resources created
configmap/adapter-config created
deployment.apps/prometheus-adapter created
rolebinding.rbac.authorization.k8s.io/resource-metrics-auth-reader created
service/prometheus-adapter created
serviceaccount/prometheus-adapter created
[root@elasticsearch01 promethues]# kubectl apply -f alertmanager/
alertmanager.monitoring.coreos.com/main created
secret/alertmanager-main created
service/alertmanager-main created
serviceaccount/alertmanager-main created
[root@elasticsearch01 promethues]# kubectl apply -f node-exporter/
clusterrole.rbac.authorization.k8s.io/node-exporter created
clusterrolebinding.rbac.authorization.k8s.io/node-exporter created
daemonset.apps/node-exporter created
service/node-exporter created
serviceaccount/node-exporter created
[root@elasticsearch01 promethues]# kubectl apply -f kube-state-metrics/
clusterrole.rbac.authorization.k8s.io/kube-state-metrics created
clusterrolebinding.rbac.authorization.k8s.io/kube-state-metrics created
deployment.apps/kube-state-metrics created
role.rbac.authorization.k8s.io/kube-state-metrics created
rolebinding.rbac.authorization.k8s.io/kube-state-metrics created
service/kube-state-metrics created
serviceaccount/kube-state-metrics created
[root@elasticsearch01 promethues]# kubectl apply -f grafana/
secret/grafana-datasources created
configmap/grafana-dashboard-k8s-cluster-rsrc-use created
configmap/grafana-dashboard-k8s-node-rsrc-use created
configmap/grafana-dashboard-k8s-resources-cluster created
configmap/grafana-dashboard-k8s-resources-namespace created
configmap/grafana-dashboard-k8s-resources-pod created
configmap/grafana-dashboard-k8s-resources-workload created
configmap/grafana-dashboard-k8s-resources-workloads-namespace created
configmap/grafana-dashboard-nodes created
configmap/grafana-dashboard-persistentvolumesusage created
configmap/grafana-dashboard-pods created
configmap/grafana-dashboard-statefulset created
configmap/grafana-dashboards created
deployment.apps/grafana created
service/grafana created
serviceaccount/grafana created
[root@elasticsearch01 promethues]# kubectl apply -f prometheus/
clusterrole.rbac.authorization.k8s.io/prometheus-k8s created
clusterrolebinding.rbac.authorization.k8s.io/prometheus-k8s created
prometheus.monitoring.coreos.com/k8s created
rolebinding.rbac.authorization.k8s.io/prometheus-k8s-config created
rolebinding.rbac.authorization.k8s.io/prometheus-k8s created
rolebinding.rbac.authorization.k8s.io/prometheus-k8s created
rolebinding.rbac.authorization.k8s.io/prometheus-k8s created
role.rbac.authorization.k8s.io/prometheus-k8s-config created
role.rbac.authorization.k8s.io/prometheus-k8s created
role.rbac.authorization.k8s.io/prometheus-k8s created
role.rbac.authorization.k8s.io/prometheus-k8s created
prometheusrule.monitoring.coreos.com/prometheus-k8s-rules created
service/prometheus-k8s created
serviceaccount/prometheus-k8s created
[root@elasticsearch01 promethues]# kubectl apply -f serviceMonitor/
servicemonitor.monitoring.coreos.com/prometheus-operator created
servicemonitor.monitoring.coreos.com/alertmanager created
servicemonitor.monitoring.coreos.com/grafana created
servicemonitor.monitoring.coreos.com/kube-state-metrics created
servicemonitor.monitoring.coreos.com/node-exporter created
servicemonitor.monitoring.coreos.com/prometheus created
servicemonitor.monitoring.coreos.com/kube-apiserver created
servicemonitor.monitoring.coreos.com/coredns created
servicemonitor.monitoring.coreos.com/kube-controller-manager created
servicemonitor.monitoring.coreos.com/kube-scheduler created
servicemonitor.monitoring.coreos.com/kubelet created
```


**6.检查验证**
```
[root@elasticsearch01 promethues]# k -n monitoring get all
NAME                                      READY   STATUS    RESTARTS   AGE
pod/alertmanager-main-0                   2/2     Running   0          23h
pod/alertmanager-main-1                   2/2     Running   0          23h
pod/alertmanager-main-2                   2/2     Running   0          23h
pod/grafana-58dc7468d7-2q8x5              1/1     Running   0          22h
pod/node-exporter-8hdk7                   2/2     Running   0          24h
pod/node-exporter-927pv                   2/2     Running   0          24h
pod/prometheus-adapter-5cd5798d96-lxvz9   1/1     Running   0          24h
pod/prometheus-k8s-0                      3/3     Running   1          23h
pod/prometheus-k8s-1                      3/3     Running   0          23h
pod/prometheus-operator-99dccdc56-bdfd5   1/1     Running   0          24h

NAME                            TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                      AGE
service/alertmanager-main       ClusterIP   10.0.0.197   <none>        9093/TCP                     24h
service/alertmanager-operated   ClusterIP   None         <none>        9093/TCP,9094/TCP,9094/UDP   23h
service/grafana                 NodePort    10.0.0.171   <none>        3000:32767/TCP               24h
service/node-exporter           ClusterIP   None         <none>        9100/TCP                     24h
service/prometheus-adapter      ClusterIP   10.0.0.210   <none>        443/TCP                      24h
service/prometheus-k8s          ClusterIP   10.0.0.89    <none>        9090/TCP                     24h
service/prometheus-operated     ClusterIP   None         <none>        9090/TCP                     23h
service/prometheus-operator     ClusterIP   None         <none>        8080/TCP                     24h

NAME                           DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
daemonset.apps/node-exporter   2         2         2       2            2           kubernetes.io/os=linux   24h

NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/grafana               1/1     1            1           24h
deployment.apps/prometheus-adapter    1/1     1            1           24h
deployment.apps/prometheus-operator   1/1     1            1           24h

NAME                                            DESIRED   CURRENT   READY   AGE
replicaset.apps/grafana-58dc7468d7              1         1         1       24h
replicaset.apps/prometheus-adapter-5cd5798d96   1         1         1       24h
replicaset.apps/prometheus-operator-99dccdc56   1         1         1       24h

NAME                                 READY   AGE
statefulset.apps/alertmanager-main   3/3     23h
statefulset.apps/prometheus-k8s      2/2     23h
```


**7.ingress配置**
```
[root@elasticsearch01 promethues]# cat ingress-yaml/ingress-monitor.yaml 
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: prometheus-ing
  namespace: monitoring
spec:
  rules:
  - host: prometheus-k8s.minminmsn.com
    http:
      paths:
      - backend:
          serviceName: prometheus-k8s
          servicePort: 9090
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: grafana-ing
  namespace: monitoring
spec:
  rules:
  - host: grafana-k8s.minminmsn.com
    http:
      paths:
      - backend:
          serviceName: grafana
          servicePort: 3000
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: alertmanager-ing
  namespace: monitoring
spec:
  rules:
  - host: alertmanager-k8s.minminmsn.com
    http:
      paths:
      - backend:
          serviceName: alertmanager-main
          servicePort: 9093


[root@elasticsearch01 promethues]# kubectl apply -f ingress-yaml/ingress-monitor.yaml 
ingress.extensions/prometheus-ing created
ingress.extensions/grafana-ing created
ingress.extensions/alertmanager-ing created
```


### 浏览器访问
**1.nodeport方式访问**
http://http://192.168.122.214:32767/
默认账号密码admin admin需要重置密码进入
