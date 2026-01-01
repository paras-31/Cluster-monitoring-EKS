# Kubernetes Cluster Monitoring with Grafana & Prometheus

Complete EKS cluster monitoring solution with Grafana dashboards, Prometheus metrics collection, and LoadBalancer-based internet access.

## 📋 Overview

This setup provides comprehensive monitoring of your EKS cluster with:
- **Real-time metrics** from Prometheus
- **Interactive dashboards** in Grafana
- **Internet-facing access** via AWS LoadBalancer
- **Clean, maintainable architecture** with separated configuration files

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   EKS Cluster                            │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  ┌──────────────┐    ┌──────────────┐                  │
│  │  Prometheus  │    │   Grafana    │                  │
│  │  (Metrics)   │←→──│ (Dashboards) │                  │
│  └──────────────┘    └──────────────┘                  │
│         ↑                    ↓                           │
│         │            AWS LoadBalancer                   │
│    Scrapes:         (Port 80)                           │
│    - Nodes          ↓                                    │
│    - Pods      Internet Access                          │
│    - kube-state └─────────────────────────────────────┘
│    - node-exporter
│
└─────────────────────────────────────────────────────────┘
```

## 📁 File Structure

```
advanced-grafana-dashboard/
├── README.md                          # This documentation
├── grafana-deployment.yml             # RBAC, Service, Grafana Deployment
├── grafana-configmaps.yml             # Dashboard, Datasource, Provisioning configs
├── prometheus-deployment.yml           # Prometheus time-series DB
├── node-exporter-deployment.yml        # Node CPU/Memory/Disk metrics
└── kube-state-metrics-deployment.yml  # Kubernetes cluster state metrics
```

## 🔧 What Was Done

### 1. **Separated Configuration Files** ⭐

**Before**: Everything in one large file  
**After**: Split into two files for better management

#### **grafana-configmaps.yml** - Contains:
- `grafana-datasources` ConfigMap → Prometheus connection details
- `grafana-dashboards-provider` ConfigMap → Dashboard auto-provisioning config
- `grafana-dashboard-kubernetes` ConfigMap → Full dashboard JSON definition

#### **grafana-deployment.yml** - Contains:
- ServiceAccount → Grafana identity
- ClusterRole → Permissions to read ConfigMaps
- ClusterRoleBinding → Grant permissions
- Service → LoadBalancer for internet access
- Deployment → Grafana pods

**Benefits**:
✅ Update dashboards without pod restart  
✅ Easier to version control  
✅ Cleaner, more maintainable structure  
✅ Reusable configuration patterns  

### 2. **Created Comprehensive Dashboard with 11 Panels**

#### **Panel 1-5: Cluster Overview (Stats) - Top Row**
| Panel | Shows | Example |
|-------|-------|---------|
| Total Nodes | Worker node count | 2 |
| Total Pods | Running pods cluster-wide | 35 |
| Total Namespaces | K8s namespaces | 5 |
| Cluster CPU Usage | Total cores in use (all nodes) | 2.67 cores |
| Cluster Memory Usage | Total RAM in use (all nodes) | 6.63 GB |

#### **Panel 6-7: Node Metrics (Line Graphs)**
- **Node CPU Usage %**: CPU utilization trend per node (visual)
- **Node Memory Usage %**: Memory utilization trend per node (visual)

Shows two lines: one for each worker node

#### **Panel 8-11: Resource Allocation (Tables)**
Shows what was **allocated** in deployment YAML:

| Panel | Shows | Example |
|-------|-------|---------|
| CPU Requests | CPU min guaranteed per pod | 0.1 cores |
| CPU Limits | CPU max allowed per pod | 0.2 cores |
| Memory Requests | Memory min guaranteed (MB) | 128 MB |
| Memory Limits | Memory max allowed (MB) | 256 MB |

#### **Additional Panels: Pod Operations**
- **Pod Restart Count**: Tracks pod crashes/restarts
- **Pod Status**: Shows Running/Pending/Failed state
- **Pod Ready Status**: Container readiness indicator
- **Network In/Out**: Network traffic per interface

### 3. **Prometheus Datasource Configuration**

**File Location**: `grafana-configmaps.yml` → `grafana-datasources` ConfigMap

```yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    url: http://prometheus-service.monitoring.svc.cluster.local:80
    isDefault: true
    jsonData:
      timeInterval: "15s"
```

**Why Kubernetes DNS (`prometheus-service.monitoring.svc.cluster.local`) instead of IP?**

| Aspect | DNS | IP |
|--------|-----|-----|
| **Stability** | Works if pod restarts | Breaks if pod IP changes |
| **Service Discovery** | Auto-finds available pods | Manual IP management |
| **Best Practice** | Kubernetes standard | ❌ Not recommended |
| **Resilience** | Built-in load balancing | Manual balancing needed |

### 4. **Dashboard Provisioning Setup**

**File Location**: `grafana-configmaps.yml` → `grafana-dashboards-provider` ConfigMap

```yaml
apiVersion: 1
providers:
  - name: 'Default'
    orgId: 1
    type: file
    options:
      path: /var/lib/grafana/dashboards
```

**How It Works**:

```
1. Grafana starts
   ↓
2. Reads provisioning config
   ↓
3. Looks for JSON files in /var/lib/grafana/dashboards
   ↓
4. Automatically imports all dashboards found
   ↓
5. Updates when ConfigMap changes (hot reload)
```

**Benefits**:
- No manual dashboard import clicks
- Infrastructure as Code (IaC)
- Easy to update and version
- Automatic deployment

### 5. **Dashboard JSON Configuration**

**File Location**: `grafana-configmaps.yml` → `grafana-dashboard-kubernetes` ConfigMap

The JSON defines:

```json
{
  "title": "Kubernetes Cluster Monitoring",  // Dashboard name
  "uid": "kubernetes-cluster",               // Unique ID
  "refresh": "30s",                          // Auto-refresh interval
  "panels": [
    {
      "title": "Total Nodes",
      "type": "stat",                        // Stat = big number
      "targets": [
        {
          "expr": "count(kube_node_info)",   // PromQL query
          "legendFormat": "Nodes"
        }
      ]
    },
    // ... more panels ...
  ]
}
```

**Panel Types Used**:
- `stat` → Large numbers (Total Nodes, Total Pods, etc.)
- `graph` → Line charts (CPU/Memory trends)
- `table` → Tabular data (Requests/Limits/Restarts)

### 6. **ConfigMap Importance** 🎯

**What is a ConfigMap?**
Kubernetes object to store non-sensitive configuration data.

**Why ConfigMaps for Dashboards?**

| Benefit | Explanation |
|---------|-------------|
| **Decoupling** | Change dashboards without container rebuild |
| **Hot Reload** | Update configs without pod restart (most cases) |
| **Version Control** | Track dashboard changes in git |
| **Reusability** | Share configs across environments |
| **Flexibility** | Easy to update queries and panels |
| **Non-sensitive** | Dashboard JSON is not sensitive (no secrets) |

**Alternative (Secrets)** would be used for:
- Database passwords
- API keys
- Certificates
- OAuth tokens

### 7. **Grafana Setup Process** 🔐

#### **A. RBAC (Role-Based Access Control)**

```yaml
---
# Identity for Grafana
apiVersion: v1
kind: ServiceAccount
metadata:
  name: grafana
  namespace: monitoring

---
# Permissions Grafana needs
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: grafana
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch"]  # Read ConfigMaps

---
# Grant permissions to Grafana
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: grafana
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: grafana
subjects:
  - kind: ServiceAccount
    name: grafana
    namespace: monitoring
```

**Why?** Grafana needs permission to read ConfigMaps for provisioning dashboards.

#### **B. Environment Variables**

```yaml
env:
  - name: GF_SECURITY_ADMIN_USER
    value: "admin"
  - name: GF_SECURITY_ADMIN_PASSWORD
    value: "admin123"
  - name: GF_SERVER_ROOT_URL
    value: "http://k8s-monitori-grafanas-2a92f98a2e-7f5015763c8b45b2.elb.us-east-1.amazonaws.com"
  - name: GF_INSTALL_PLUGINS
    value: "grafana-clock-panel"
```

| Variable | Purpose |
|----------|---------|
| `ADMIN_USER` | Login username |
| `ADMIN_PASSWORD` | Login password |
| `ROOT_URL` | External access URL (LoadBalancer DNS) |
| `INSTALL_PLUGINS` | Additional plugins |

#### **C. Volume Mounts**

```yaml
volumeMounts:
  - name: grafana-storage
    mountPath: /var/lib/grafana          # Grafana data
  - name: datasource-config
    mountPath: /etc/grafana/provisioning/datasources
  - name: dashboard-provider
    mountPath: /etc/grafana/provisioning/dashboards
  - name: dashboard-config
    mountPath: /var/lib/grafana/dashboards
```

**What Each Contains**:
- `grafana-storage` → Grafana database, user preferences (emptyDir = lost on pod restart)
- `datasource-config` → Prometheus connection details
- `dashboard-provider` → Where to find dashboards
- `dashboard-config` → Actual dashboard JSON files

#### **D. Resource Allocation**

```yaml
resources:
  requests:
    memory: "128Mi"
    cpu: "100m"
  limits:
    memory: "256Mi"
    cpu: "200m"
```

Keeps Grafana lightweight and prevents resource hogging.

## 🌐 LoadBalancer Access Explained

### **What is a LoadBalancer Service?**

In Kubernetes, a `Service` of type `LoadBalancer` exposes your application to the internet by leveraging cloud provider load balancers.

### **How Grafana LoadBalancer Works**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: grafana-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
spec:
  type: LoadBalancer
  selector:
    app: grafana
  ports:
    - port: 80              # External port (internet)
      targetPort: 3000      # Container port (Grafana)
```

### **Access Flow**

```
┌─ User Browser ──┐
│ http://DNS:80   │
└────────┬────────┘
         ↓
    [AWS Network Load Balancer]
    (Manages port 80 publicly)
         ↓
    [Kubernetes Service - Port 80]
    (Routes to pod port 3000)
         ↓
    [Grafana Pod - Port 3000]
    (Running Grafana application)
         ↓
    [Dashboard UI]
```

### **Why `internet-facing` Annotation?**

```yaml
annotations:
  service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
```

| Option | What it Does | Use Case |
|--------|--------------|----------|
| `internet-facing` | Public IP, accessible from internet | ✅ User dashboards |
| `internal` | VPC-only access, no internet | Internal services |

## 🚀 Accessing the Dashboard

### **Step 1: Get LoadBalancer DNS Name**
```bash
kubectl get svc grafana-service -n monitoring \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

**Output** (AWS NLB DNS):
```
k8s-monitori-grafanas-2a92f98a2e-7f5015763c8b45b2.elb.us-east-1.amazonaws.com
```

### **Step 2: Open in Browser**
```
http://k8s-monitori-grafanas-2a92f98a2e-7f5015763c8b45b2.elb.us-east-1.amazonaws.com
```

**Note**: Takes 1-2 minutes for AWS to provision the LoadBalancer initially

### **Step 3: Login**
| Field | Value |
|-------|-------|
| Username | `admin` |
| Password | `admin123` |

⚠️ **Change password in production!**

### **Step 4: View Dashboard**
Navigate to: **Dashboards** → **Kubernetes Cluster Monitoring**

## 📸 Dashboard Preview

Here's what the Kubernetes Cluster Monitoring dashboard looks like:

![Grafana Dashboard](./dashboard-screenshot.png)

**Dashboard Features Visible**:
- **Top Row Stats**: Total Nodes (2), Total Pods (35), Total Namespaces, Cluster CPU Usage (2.63 cores), Cluster Memory Usage (6.65 GB)
- **Node CPU Usage Graph**: Shows CPU utilization trends for both worker nodes over time
- **Node Memory Usage Graph**: Shows memory utilization (around 70%) per node
- **CPU Requests & Limits Tables**: Shows actual CPU allocations for pods in all namespaces
- **Clean, Dark Theme**: Easy on the eyes for long monitoring sessions



## 📊 Understanding Dashboard Metrics

### **CPU Requests vs CPU Limits**

```yaml
resources:
  requests:
    cpu: 100m      # 0.1 cores guaranteed
  limits:
    cpu: 200m      # Max 0.2 cores allowed
```

| Metric | Meaning | Example |
|--------|---------|---------|
| **CPU Request** | Minimum CPU guaranteed by K8s scheduler | If set to 100m, scheduler ensures this CPU is available |
| **CPU Limit** | Maximum CPU pod can use | If exceeded, pod gets throttled |
| **CPU Usage** | Actual CPU being used right now | Seen in `kubectl top pod` |

**In Dashboard**: 
- CPU shows in **cores** (100m = 0.1 cores)
- Memory shows in **MB** (converted from bytes)

### **Example Interpretation**

From the dashboard, if you see:
```
Pod: kafka
Memory Request: 600 MB  ← K8s guarantees 600MB available
Memory Limit: 600 MB    ← Pod cannot exceed 600MB
```

This means Kafka will get exactly 600MB reserved.

**Why Default Namespace Shows Only Memory?**

Default namespace pods only have memory configured:
```bash
kubectl get pods -n default -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].resources}{"\n"}{end}'
```

Output shows memory but no CPU:
```
kafka  {"memory":"600Mi"}  ← No CPU request!
```

To add CPU, edit the deployment:
```yaml
resources:
  requests:
    cpu: 250m        # Add this
    memory: 600Mi
  limits:
    cpu: 500m        # Add this
    memory: 600Mi
```

## 🔄 Deployment Commands

### **Initial Deployment (All Components)**
```bash
# 1. Deploy ConfigMaps (dashboards, datasources)
kubectl apply -f grafana-configmaps.yml

# 2. Deploy Grafana (RBAC, Service, Deployment)
kubectl apply -f grafana-deployment.yml

# 3. Deploy Prometheus
kubectl apply -f prometheus-deployment.yml

# 4. Deploy Node Exporter (node metrics)
kubectl apply -f node-exporter-deployment.yml

# 5. Deploy kube-state-metrics
kubectl apply -f kube-state-metrics-deployment.yml
```

### **Update Dashboard Only (No Pod Restart)**
```bash
# Edit the JSON in grafana-configmaps.yml
# Then apply:
kubectl apply -f grafana-configmaps.yml

# Grafana hot-reloads the dashboard!
```

### **Restart Grafana Pods**
```bash
kubectl rollout restart deployment/grafana -n monitoring

# Watch the rollout
kubectl rollout status deployment/grafana -n monitoring
```

## 🛠️ Troubleshooting

### **Dashboard Shows "No Data"**
```bash
# Check Prometheus is scraping
kubectl port-forward -n monitoring svc/prometheus-service 9090:80
# Visit: http://localhost:9090/targets
# All targets should show "UP"
```

### **LoadBalancer Shows "Pending"**
```bash
# AWS NLB takes 1-2 minutes to provision
kubectl get svc grafana-service -n monitoring -w

# When ready, you'll see EXTERNAL-IP populated
```

### **Cannot Access via LoadBalancer DNS**
```bash
# 1. Check service is internet-facing
kubectl get svc grafana-service -n monitoring -o yaml | grep load-balancer-scheme

# 2. Check AWS security groups allow port 80
# 3. Verify pod is running
kubectl get pods -n monitoring -l app=grafana
```

### **Grafana Pod Crashes**
```bash
# Check logs
kubectl logs -n monitoring -l app=grafana --tail=100

# Common issues:
# - ConfigMap not mounted correctly
# - Memory/CPU limits too low
# - Datasource unreachable
```

## 📝 Summary of What Was Implemented

✅ **Prometheus** - Metrics collection (scrapes all targets)  
✅ **Grafana** - Dashboard visualization  
✅ **node-exporter** - Node CPU/Memory/Disk metrics  
✅ **kube-state-metrics** - Kubernetes state metrics  
✅ **LoadBalancer** - Internet-facing access (no port-forward!)  
✅ **ConfigMaps** - Dashboard as code (versioning support)  
✅ **RBAC** - Proper permissions for Grafana  
✅ **Auto-provisioning** - Dashboards load automatically  
✅ **Clean Architecture** - Separated config files  

## 🔒 Security Recommendations

1. **Change Default Password**
   ```bash
   # In Grafana UI: Admin Panel → Change Password
   ```

2. **Use HTTPS**
   ```bash
   # Add TLS cert to LoadBalancer
   # Or use AWS ACM certificate
   ```

3. **Restrict Access**
   ```bash
   # Add AWS security group rules
   # Only allow your IP ranges
   ```

4. **Use Secrets for Sensitive Data**
   ```bash
   # Database passwords
   # API keys
   # OAuth tokens
   # Use Kubernetes Secrets, not ConfigMaps
   ```

## 📚 Key Technologies

| Component | Version | Purpose |
|-----------|---------|---------|
| Prometheus | 3.8.1 | Time-series metrics database |
| Grafana | 10.2.0 | Dashboard & visualization |
| node-exporter | 1.6.1 | Node OS metrics |
| kube-state-metrics | 2.9.2 | Kubernetes state metrics |
| AWS NLB | - | Network Load Balancer |

---

**Last Updated**: January 1, 2026  
**Status**: ✅ Production Ready  
**Maintainer**: Devops Team

  - Running vs Desired Replicas

## Quick Start

### Deploy Everything
```bash
cd advanced-grafana-dashboard
kubectl apply -f prometheus-deployment.yml
kubectl apply -f kube-state-metrics-deployment.yml
kubectl apply -f grafana-deployment.yml
kubectl apply -f grafana-dashboard-comprehensive.yml
```

### Access Grafana
```bash
# Get LoadBalancer URL
kubectl get svc -n monitoring grafana-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

Then open in browser: `http://<LoadBalancer-DNS>`
- Username: `admin`
- Password: `admin123`

### Access Prometheus
```bash
# Get LoadBalancer URL
kubectl get svc -n monitoring prometheus-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

Then open: `http://<LoadBalancer-DNS>`

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    EKS Cluster                          │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌──────────────────────────────────────────────────┐  │
│  │          monitoring namespace                     │  │
│  │                                                  │  │
│  │  ┌─────────────────┐  ┌──────────────────────┐  │  │
│  │  │   Prometheus    │  │  kube-state-metrics  │  │  │
│  │  │   (TSDB)        │  │  (Cluster state)     │  │  │
│  │  │ Port: 9090      │  │  Port: 8080          │  │  │
│  │  └────────┬────────┘  └──────────┬───────────┘  │  │
│  │           │                      │              │  │
│  │           └──────────┬───────────┘              │  │
│  │                      │ Scrapes                  │  │
│  │           ┌──────────▼──────────┐              │  │
│  │           │ Node Exporter (x2)  │              │  │
│  │           │ DaemonSet           │              │  │
│  │           └─────────────────────┘              │  │
│  │                                                  │  │
│  │  ┌──────────────────────────────────────────┐  │  │
│  │  │          Grafana                          │  │  │
│  │  │      (Visualization)                      │  │  │
│  │  │      Port: 3000 (via port 80 ELB)        │  │  │
│  │  └──────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────┘  │
│                                                         │
└─────────────────────────────────────────────────────────┘
         │                                    │
         │ AWS ELB (LoadBalancer)              │
         │                                    │
    ┌────▼────────────────────────────────────▼────┐
    │        External Internet                      │
    │  (Accessible via DNS)                        │
    └───────────────────────────────────────────────┘
```

## Metrics Available

### Node Metrics (from Node Exporter)
- `node_cpu_seconds_total` - CPU usage
- `node_memory_MemTotal_bytes` - Total memory
- `node_memory_MemAvailable_bytes` - Available memory
- `node_load1`, `node_load5`, `node_load15` - Load average
- `node_filesystem_*` - Disk usage

### Cluster Metrics (from kube-state-metrics)
- `kube_node_labels` - Node information
- `kube_pod_info` - Pod information
- `kube_pod_status_phase` - Pod status (Running, Pending, Failed, etc.)
- `kube_deployment_*` - Deployment state
- `kube_namespace_labels` - Namespace info
- `kube_pod_container_resource_requests` - Resource requests
- `kube_pod_container_resource_limits` - Resource limits
- `kube_pod_container_status_restarts_total` - Container restarts

## Troubleshooting

### Dashboard Shows "No data"
1. **Wait 2-3 minutes** - Prometheus needs time to scrape metrics
2. **Refresh browser** - F5 or Ctrl+R
3. **Check Prometheus**:
   ```bash
   kubectl port-forward -n monitoring svc/prometheus-service 9090:80
   # Go to http://localhost:9090
   # Search for: kube_pod_info, node_cpu_seconds_total
   ```

### Datasource Connection Error (504)
1. Check datasource URL is correct:
   ```bash
   kubectl get configmap -n monitoring grafana-datasources -o yaml | grep url
   ```
   Should be: `http://prometheus-service.monitoring.svc.cluster.local:80`

2. Restart Grafana:
   ```bash
   kubectl rollout restart deployment/grafana -n monitoring
   ```

### kube-state-metrics Not Collecting
1. Check pod is running:
   ```bash
   kubectl get pods -n monitoring -l app=kube-state-metrics
   ```

2. Check logs for RBAC errors:
   ```bash
   kubectl logs -n monitoring -l app=kube-state-metrics | grep -i error
   ```

3. Verify it can list resources:
   ```bash
   kubectl exec -it -n monitoring $(kubectl get pod -n monitoring -l app=kube-state-metrics -o jsonpath='{.items[0].metadata.name}') -- wget -q -O- http://localhost:8080/metrics | head -20
   ```

## Performance Tuning

### For Large Clusters (>100 nodes)
In `prometheus-deployment.yml`, adjust:
```yaml
scrape_interval: 30s  # Increase from 15s
evaluation_interval: 30s  # Increase from 15s
```

### Reduce Memory Usage
In `prometheus-deployment.yml`:
```yaml
retention: 7d  # Reduce from 15d
```

In `grafana-deployment.yml`:
```yaml
limits:
  memory: "128Mi"  # Reduce from 256Mi
```

## Security

### Production Hardening

1. **Change Grafana Password**:
   - Log in to Grafana
   - Profile → Change password
   - Use strong password

2. **Enable HTTPS**:
   ```bash
   # Create certificate
   kubectl create secret tls grafana-tls \
     --cert=path/to/cert.crt \
     --key=path/to/key.key \
     -n monitoring
   ```

3. **Restrict Access**:
   - Add AWS Security Group ingress rules
   - Or use Kubernetes NetworkPolicy

4. **Enable Authentication**:
   - In `grafana-deployment.yml`, add OAuth/LDAP configuration

## Cleanup

```bash
# Delete all monitoring resources
kubectl delete namespace monitoring

# Or delete individual components
kubectl delete -f prometheus-deployment.yml
kubectl delete -f grafana-deployment.yml
kubectl delete -f kube-state-metrics-deployment.yml
kubectl delete -f grafana-dashboard-comprehensive.yml
```

## Next Steps

1. **Customize Dashboard** - Edit dashboard in Grafana UI
2. **Add Alerts** - Configure alert rules in Prometheus
3. **Add More Dashboards** - Import from Grafana dashboard library
4. **Set Up Persistence** - Replace emptyDir with PVC for Prometheus/Grafana data
5. **Enable RBAC** - Restrict Grafana user permissions

## Files Reference

| File | Purpose | Key Component |
|------|---------|---|
| `prometheus-deployment.yml` | Metrics collection | Prometheus + Node Exporter |
| `grafana-deployment.yml` | Visualization | Grafana + Datasource config |
| `kube-state-metrics-deployment.yml` | Cluster state | kube-state-metrics |
| `grafana-dashboard-comprehensive.yml` | Pre-built dashboard | 13-panel comprehensive dashboard |

---

**Version**: 1.0  
**Last Updated**: December 19, 2025  
**Status**: Production Ready ✅
