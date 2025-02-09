---
layout: post
title: kubesphere学习使用
categories: [coder, cloud]
tags: [k8s, kubesphere]
date: 2024-12-02 17:00 +0800
---

## 背景

接上文[kubeadm部署k8s新版](/coder/cloud/2024-12-01-kubeadm部署k8s新版.md)，我们搭建好了k8s集群，同时使用最简单的命令行或kube-dashboard管理集群。这种管理模式用久了之后，会发现以下痛点：
* yaml文件管理困难：yaml文件需要手动编写，且没有版本控制，为此需要引入单独的版本管理工具。另外，yaml文件回滚、对比等功能也不够方便。这是最大的痛点。
* 监控告警困难：dashboard基于k8s自带的metrics-server，监控功能非常有限，且没有告警功能。难以对监控做进一步的优化改造，无法较好的发现问题。
* 集群资源管理不够直观：dashboard提供了基本的资源管理功能，但不够直观，尤其是对于多集群管理。
* 权限管理不够灵活：dashboard的权限管理比较简单，无法满足复杂的权限管理需求。

因此，我们需要一个更加强大、易用的**k8s管理平面**，来替代dashboard。经过调研，我们选择了kubesphere。

## 为什么选择kubesphere

KubeSphere 是在 Kubernetes 之上构建的分布式多租户容器平台，提供全栈的 IT 自动化运维能力，简化企业的 DevOps 工作流。KubeSphere 提供了丰富的企业级功能，包括但不限于：

* 多租户管理：支持多租户管理，可以为不同的团队或项目创建独立的命名空间，实现资源隔离和权限控制。
* 应用管理：支持应用模板、应用商店、应用部署等功能，可以方便地部署和管理容器化应用。
* 监控告警：集成了 Prometheus 和 Grafana，提供了强大的监控和告警功能。
* 日志管理：集成了 Fluentd 和 Elasticsearch，提供了强大的日志管理和查询功能。
* 安全管理：提供了 RBAC、网络策略、存储策略等功能，确保集群的安全性和合规性。
* DevOps：提供了 CI/CD、镜像仓库、代码仓库等功能，支持企业级 DevOps 工作流。

而对于我们来说，kubesphere管理平面最吸引我们的地方是：
* 提供yaml文件统一管理，支持版本控制、回滚、对比等功能。
* 基于 Prometheus 和 Grafana，提供了强大的监控和告警功能。
* 可插拔的组件：包括应用管理、监控告警、日志管理、安全管理、DevOps等，可以根据需求选择安装，提供了极高的灵活性。
* 提供了丰富的文档和社区支持，方便学习和使用。

## kubesphere部署

### 部署规划
kubesphere部署方式有多种，包括helm、k8s部署、二进制部署等。我们选择基于上文现有k8s集群之上，最小化部署kubesphere的部署方式，参考：[在 Kubernetes 上最小化安装 KubeSphere](https://kubesphere.io/zh/docs/v3.4/quick-start/minimal-kubesphere-on-k8s/)

一些前置条件列举如下：
* 选用版本：v3.4.1
* 集群资源：使用当前部署好的k8s集群资源即可，实测可以在 **2 台 4C4G** 虚拟机上完全安装 `k8s + kubesphere + cilium`，并保留一定空余空间提供后续扩展。
    * 占用内存：总分配2G左右，实际3.8G左右；这里主要体现在kube-apiserver、etcd、prometheus上，其他组件占用内存较少。
    * 占用CPU：总分配2核心左右，实际用量不到2%。
* 组件选择：不额外添加可选组件，仅安装核心组件。需要注意各组件引入后对于系统资源的需求，*防止卡死集群*，参考：[启用可插拔组件/概述](https://kubesphere.io/zh/docs/v3.4/pluggable-components/overview/)！包括：
    * kubesphere-system：核心组件，包括ks-apiserver、ks-console、ks-controller-manager等。
    * kubesphere-monitoring-system：基于Prometheus提供监控、告警功能。
  其他将来可能会尝试的组件如下：
    * kubesphere-logging-system：基于Fluentd和Elasticsearch提供日志管理功能。
    * kubesphere-devops-system：基于Jenkins提供CI/CD功能。

### 部署步骤总结

1. 准备yaml部署文件：
    ```shell
    # kubesphere-installer.yaml用于安装kubesphere，cluster-configuration.yaml用于配置集群信息。
    wget https://github.com/kubesphere/ks-installer/releases/download/v3.4.1/kubesphere-installer.yaml
    wget https://github.com/kubesphere/ks-installer/releases/download/v3.4.1/cluster-configuration.yaml

    # kubesphere-delete.sh用于完全移除kubesphere。可以在初始化出错时，先卸载kubesphere，再重新初始化。
    wget https://github.com/kubesphere/ks-installer/blob/release-3.1/scripts/kubesphere-delete.sh
    ```
    如需修改使用私有hub仓库获取镜像进行安装，调整`kubesphere-installer.yaml`文件中的`image`字段获取ks-installer镜像地址，同时调整`cluster-configuration.yaml`文件中的`local_registry`字段使用私有hub仓库地址。

1. 准备默认的存储类，并添加必要的pv用于绑定。这里只需准备了一块pv：`local-pv-1`，用于prometheus监控，大小为20Gi。如下：
    ```shell
    # vi ./storageclass.yaml，添加以下内容：
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: local-path
    provisioner: rancher.io/local-path
    volumeBindingMode: WaitForFirstConsumer

    ---
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: local-pv-1
    spec:
      capacity:
        storage: 5Gi
      volumeMode: Filesystem
      accessModes:
        - ReadWriteOnce
      persistentVolumeReclaimPolicy: Retain
      storageClassName: local-path
      local:
        path: /home/data/k8s/local-pv-1 #使用该路径作为pv，注意路径需要先创建好。
      nodeAffinity:
        required:
          nodeSelectorTerms:
          - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
              - master # 选择kubernetes.io/hostname=master的节点上的pv
    ```

1. 调整cluster-configuration.yaml配置的节点数：
    ```shell
    ...
    monitoring:
      storageClass: ""                 # If there is an independent StorageClass you need for Prometheus, you can specify it here. The default StorageClass is used by default.
      node_exporter:
        port: 9100
        # resources: {}
      kube_rbac_proxy:
        resources: {}
      kube_state_metrics:
        resources: {}
      prometheus:
        replicas: 1  # Prometheus replicas are responsible for monitoring different segments of data source and providing high availability.
        volumeSize: 20Gi  # Prometheus PVC size.
        resources: {}
        operator:
          resources: {}
      alertmanager:
        replicas: 1          # AlertManager Replicas.
        resources: {}
      notification_manager:
        replicas: 1
        resources: {}
        operator:
          resources: {}
        proxy:
          resources: {}
    
    ...
    ```

1. 安装kubesphere：
    ```shell
    # 初始化kubesphere，指定部署文件、集群配置文件、存储类文件。
    kubectl apply -f kubesphere-installer.yaml -f cluster-configuration.yaml -f storageclass.yaml

    # 如遇问题可以随时卸载，然后重装：
    sh ./kubesphere-delete.sh
    ```

1. 检查kubesphere部署状态：
    ```shell
    # 检查kubesphere部署状态，确保所有组件都处于运行状态。注意，这里需要指定集群的名称，例如：ks-installer、ks-console、ks-controller-manager等。
    kubectl logs -n kubesphere-system $(kubectl get pod -n kubesphere-system -l app=ks-installer -o jsonpath='{.items[0].metadata.name}')
    ```

1. 访问kubesphere控制台：
    ```shell
    # 访问kubesphere控制台，默认用户名和密码都是：admin
    http://<your-host-ip>:30880
    ```

## 组件调优

### 控制CRD（Custom Resource Definition）
kubesphere中的组件普遍通过CRD扩展了Kubernetes API，用于实现自定义资源。这些CRD定义了集群中的资源类型，例如集群、命名空间、工作负载等。在kubesphere中，CRD通常由ks-controller-manager组件管理，该组件负责处理CRD的创建、更新和删除操作。

因此，我们需要先了解CRD相关的一些定义，以及如何通过**控制CRD达到调节组件行为**的目的。

理解CRD：
* 定制资源（Custom Resource） 是对 Kubernetes API 的扩展。
* CRD 是 Kubernetes API 的扩展机制，允许用户自定义资源类型。
。。。

### Prometheus监控调优
kubesphere中，Prometheus组件用于监控集群中的各种资源，包括节点、Pod、服务、Ingress等。Prometheus通过收集和存储指标数据，提供实时监控和告警功能。
KubeSphere 3.4 使用 Prometheus Operator 来管理 Prometheus/Alertmanager 配置和生命周期、ServiceMonitor（用于管理抓取配置）和 PrometheusRule（用于管理 Prometheus 记录/告警规则）。因此：
* 如需调优监控抓取指标，可以修改ServiceMonitor（CRD）配置。
* 如需调优告警规则，可以修改PrometheusRule（CRD）配置。
* 如需调优Prometheus本身的配置，可以修改Prometheus（CRD）配置。如调整`k8s`的yaml配置，设置`spec.replicas=1`将集群副本数设置为1，以减少Prometheus的内存占用。

无用指标分析，可以参考：
* [如何精简 Prometheus 的指标和存储占用](https://blog.csdn.net/east4ming/article/details/127916705)
* [优化实践：Prometheus 性能和高基数问题](https://flashcat.cloud/blog/prometheus-performance-and-cardinality-in-practice/)

调整Prometheus采集指标，调整 ServiceMonitor（CRD）配置示例：
  ```yaml
  apiVersion: monitoring.coreos.com/v1
  kind: ServiceMonitor
  metadata:
    labels:
      app.kubernetes.io/name: kube-state-metrics
      app.kubernetes.io/version: 1.9.7
    name: kube-state-metrics
    namespace: kube-system
  spec:
    endpoints:
    - bearerTokenSecret:
        key: ""
      interval: 15s # 该参数为采集频率，您可以调大以降低数据存储费用，例如不重要的指标可以改为 300s，可以降低20倍的监控数据采集量
      port: http-metrics
      scrapeTimeout: 15s # 该参数为采集超时时间，Prometheus 的配置要求采集超时时间不能超过采集间隔，即：scrapeTimeout <= interval
    metricRelabelings: # 针对每个采集到的点都会做如下处理
    - sourceLabels: ["__name__"] # 要检测的label名称，__name__ 表示指标名称，也可以是任意这个点所带的label
      regex: kube_node_info|kube_node_role # 上述label是否满足这个正则，在这里，我们希望__name__满足kube_node_info或kube_node_role
      action:  keep # 如果点满足上述条件，则保留，否则就自动抛弃
    jobLabel: app.kubernetes.io/name
    namespaceSelector: {}
    selector:
      matchLabels:
        app.kubernetes.io/name: kube-state-metrics
  ```


### kube-apiserver内存占用优化
kube-apiserver是Kubernetes集群的核心组件之一，负责处理集群中所有资源的请求。kube-apiserver的性能和资源占用情况对集群的整体性能有很大影响。

kube-apiserver的内存占用主要由两部分组成：
* 缓存：kube-apiserver会缓存一些常用的资源对象，例如Pod、Service、Ingress等，以便快速响应请求。缓存的大小可以通过`--etcd-compaction-interval-minutes`参数进行配置。默认情况下，该参数的值为10分钟，表示每10分钟对缓存进行一次压缩，以减少内存占用。
* 请求处理：kube-apiserver在处理请求时，会创建一些临时对象，例如请求对象、响应对象等。这些临时对象会占用一定的内存空间。可以通过调整`--max-requests-inflight`和`--max-mutating-requests-inflight`参数来限制并发请求的数量，从而减少内存占用。

kube-apiserver参数可以通过kube-apiserver的配置文件进行修改，例如：
```shell
# vi /etc/kubernetes/manifests/kube-apiserver.yaml
```
在配置文件中，找到`command`字段，添加或修改以下参数：
```shell
--etcd-compaction-interval-minutes=5
--max-requests-inflight=500
--max-mutating-requests-inflight=200
```
修改完成后，kube-apiserver会自动重新加载配置文件，并应用新的参数设置。
