# Multi-Cluster Monitoring Setup with Remote Read

## Overview

This guide walks you through installing monitoring tools for a multi-cluster environment using Prometheus, Grafana, and Thanos, with an example setup across two clusters: `Obserbility` and `Database`.

---

## Technical Overview

### Thanos Component

`Thanos sidecar`: connect prometheus to Thanos system.
`Thanos query`: Read data from prometheus tsdb or longterm storage by `Thanos Storage Gateway` or from `Thanos query` in local prometheus
`Thanos Storage Gateway`: Read and write data from and to longterm storage
`Thanos Compactor`: Group block data, downsample old data
`Thanos Receiver`: Receive data from remote write(prometheus will push data to `Thanos Receiver`)

## How to setup

### Cluster: Obserbility

#### Create Namespaces

- `monitoring`
- `thanos`

#### Install Monitoring Tools in `monitoring` Namespace

```bash
helm install monitoring prometheus-community/kube-prometheus-stack --version=3.12.0 -n monitoring -f scrape-prometheus/helm/kube-prometheus-stack.yaml
```

Includes:

- Grafana
- Prometheus
- Prometheus Operator
- Thanos Sidecar

#### Install Thanos Query in `thanos` Namespace

```bash
helm install thanos bitnami/thanos --version=16.0.4 --namespace thanos -f thanos/helm/thanos-query.yaml
```

This allows reading data from multiple clusters.

---

### Cluster: Database (Monitored Cluster)

#### Create Namespaces

- `monitoring`
- `mongodb`
- `exporter`

#### Deploy MongoDB in `mongodb` Namespace

##### For ARM

```bash
kubectl -n mongodb create deployment mongodb --image=mongodb/mongodb-comunity-server:latest
kubectl -n mongodb expose deployment mongodb --port=27017 --target-port=27017
```

##### For AMD

Use Helm (e.g., Bitnami's MongoDB chart includes exporter by default).

#### Deploy MongoDB Exporter in `exporter` Namespace

```bash
kubectl -n exporter apply -f metrics-exporter/mongodb/k8s/deployment.yaml
kubectl -n exporter apply -f metrics-exporter/mongodb/k8s/service.yaml
```

#### Install Monitoring Stack in `monitoring` Namespace

```bash
helm install monitoring prometheus-community/kube-prometheus-stack --version=3.12.0 -n monitoring -f scrape-prometheus/helm/kube-prometheus-stack.yaml
```

#### Expose Thanos Sidecar

```bash
kubectl -n monitoring apply -f thanos/helm/thanos-sidecar-ingress.yaml
```

#### Apply ServiceMonitor

ServiceMonitor is custom CRD of prometheus operator. When apply ServiceMonitor, prometheus will scrape resource automaticaly.
Ensure `namespaceSelector.matchNames` includes:

```yaml
namespaceSelector:
  matchNames:
    - exporter <--- it must be namespace of exporter deployment
```

Apply ServiceMonitor:

```bash
kubectl -n monitoring apply -f scrape-prometheus/helm/service-monitor/mongo-exporter.yaml
```

Note: ServiceMonitor need to install in the same namespace with prometheus

### Back to Obserbility Cluster

Update Thanos Query endpoints with new sidecar info from the Database cluster, then redeploy:

```bash
helm upgrade thanos bitnami/thanos -n thanos -f thanos/helm/thanos-query.yaml
```

---

Bonus: If you want to setup monitoring in vm instance(ec2, gce,...), install all docker file in scrape-prometheus/docker. Then, back to step Update Thanos Query endpoints

_End of guide._
