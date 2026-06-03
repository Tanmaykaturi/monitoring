# 🌐 Prometheus with Blackbox Exporter Setup

## 🚀 Installation Steps

### Step 1: Update Configuration Files

#### 📄 `prometheus-values.yaml`
Update the following fields with your specific environment details:
* **Line 13:** `cluster` ➡️ `"YOUR-CLUSTER-NAME"`
* **Line 19:** `storageClassName` ➡️ `"YOUR-STORAGE-CLASS"` *(Run `kubectl get storageclass` to find this)*
* **Line 26-28:** `targets` (http) ➡️ `https://myapp.com`
* **Line 43-45:** `targets` (tcp) ➡️ `db.example.com:5432`
* **Line 59-61:** `targets` (icmp) ➡️ `8.8.8.8`


#### 📄 `prometheus-ingress.yaml`
* **Line 13 & 18:** `hosts` / `host` ➡️ Your Prometheus domain (e.g., `prometheus.your-domain.com`)

### Step 2: Set Kubeconfig & Verify
```bash
export KUBECONFIG=/path/to/your/cluster.config
kubectl get nodes

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

### Step 5: Install Prometheus & Blackbox Exporter

```bash
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values prometheus-values.yaml

```

### Step 6: Wait for Pods to Initialize

```bash
# Wait for Prometheus
kubectl wait --for=condition=ready pod \
  -l app.kubernetes.io/name=prometheus \
  -n monitoring \
  --timeout=300s

# Wait for Blackbox Exporter
kubectl wait --for=condition=ready pod \
  -l app.kubernetes.io/name=prometheus-blackbox-exporter \
  -n monitoring \
  --timeout=300s

```

### Step 7: Create SSL Certificate Issuer

```bash
kubectl apply -f cert-issuer.yaml
kubectl get issuer -n monitoring

```

### Step 8: Apply Ingress Route

```bash
kubectl apply -f prometheus-ingress.yaml
kubectl get certificate -n monitoring

```

### Step 9: Configure DNS Settings

Retrieve your ingress IP address:

```bash
kubectl get ingress -n monitoring

```

In your domain registrar, create an **A Record** or **CNAME** pointing `prometheus.your-domain.com` to the retrieved Ingress IP.

### Step 10: Verify the Installation

Allow 2-5 minutes for DNS propagation and SSL certificate issuance.

```bash
# Test Prometheus locally via Port-Forwarding
kubectl port-forward -n monitoring svc/prometheus-operated 9090:9090

# Test Blackbox Exporter locally via Port-Forwarding
kubectl port-forward -n monitoring svc/prometheus-blackbox-exporter 9115:9115

```

Once DNS propagates, you can access your secure Prometheus UI at `https://prometheus.your-domain.com`.

```

```
