# Advanced Grafana with Dashboard Setup - Complete Guide

## 📋 Overview

This folder contains a **production-ready Kubernetes cluster monitoring stack** with:
- **Prometheus** - Metrics collection and storage
- **Grafana** - Metrics visualization and dashboards
- **Node Exporter** - Node-level metrics collection
- **Pre-built Dashboard** - Cluster metrics visualization

This setup allows you to monitor:
- ✅ Node CPU, Memory, and Disk utilization
- ✅ Pod resource consumption
- ✅ Pod distribution across nodes
- ✅ Cluster load and performance
- ✅ Container restarts and pod health

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────┐
│           Kubernetes Cluster (EKS)                  │
├─────────────────────────────────────────────────────┤
│                                                     │
│  ┌──────────────┐  ┌──────────────┐ ┌────────┐   │
│  │  Node 1      │  │  Node 2      │ │ Node N │   │
│  │              │  │              │ │        │   │
│  │ ┌──────────┐ │  │ ┌──────────┐ │ │┌──────┐│   │
│  │ │Node      │ │  │ │Node      │ │ ││Node  ││   │
│  │ │Exporter  │ │  │ │Exporter  │ │ ││Export││   │
│  │ │:9100     │ │  │ │:9100     │ │ ││:9100 ││   │
│  │ └────┬─────┘ │  │ └────┬─────┘ │ │└───┬──┘│   │
│  └──────┼────────┘  └──────┼────────┘ └────┼───┘   │
│         │                  │               │       │
│         └──────────┬───────┴───────────────┘       │
│                    │ (scrapes metrics)             │
│          ┌─────────▼──────────┐                    │
│          │  Prometheus        │                    │
│          │  Port: 9090        │                    │
│          │  Storage: 15 days  │                    │
│          └─────────┬──────────┘                    │
│                    │ (queries metrics)             │
│          ┌─────────▼──────────┐                    │
│          │  Grafana           │                    │
│          │  Port: 3000        │                    │
│          │  Dashboard: Auto   │                    │
│          └────────────────────┘                    │
│                    │ (visualizes)                  │
└────────────────────┼─────────────────────────────┘
                     │
              ┌──────▼──────┐
              │  Your Browser │
              │ localhost:3000 │
              └────────────────┘
```

---

## 📁 Files in This Folder

### 1. **prometheus-deployment.yml**
**What it does:**
- Deploys Prometheus as a single-replica Deployment
- Configures scrape jobs for cluster metrics
- Sets up RBAC permissions
- Defines retention policy (15 days)

**Key components:**
- ConfigMap: Prometheus configuration
- ServiceAccount & RBAC: Permissions to access Kubernetes APIs
- Deployment: Prometheus container
- Service: NodePort exposed on port 30090

### 2. **node-exporter-deployment.yml**
**What it does:**
- Deploys Node Exporter as a DaemonSet (one per node)
- Collects node-level metrics (CPU, Memory, Disk, Network)
- Exposes metrics on port 9100

**Key components:**
- DaemonSet: Runs on every node
- ServiceAccount & RBAC: Permissions for node access
- Service: Headless service for discovery
- Tolerations: Runs on all nodes including tainted ones

**Why Node Exporter?**
- EKS kubelet API requires elevated permissions
- Node Exporter is simpler and more reliable
- Collects comprehensive node metrics
- Works with any Kubernetes distribution

### 3. **grafana-deployment.yml**
**What it does:**
- Deploys Grafana for visualization
- Auto-provisions Prometheus data source
- Auto-loads the cluster metrics dashboard
- Sets up default admin credentials

**Key components:**
- ConfigMap: Datasource configuration
- ConfigMap: Dashboard provider configuration
- Deployment: Grafana container
- Service: NodePort exposed on port 30300
- Environment variables for auto-configuration

### 4. **grafana-cluster-dashboard-complete.yml**
**What it does:**
- Defines a pre-built Grafana dashboard
- 10 panels for cluster monitoring
- Auto-loaded when Grafana starts

**Dashboard panels:**
1. Node CPU Usage % - Time series per node
2. Node Memory Usage % - Time series per node
3. Node Load Average - 1min & 5min trends
4. CPU Cores per Node - Static view
5. Pod Distribution - Pie chart per node
6. Total Running Pods - Status card
7. Top 10 Memory Consuming Pods - Bar chart
8. Top 10 CPU Consuming Pods - Bar chart
9. Pod Count per Node - Bar chart
10. Node Disk Usage % - Trend with thresholds

### 5. **PROMETHEUS_METRICS_GUIDE.md**
Complete reference for 20+ Prometheus queries including:
- Cluster load metrics
- Node CPU/Memory metrics
- Pod resource metrics
- Container metrics
- Example queries for each

### 6. **GRAFANA_ACCESS_GUIDE.md**
Step-by-step guide for:
- Accessing Grafana
- Logging in
- Verifying Prometheus connection
- Creating custom dashboards
- Troubleshooting

### 7. **GRAFANA_DASHBOARD_GUIDE.md**
Detailed guide for:
- Understanding each dashboard panel
- Interpreting metrics
- Health thresholds
- Customizing dashboards
- Exporting/sharing dashboards

---

## 🚀 Quick Start

### Step 1: Deploy Everything
```bash
# Navigate to this folder
cd /Users/Paras_Kamboj/eks-work/cluster-monitoring

# Deploy Prometheus
kubectl apply -f prometheus-deployment.yml

# Deploy Node Exporter
kubectl apply -f node-exporter-deployment.yml

# Deploy Grafana
kubectl apply -f grafana-deployment.yml

# Deploy Dashboard
kubectl apply -f grafana-cluster-dashboard-complete.yml

# Verify all pods are running
kubectl get pods -n monitoring
```

### Step 2: Access Grafana
```bash
# Port-forward Grafana
kubectl port-forward -n monitoring svc/grafana-service 3000:3000

# Open browser: http://localhost:3000
# Login: admin / admin123
```

### Step 3: View Dashboard
1. Click **Home** (top-left)
2. Select **Kubernetes Cluster Metrics**
3. View all 10 panels with cluster metrics

---

## 📊 What You Can Monitor

### Node Level Metrics
- **CPU Usage %** - How much CPU is being used
- **Memory Usage %** - How much memory is being used
- **Memory Available (GB)** - Free memory per node
- **Disk Usage %** - Storage utilization
- **Load Average** - System load (1min, 5min, 15min)
- **Network I/O** - Bytes in/out per second
- **CPU Cores** - Number of cores per node

### Pod Level Metrics
- **Top CPU Consumers** - Which pods use most CPU
- **Top Memory Consumers** - Which pods use most memory
- **Pod Count per Node** - Load distribution
- **Pod Distribution** - Pie chart across nodes
- **Container Restarts** - Crashing containers
- **Pod Creation Rate** - New pods per second
- **Pod Status** - Running/Pending/Failed counts

### Cluster Level Metrics
- **Total Running Pods** - Cluster pod count
- **API Server Requests** - Kubernetes API load
- **API Server Latency** - API performance
- **Node Status** - Ready/NotReady nodes
- **Deployment Replica Status** - Scaling info

---

## 🔧 How Everything Works

### Metric Collection Flow

```
Nodes (System Metrics)
    ↓
Node Exporter (:9100)
    ↓
Prometheus (scrapes every 15s)
    ↓
Prometheus Storage (TSDB - 15 days retention)
    ↓
Grafana (queries Prometheus)
    ↓
Dashboard Panels (visualized on browser)
```

### Configuration Chain

```
1. prometheus-deployment.yml
   └─ ConfigMap: prometheus.yml
      └─ Scrape jobs:
         ├─ prometheus (self)
         ├─ kubernetes-apiservers
         ├─ kubernetes-nodes
         ├─ kubernetes-cadvisor
         ├─ kubernetes-pods
         └─ node-exporter (NEW)

2. node-exporter-deployment.yml
   └─ DaemonSet: Runs on every node
      └─ Exposes: /metrics on :9100

3. grafana-deployment.yml
   └─ ConfigMap: Datasource config
   └─ ConfigMap: Dashboard provider
   └─ Mounts: grafana-cluster-dashboard-complete.yml

4. grafana-cluster-dashboard-complete.yml
   └─ 10 pre-built panels
      └─ Connected to Prometheus datasource
```

---

## ❓ Why Each Component?

### Why Prometheus?
- **Time-series database** - Optimized for metrics
- **Pull-based** - Prometheus pulls from targets (more secure)
- **Query language (PromQL)** - Powerful metric transformations
- **Built-in alerting** - Can trigger alerts on thresholds
- **Industry standard** - Used by millions

### Why Grafana?
- **Beautiful dashboards** - Professional visualizations
- **Multi-datasource** - Can query multiple data sources
- **Templating** - Create dynamic dashboards
- **Alerts & Notifications** - Email, Slack, PagerDuty
- **Authentication** - LDAP, OAuth, SAML support

### Why Node Exporter?
- **Node metrics** - CPU, memory, disk, network
- **Lightweight** - DaemonSet uses minimal resources
- **Reliable** - Stable, proven exporter
- **Works everywhere** - All Kubernetes distributions
- **Pre-built queries** - Well-known metric names

### Why Pre-built Dashboard?
- **No manual setup** - Auto-loads when Grafana starts
- **Best practices** - Metrics are pre-configured
- **Time saving** - Start monitoring immediately
- **Consistent** - Same dashboard across environments
- **Shareable** - Easy to export and share

---

## 🎯 Key Metrics to Watch

### Critical Thresholds

| Metric | Good | Warning | Critical |
|--------|------|---------|----------|
| Node CPU % | < 60% | 60-80% | > 80% |
| Node Memory % | < 70% | 70-85% | > 85% |
| Node Disk % | < 70% | 70-90% | > 90% |
| Load Average | < cores | cores - 2×cores | > 2×cores |
| Container Restarts | 0/day | < 5/day | > 5/day |
| Pod Pending | 0 | 1-5 | > 5 |

### What They Mean

**High CPU Usage:**
- Workloads are computationally heavy
- Consider: Optimization, vertical scale, horizontal scale

**High Memory Usage:**
- Memory limits may be exceeded
- Consider: Increase memory limit, optimize code, scale out

**High Disk Usage:**
- Storage is filling up
- Consider: Clean logs, add storage, scale storage

**High Load Average:**
- System is under stress
- Consider: CPU/Memory scaling, workload optimization

---

## 📈 Need More Monitoring?

### Add Custom Metrics

#### Option 1: Add Application Metrics
```yaml
# Add to your application pod annotations
annotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "8080"
  prometheus.io/path: "/metrics"
```

Your app must expose metrics in Prometheus format on port 8080/metrics.

#### Option 2: Add Custom Exporter
```bash
# Example: Deploy Redis Exporter
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-exporter
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis-exporter
  template:
    metadata:
      labels:
        app: redis-exporter
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9121"
    spec:
      containers:
      - name: redis-exporter
        image: oliver006/redis_exporter:latest
        ports:
        - containerPort: 9121
        env:
        - name: REDIS_ADDR
          value: "redis-service:6379"
EOF
```

#### Option 3: Add kube-state-metrics
For advanced Kubernetes state metrics:
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install kube-state-metrics prometheus-community/kube-state-metrics -n monitoring
```

### Add New Dashboard Panel

#### Step 1: Access Grafana
```bash
kubectl port-forward -n monitoring svc/grafana-service 3000:3000
# Open http://localhost:3000
```

#### Step 2: Edit Dashboard
1. Click on dashboard **title** → **Edit**
2. Click **Add panel**
3. Select **Prometheus** datasource

#### Step 3: Add Query
```promql
# Example: Pod restart rate
sum(rate(kube_pod_container_status_restarts_total[5m])) by (namespace)
```

#### Step 4: Configure Display
1. Set title, units, legend
2. Click **Apply**
3. Click **Save dashboard**

### Add Alerting

#### Step 1: Create Alert Rule
```bash
# Create alert for high CPU
kubectl apply -f - <<EOF
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: high-cpu-alert
  namespace: monitoring
spec:
  groups:
  - name: kubernetes
    rules:
    - alert: NodeHighCPU
      expr: (1 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance)) > 0.8
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Node {{ \$labels.instance }} CPU is high"
EOF
```

#### Step 2: Configure Notification Channel
In Grafana:
1. Click **Alerts** (bell icon)
2. **Notification channels**
3. Add Email, Slack, or PagerDuty
4. Test notification

### Add More Node Exporters

Node Exporter is already deployed as DaemonSet (runs on all nodes automatically).

To verify:
```bash
kubectl get daemonset -n monitoring
kubectl get pods -n monitoring -l app=node-exporter
```

### Add Storage for Grafana

Current setup uses `emptyDir` (temporary storage). For persistent dashboards:

```yaml
# Create PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: grafana-pvc
  namespace: monitoring
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi

---
# Update grafana-deployment.yml volumes section:
# Change:
#   emptyDir: {}
# To:
#   persistentVolumeClaim:
#     claimName: grafana-pvc
```

### Increase Prometheus Retention

In `prometheus-deployment.yml`, update the args:
```yaml
args:
  - '--storage.tsdb.retention.time=30d'  # Change from 15d to 30d
```

Then:
```bash
kubectl apply -f prometheus-deployment.yml
```

---

## 🐛 Troubleshooting

### Grafana Not Starting
```bash
# Check logs
kubectl logs -n monitoring -l app=grafana

# Check resources
kubectl describe pod -n monitoring -l app=grafana

# Restart
kubectl rollout restart deployment/grafana -n monitoring
```

### No Metrics in Dashboard
```bash
# Verify Prometheus has data
curl http://localhost:9090/api/v1/query?query=up

# Check Prometheus targets
http://localhost:9090/targets

# Verify Node Exporter is running
kubectl get pods -n monitoring -l app=node-exporter
```

### Dashboard Missing Panels
```bash
# Verify ConfigMap exists
kubectl get configmap -n monitoring

# Check if dashboard was loaded
kubectl logs -n monitoring -l app=grafana | grep dashboard

# Reload dashboard in Grafana:
1. Click dashboard settings (gear icon)
2. Click "Go to dashboard"
3. Refresh browser (Ctrl+R)
```

---

## 📦 Resource Usage

### CPU & Memory Requests
```
Prometheus:  500m CPU, 512Mi Memory
Grafana:     100m CPU, 128Mi Memory
Node Exporter: 50m CPU per node, 64Mi Memory per node
```

### Storage
```
Prometheus: ~1GB per day of metrics (configurable)
Grafana: 100MB for dashboard data
Retention: 15 days (configurable)
```

---

## 🔐 Security Considerations

### Current Setup (Development)
- Default credentials (admin/admin123)
- No HTTPS
- No authentication for Prometheus

### Production Recommendations
1. **Change Grafana Password**
   ```bash
   kubectl set env deployment/grafana -n monitoring GF_SECURITY_ADMIN_PASSWORD=secure_password
   kubectl rollout restart deployment/grafana -n monitoring
   ```

2. **Enable HTTPS**
   - Use cert-manager for TLS certificates
   - Configure Ingress with TLS

3. **Add RBAC/Authentication**
   - Enable LDAP/OAuth in Grafana
   - Restrict RBAC permissions

4. **Network Policies**
   - Restrict traffic to monitoring namespace
   - Only allow ingress from specific IPs

---

## 📚 Additional Resources

### Prometheus PromQL Queries
See: `PROMETHEUS_METRICS_GUIDE.md`

### Grafana Documentation
See: `GRAFANA_ACCESS_GUIDE.md` and `GRAFANA_DASHBOARD_GUIDE.md`

### Official Docs
- Prometheus: https://prometheus.io/docs/
- Grafana: https://grafana.com/docs/
- Node Exporter: https://github.com/prometheus/node_exporter

---

## 🚦 Next Steps

### Immediate
1. ✅ Deploy all components
2. ✅ Access Grafana dashboard
3. ✅ Verify metrics are flowing

### Short Term (Week 1)
1. Add custom application metrics
2. Set up email alerts
3. Create team-specific dashboards

### Medium Term (Month 1)
1. Configure persistent storage
2. Set up Prometheus remote storage
3. Add integration tests for monitoring

### Long Term (Quarter 1)
1. Implement AlertManager
2. Add Loki for log aggregation
3. Set up SLOs/SLIs monitoring

---

## ❓ FAQ

### Q: Can I add more custom metrics?
**A:** Yes! Any application that exposes Prometheus metrics on a port can be scraped. Add annotations to pod or update prometheus.yml.

### Q: How long does Prometheus store data?
**A:** Default 15 days. Configurable via `--storage.tsdb.retention.time` argument.

### Q: Can I run multiple Prometheus instances?
**A:** Yes, but you'll need a load balancer or use Prometheus HA setup.

### Q: How do I backup dashboards?
**A:** In Grafana, click dashboard settings → Export → Save JSON file.

### Q: Can I use this with EKS Managed Prometheus?
**A:** You can, but this setup is more cost-effective for small clusters.

### Q: What if I need logs too?
**A:** Add Loki (logs) + Promtail (log collector) to the stack.

---

## 📞 Support

For issues:
1. Check logs: `kubectl logs -n monitoring <pod-name>`
2. Check events: `kubectl describe pod -n monitoring <pod-name>`
3. Verify connectivity: `kubectl exec -n monitoring <pod> -- curl <target>`
4. Review guides: See PROMETHEUS_METRICS_GUIDE.md, GRAFANA_*.md files

---

**Last Updated:** 18 December 2025
**Version:** 1.0
**Status:** Production Ready
