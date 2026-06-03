# 📊 Prometheus Monitoring Setup

A quick and comprehensive installation guide for deploying Prometheus with automated SSL certificate management on Kubernetes.

---

## 📌 Prerequisites

Before you begin, ensure you have the following ready:
* **CLI Tools:** `kubectl` and `helm` installed locally.
* **Access:** `kubeconfig` access to your target Kubernetes cluster.
* **Ingress:** Nginx Ingress Controller installed and running.
* **Certificates:** `cert-manager` installed for handling SSL.
* **DNS:** A valid domain name designated for your Prometheus instance.

---

## 🚀 Installation Steps

### Step 1: Configure Values
Update the following configuration files with your cluster-specific values before deployment.

**`prometheus-values.yaml`**
* **Line 12:** `cluster: "YOUR-CLUSTER-NAME"`
* **Line 18:** `storageClassName: "YOUR-STORAGE-CLASS"`
> 💡 *To find your available storage classes, run:* `kubectl get storageclass`

**`cert-issuer.yaml`**
* **Line 8:** `email: your-email@example.com`

**`prometheus-ingress.yaml`**
* **Line 13:** `- prometheus.YOUR-DOMAIN.com`
* **Line 18:** `- host: prometheus.YOUR-DOMAIN.com`

### Step 2: Set Kubeconfig & Verify
```bash
export KUBECONFIG=/path/to/your/cluster.config
kubectl get nodes  # Verify connection

```

### Step 3: Create Namespace

```bash
kubectl create namespace monitoring

```

### Step 4: Add Helm Repository

```bash
helm repo add prometheus-community [https://prometheus-community.github.io/helm-charts](https://prometheus-community.github.io/helm-charts)
helm repo update

```

### Step 5: Install Prometheus

```bash
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values prometheus-values.yaml

```

### Step 6: Wait for Pods

```bash
# Check status
kubectl get pods -n monitoring

# Wait for ready state
kubectl wait --for=condition=ready pod \
  -l app.kubernetes.io/name=prometheus \
  -n monitoring \
  --timeout=300s

```

### Step 7: Create SSL Certificate Issuer

```bash
kubectl apply -f cert-issuer.yaml

# Verify issuer
kubectl get issuer -n monitoring

```

### Step 8: Apply Ingress

```bash
kubectl apply -f prometheus-ingress.yaml

# Check certificate status
kubectl get certificate -n monitoring

```

### Step 9: Configure DNS

First, retrieve your ingress IP or hostname:

```bash
kubectl get ingress -n monitoring prometheus-ingress

```

Next, create a DNS record with your domain provider:

* **Type:** `A` or `CNAME`
* **Host:** `prometheus.your-domain.com`
* **Value:** `[IP/hostname from the command above]`

### Step 10: Verify Installation

Wait 2-5 minutes for DNS propagation and SSL certificate issuance.

```bash
# Check if the certificate is ready
kubectl get certificate -n monitoring

```

* **Access Prometheus UI:** `https://prometheus.your-domain.com`
* **Test Locally (Optional):**

```bash
  kubectl port-forward -n monitoring svc/prometheus-operated 9090:9090
  # Access at: http://localhost:9090

```

---

## 📈 Add to Grafana

### Step 1: Access Grafana

```bash
# Switch to monitoring cluster context
export KUBECONFIG=/path/to/monitoring-cluster.config

# Port-forward to Grafana
kubectl port-forward -n monitoring svc/grafana 3000:80

# Get admin password
kubectl get secret -n monitoring grafana \
  -o jsonpath="{.data.admin-password}" | base64 --decode

```

### Step 2: Add Data Source

1. Login to Grafana at `http://localhost:3000`
2. Navigate to **Configuration** → **Data Sources**
3. Click **Add data source** → Select **Prometheus**
4. Configure the following fields:
* **Name:** `Prometheus-<cluster-name>`
* **URL:** `https://prometheus.your-domain.com`
* **HTTP Method:** `POST`


5. Click **Save & Test**

### Step 3: Verify Metrics Collection

* **In Prometheus UI:** Go to **Status** → **Targets**. All targets should show as **"UP"**.
* **In Grafana:** Go to **Explore**, select your new Prometheus datasource, and query `up` to verify metrics are flowing.

---

## 🛠️ Useful Commands

| Action | Command |
| --- | --- |
| **Check all resources** | `kubectl get pods,svc,ingress,certificate -n monitoring` |
| **View Prometheus logs** | `kubectl logs -n monitoring prometheus-prometheus-kube-prometheus-prometheus-0` |
| **Check certificate details** | `kubectl describe certificate -n monitoring prometheus-tls-cert` |
| **Restart ingress** | `kubectl rollout restart deployment -n ingress-nginx ingress-nginx-controller` |

---

## ⚠️ Troubleshooting

**Certificate not issuing**

```bash
kubectl describe certificate -n monitoring prometheus-tls-cert
kubectl get certificaterequest -n monitoring
kubectl logs -n cert-manager -l app=cert-manager

```

**Ingress not working**

```bash
kubectl describe ingress -n monitoring prometheus-ingress
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx

```

**Pods not starting**

```bash
kubectl describe pod -n monitoring prometheus-prometheus-kube-prometheus-prometheus-0
kubectl get events -n monitoring --sort-by='.lastTimestamp'

```

---

## 🔄 Lifecycle Management

### Upgrade Configuration

Run this after making changes to your `prometheus-values.yaml`:

```bash
helm upgrade prometheus prometheus-community/kube-prometheus-stack \
  -n monitoring \
  --values prometheus-values.yaml

```

### Uninstall

To completely remove the installation:

```bash
helm uninstall prometheus -n monitoring
kubectl delete issuer letsencrypt-monitoring -n monitoring
kubectl delete namespace monitoring

```

```

```
