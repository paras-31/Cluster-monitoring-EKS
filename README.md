# Cluster-monitoring-EKS
using prometheus and grafana with iml file 
# Kubernetes Cluster Monitoring with Prometheus and Grafana

This directory contains deployment files for setting up a complete monitoring stack on an EKS cluster using Prometheus and Grafana.

---

## 📋 Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Components](#components)
4. [File Structure](#file-structure)
5. [Installation](#installation)
6. [Configuration Explained](#configuration-explained)
7. [Accessing the Services](#accessing-the-services)
8. [Common Queries](#common-queries)
9. [Troubleshooting](#troubleshooting)

---

## 🎯 Overview

This monitoring stack provides:
- **Real-time metrics collection** from Kubernetes clusters
- **Centralized metric storage** with Prometheus
- **Beautiful dashboards** with Grafana
- **Service discovery** for automatic target detection
- **Role-Based Access Control (RBAC)** for security

### Key Features
✅ Automatic discovery of Kubernetes API servers, nodes, and pods  
✅ 15-day metric retention  
✅ Lightweight deployments with resource limits  
✅ Health checks for reliability  
✅ Provisioned dashboards in Grafana  

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────┐
│          Kubernetes EKS Cluster                     │
├─────────────────────────────────────────────────────┤
│                                                     │
│  ┌──────────────────────────────────────────────┐  │
│  │        Monitoring Namespace                  │  │
│  │                                              │  │
│  │  ┌─────────────────┐   ┌──────────────────┐ │  │
│  │  │   Prometheus    │   │     Grafana      │ │  │
│  │  │   (Port 9090)   │◄──┤   (Port 3000)    │ │  │
│  │  │                 │   │                  │ │  │
│  │  └────────┬────────┘   └──────────────────┘ │  │
│  │           │                                  │  │
│  │           └──► Scrapes Metrics From         │  │
│  │               - API Servers                 │  │
│  │               - Nodes                       │  │
│  │               - Pods (annotated)            │  │
│  │               - Prometheus itself           │  │
│  │                                              │  │
│  └──────────────────────────────────────────────┘  │
│                                                     │
└─────────────────────────────────────────────────────┘
         ↓
    Your Computer (via port-forward)
    localhost:9090 (Prometheus)
    localhost:3000 (Grafana)
```

---

## 📦 Components

### 1. **Prometheus**
**What:** Time-series database that collects and stores metrics  
**Why:** Central metrics collection point for the cluster  
**How:** Scrapes endpoints at regular intervals (5s by default)

**Resources:**
- Memory: 256Mi (request) / 512Mi (limit)
- CPU: 250m (request) / 500m (limit)
- Storage: emptyDir (temporary, lost on restart)

### 2. **Grafana**
**What:** Visualization and dashboard platform  
**Why:** Display metrics in beautiful, understandable dashboards  
**How:** Queries Prometheus datasource and renders visualizations

**Resources:**
- Memory: 128Mi (request) / 256Mi (limit)
- CPU: 100m (request) / 200m (limit)
- Storage: emptyDir (temporary, lost on restart)

### 3. **Kubernetes Service Discovery**
**What:** Automatic detection of monitoring targets  
**Why:** Reduces manual configuration, finds new targets automatically  
**How:** Uses Kubernetes API to list endpoints, nodes, and pods

### 4. **RBAC (Role-Based Access Control)**
**What:** Security permissions for Prometheus and Grafana  
**Why:** Restrict what data each pod can access  
**How:** ServiceAccounts, ClusterRoles, and ClusterRoleBindings

---

## 📁 File Structure

```
cluster-monitoring/
├── README.md                          # This file
├── prometheus-deployment.yml          # Prometheus configuration & deployment
├── grafana-deployment.yml             # Grafana configuration & deployment
└── storageclass-ebs.yml              # (Optional) EBS storage configuration
```

---

## 🚀 Installation

### Prerequisites
- EKS cluster running
- `kubectl` configured to access your cluster
- `kubectl` command-line tool installed

### Step 1: Deploy Prometheus
```bash
kubectl apply -f prometheus-deployment.yml
```

**What this creates:**
- Namespace: `monitoring`
- ConfigMap: Prometheus configuration
- ServiceAccount: `prometheus`
- ClusterRole & ClusterRoleBinding: RBAC permissions
- Service: `prometheus-service` (NodePort on port 30090)
- Deployment: Prometheus pod

### Step 2: Deploy Grafana
```bash
kubectl apply -f grafana-deployment.yml
```

**What this creates:**
- ServiceAccount: `grafana`
- ClusterRole & ClusterRoleBinding: RBAC permissions
- ConfigMaps: Datasources and dashboards
- Service: `grafana-service` (LoadBalancer on port 3000)
- Deployment: Grafana pod

### Step 3: Verify Deployment
```bash
kubectl get pods -n monitoring
kubectl get svc -n monitoring
```

Both pods should show `1/1 Running`.

---

## 🔧 Configuration Explained

### Prometheus ConfigMap (`prometheus.yml`)

#### Global Settings
```yaml
global:
  scrape_interval: 5s              # How often to collect metrics
  evaluation_interval: 5s          # How often to evaluate alert rules
  external_labels:
    cluster: 'eksdemo1'            # Label all metrics with cluster name
    environment: 'production'      # Mark as production environment
```

#### Scrape Configs

##### 1. Prometheus Itself
```yaml
- job_name: 'prometheus'
  static_configs:
    - targets: ['localhost:9090']
```
**Why:** Monitor Prometheus's own health and performance

**Metrics Example:**
- `prometheus_build_info` - Prometheus version info
- `prometheus_tsdb_metric_chunks_created_total` - Metrics stored
- `up{job="prometheus"}` - Prometheus availability (1=UP, 0=DOWN)

---

##### 2. Kubernetes API Servers
```yaml
- job_name: 'kubernetes-apiservers'
  kubernetes_sd_configs:
    - role: endpoints
  scheme: https
  tls_config:
    ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
  bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
```

**Why:** Monitor control plane health  
**How:** Auto-discovers API server endpoints via Kubernetes API  
**Metrics Example:**
- `apiserver_request_total` - Total API requests
- `apiserver_latency_seconds` - API response time

---

##### 3. Kubernetes Nodes
```yaml
- job_name: 'kubernetes-nodes'
  kubernetes_sd_configs:
    - role: node
  scheme: https
  relabel_configs:
    - source_labels: [__address__]
      replacement: ${1}:10250       # Connect to kubelet port
      target_label: __address__
```

**Why:** Monitor node-level metrics (CPU, memory, disk)  
**How:** Connects to kubelet on each node (port 10250)  
**Metrics Example:**
- `node_cpu_seconds_total` - CPU usage
- `node_memory_MemAvailable_bytes` - Available memory

---

##### 4. Kubernetes Pods
```yaml
- job_name: 'kubernetes-pods'
  kubernetes_sd_configs:
    - role: pod
  relabel_configs:
    - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
      action: keep
      regex: 'true'                 # Only scrape pods with annotation
```

**Why:** Monitor pod-level application metrics  
**How:** Auto-discovers pods with `prometheus.io/scrape: "true"` annotation  
**How to enable:** Add to pod annotations:
```yaml
annotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "8080"        # Custom port (default 9090)
  prometheus.io/path: "/metrics"    # Custom path (default /metrics)
```

---

### RBAC Permissions

#### ServiceAccount
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: monitoring
```
**Why:** Pod identity for authentication with Kubernetes API

---

#### ClusterRole
```yaml
rules:
  - apiGroups: [""]
    resources:
      - nodes
      - nodes/proxy
      - services
      - endpoints
      - pods
    verbs: ["get", "list", "watch"]
```

**What each permission does:**

| Resource | Why Needed | Use Case |
|----------|-----------|----------|
| `nodes` | Discover and monitor nodes | Get node metrics |
| `nodes/proxy` | Access kubelet metrics | Get node stats |
| `services` | Discover services | Find monitoring targets |
| `endpoints` | Discover actual pod IPs | Route to correct pods |
| `pods` | Discover pods | Find pods to monitor |

---

### Services

#### Prometheus Service (NodePort)
```yaml
type: NodePort
ports:
  - port: 9090
    targetPort: 9090
    nodePort: 30090
```

**Why NodePort?**
- Accessible from outside cluster
- Each node exposes port 30090
- Access via: `http://<node-ip>:30090`

---

#### Grafana Service (LoadBalancer)
```yaml
type: LoadBalancer
ports:
  - port: 3000
    targetPort: 3000
```

**Why LoadBalancer?**
- Easier access from outside (gets public IP)
- Auto load-balancing
- More user-friendly than NodePort

---

## 🌐 Accessing the Services

### Method 1: Port-Forward (Recommended for Testing)

**Prometheus:**
```bash
kubectl port-forward -n monitoring svc/prometheus-service 9090:9090
# Access: http://localhost:9090
```

**Grafana:**
```bash
kubectl port-forward -n monitoring svc/grafana-service 3000:3000
# Access: http://localhost:3000
# Login: admin / admin123
```

### Method 2: NodePort (Direct Access)

**Get Node IP:**
```bash
kubectl get nodes -o wide
```

**Prometheus:**
```
http://<node-ip>:30090
```

### Method 3: LoadBalancer (Grafana)

**Get External IP:**
```bash
kubectl get svc -n monitoring grafana-service
```

**Access:**
```
http://<external-ip>:3000
```

---

## 📊 Common Queries

### Check Target Health
```
up
```
**Result:** Shows all scrape targets with status (1=UP, 0=DOWN)

---

### API Servers Status
```
up{job="kubernetes-apiservers"}
```
**Result:** How many API servers are running

---

### Pods Being Monitored
```
up{job="kubernetes-pods"}
```
**Result:** Which pods are actively being scraped

---

### Prometheus Health
```
prometheus_build_info
```
**Result:** Prometheus version and build info

---

### Memory Usage
```
node_memory_MemAvailable_bytes
```
**Result:** Available memory on nodes (in bytes)

---

### Container CPU Usage
```
container_cpu_usage_seconds_total
```
**Result:** Total CPU time used by containers

---

### Pod Information
```
kube_pod_info
```
**Result:** Metadata about all pods

---

## 🐛 Troubleshooting

### Issue: Prometheus targets showing DOWN

**Cause:** RBAC permissions insufficient  
**Solution:** Verify ClusterRole has required permissions:
```bash
kubectl describe clusterrole prometheus -n monitoring
```

**Check specific permission:**
```bash
kubectl auth can-i list nodes --as=system:serviceaccount:monitoring:prometheus
```

---

### Issue: Grafana can't connect to Prometheus

**Cause:** Using `localhost:9090` inside cluster  
**Solution:** Use Kubernetes Service DNS:
```
http://prometheus-service:9090
```

**Why:** Inside cluster, use service names not localhost

---

### Issue: Pods not appearing in Prometheus

**Cause:** Missing annotations on pod spec  
**Solution:** Add to your pod:
```yaml
annotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "8080"      # Your app's metrics port
  prometheus.io/path: "/metrics"  # Your metrics endpoint
```

---

### Issue: Metrics disappearing after pod restart

**Cause:** Using emptyDir storage (temporary)  
**Solution:** To persist metrics, use PVC:
1. Create PVC with StorageClass
2. Reference in Prometheus deployment
3. Note: This increases pod startup time

---

### View Logs

**Prometheus:**
```bash
kubectl logs -n monitoring -l app=prometheus --tail=50
```

**Grafana:**
```bash
kubectl logs -n monitoring -l app=grafana --tail=50
```

---

## 🔍 What's Being Monitored?

### By Default ✅
- Kubernetes API Server health and performance
- Pod discovery and lifecycle
- Service and endpoint availability
- Prometheus itself

### Not Monitored ❌
- Node-level detailed metrics (requires Node Exporter DaemonSet)
- Application-specific metrics (unless app exports them)
- Logs (requires separate logging solution like ELK)

### To Monitor Your App
Add to your pod spec:
```yaml
annotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "8080"
  prometheus.io/path: "/metrics"
```

Your app must expose metrics on the specified port/path in Prometheus format.

---

## 📈 Next Steps

### 1. Create Custom Dashboards
- Go to Grafana
- Create new dashboard
- Add panels with custom queries
- Save and share

### 2. Set Up Alerts
- In Prometheus: Create alerting rules
- In Grafana: Configure alert notifications
- Send to Slack, email, etc.

### 3. Add Node Exporter
For detailed node metrics:
```bash
kubectl apply -f node-exporter-daemonset.yml
```

### 4. Scale Storage
Replace emptyDir with PVC:
- Edit prometheus-deployment.yml
- Add PVC reference
- Persist metrics for long-term analysis

---

## 🔐 Security Notes

### Current Setup
- ServiceAccount-based authentication ✅
- RBAC permissions restricted ✅
- TLS enabled for API communication ✅
- Temporary storage (safe from data loss) ✅

### Production Recommendations
- [ ] Use persistent storage for Prometheus
- [ ] Enable network policies to restrict traffic
- [ ] Change Grafana default password
- [ ] Set up backup strategy
- [ ] Configure SSL/TLS for Grafana UI
- [ ] Implement authentication (OAuth, LDAP)
- [ ] Set up audit logging

---

## 📚 Resources

- **Prometheus Docs:** https://prometheus.io/docs/
- **Grafana Docs:** https://grafana.com/docs/
- **Kubernetes Monitoring:** https://kubernetes.io/docs/tasks/debug-application-cluster/resource-metrics-pipeline/
- **PromQL Guide:** https://prometheus.io/docs/prometheus/latest/querying/basics/

---

## ❓ FAQ

**Q: How much data does Prometheus store?**  
A: 15 days by default (configured in `--storage.tsdb.retention.time`). After that, old data is automatically deleted.

**Q: Can I monitor multiple clusters?**  
A: Yes! Use `external_labels` in Prometheus config to identify which cluster the metrics come from.

**Q: How do I backup Prometheus data?**  
A: Use PVC with regular volume snapshots or backup the `/prometheus` directory periodically.

**Q: Can Prometheus alerting work without Grafana?**  
A: Yes! Prometheus has built-in alerting rules independent of Grafana. Grafana is only for visualization.

**Q: What's the difference between labels and annotations?**  
A: Labels are indexed and used for querying. Annotations are metadata not indexed (like Prometheus scrape config).

---

## 📝 Version Information

- **Prometheus:** Latest (prom/prometheus:latest)
- **Grafana:** Latest (grafana/grafana:latest)
- **Kubernetes:** EKS compatible (tested on 1.28+)
- **Created:** December 2025

---

**Happy Monitoring! 🎉**
