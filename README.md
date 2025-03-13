# Retina Installation Guide

Retina is a network observability tool for Kubernetes clusters. This guide explains how to install and configure Retina using Helm.

## Prerequisites

- Kubernetes cluster
- Helm v3 installed
- Access to ghcr.io container registry
- Prometheus installed in your cluster

## Installation Steps

Install Retina using the below command:

```shell
helm upgrade --install retina oci://ghcr.io/microsoft/retina/charts/retina \
    --version v0.0.27 \
    --namespace kube-system \
    --set image.tag=v0.0.27 \
    --set operator.tag=v0.0.27 \
    --set image.pullPolicy=Always \
    --set logLevel=info \
    --set os.windows=true \
    --set operator.enabled=true \
    --set operator.enableRetinaEndpoint=true \
    --skip-crds \
    --set enabledPlugin_linux="\[dropreason\,packetforward\,linuxutil\,dns\,packetparser\]" \
    --set enablePodLevel=true \
    --set enableAnnotations=true \
    --set remoteContext=true
```

## Install MetricsConfiguration CRD

Apply the MetricsConfiguration CRD using the following command or create a file named `metricsconfigcrd.yaml` with the following content change the `namespace` to your namespace:

```yaml
    apiVersion: retina.sh/v1alpha1
    kind: MetricsConfiguration
    metadata:
    name: metricsconfigcrd
    spec:
    contextOptions:
        - metricName: drop_count
        sourceLabels:
            - ip
            - podname
            - port
        additionalLabels:
            - direction
        - metricName: forward_count
        sourceLabels:
            - ip
            - podname
            - port
        destinationLabels:
            - ip
            - podname
            - port
        additionalLabels:
            - direction
    namespaces:
        include:
        # Production namespace
        - prod
        # Staging/pre-production namespace  
        - staging
```

Apply the MetricsConfiguration CRD using the following command:

```shell
kubectl apply -f metricsconfigcrd.yaml
```

## Prometheus Configuration

Add the following job configuration to your Prometheus configuration to scrape Retina metrics:

```yaml
scrape_configs:
  - job_name: "retina-pods"
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_container_name]
        action: keep
        regex: retina(.*)
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        separator: ":"
        regex: ([^:]+)(?::\d+)?
        target_label: __address__
        replacement: ${1}:${2}
        action: replace
      - source_labels: [__meta_kubernetes_pod_node_name]
        action: replace
        target_label: instance
    metric_relabel_configs:
      - source_labels: [__name__]
        action: keep
        regex: (.*)
```
make sure in kube-state metrics have topology labels zone is coming in the metrics for below metrics. for more information please refer to [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics)

```
sum by (node, label_topology_kubernetes_io_zone, job) (kube_node_labels{job="kube-state-metrics"})
```
![alt text](https://github.com/anup1384/k8s-topology/blob/main/sc-1.png)

if metrics are not coming then please check the kube-state-metrics pod logs for more information. or add argument `- --metric-labels-allowlist=nodes=[topology.kubernetes.io/zone]` to kube-state-metrics pod.

![alt text](https://github.com/anup1384/k8s-topology/blob/main/sc-2.png)
