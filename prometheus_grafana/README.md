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
helm install monitoring prometheus-community/kube-prometheus-stack --version=3.12.0 -n monitoring -f prom_grafana-values.yaml
```

Includes:

- Grafana
- Prometheus
- Thanos Sidecar
- Alert Manager

#### Install Thanos Query in `thanos` Namespace

```bash
helm install thanos bitnami/thanos --namespace thanos -f values-thanos-query.yaml
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
kubectl -n exporter apply -f k8s/mongo-exporter/deployment.yaml
kubectl -n exporter apply -f k8s/mongo-exporter/service.yaml
```

#### Install Monitoring Stack in `monitoring` Namespace

```bash
helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring -f values-book.yaml
```

#### Expose Thanos Sidecar

```bash
kubectl -n monitoring apply -f thanos-sidecar-ingress.yaml
```

#### Apply ServiceMonitor

Ensure `namespaceSelector.matchNames` includes:

```yaml
namespaceSelector:
  matchNames:
    - exporter <--- it must be namespace of exporter deployment
```

Apply ServiceMonitor:

```bash
kubectl -n monitoring apply -f service-monitor/mongo-exporter.yaml
```

Note: ServiceMonitor need to install in the same namespace with prometheus

### Back to Obserbility Cluster

Update Thanos Query endpoints with new sidecar info from the Database cluster, then redeploy:

```bash
helm upgrade thanos bitnami/thanos -n thanos -f values-thanos-query.yaml
```

---

_End of guide._
