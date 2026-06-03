# 📊 Prometheus Monitoring with cAdvisor - Multi-Cluster Setup

## 🚀 Installation Steps

### Step 1: Prepare Environment

Set your kubeconfig and locate your available storage class.
```bash
# Set your kubeconfig
export KUBECONFIG=/path/to/your-cluster.config

# Find your storage class
kubectl get storageclass
# Note the NAME column - you will need this for the values file.

```

### Step 2: Create Monitoring Namespace

```bash
kubectl create namespace monitoring

```

### Step 3: Add Prometheus Helm Repository

```bash
helm repo add prometheus-community [https://prometheus-community.github.io/helm-charts](https://prometheus-community.github.io/helm-charts)
helm repo update

```

### Step 4: Configure `prometheus-values.yaml`

Update `prometheus-values.yaml` before installing. Find and replace these values:

* **Line 13 (`cluster`):** Change to your unique cluster name (e.g., `"production-us-east"`).
* **Line 19 (`storageClassName`):** Change to the storage class you found in Step 1.

### Step 5: Install Prometheus

```bash
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values prometheus-values.yaml

```

Wait for all pods to reach a `Ready` state:

```bash
kubectl wait --for=condition=ready pod \
  -l app.kubernetes.io/name=prometheus \
  -n monitoring \
  --timeout=300s

```

### Step 6: Deploy cAdvisor

```bash
# Deploy cAdvisor DaemonSet
kubectl apply -f cadvisor-daemonset.yaml

# Configure Prometheus to scrape cAdvisor
kubectl apply -f cadvisor-servicemonitor.yaml

```

### Step 7: Configure `prometheus-ingress.yaml` and Apply

Update `prometheus-ingress.yaml` before applying.

* **Line 11 (`host`):** Change to your domain (e.g., `prometheus.your-domain.com`).

```bash
kubectl apply -f prometheus-ingress.yaml

```

### Step 8: Configure DNS (If using a custom domain)

Get your ingress external IP:

```bash
kubectl get ingress -n monitoring prometheus-ingress -o wide

```

Create a DNS record (Type: A or CNAME) pointing your configured domain (`prometheus.your-domain.com`) to the External IP retrieved above.

---

## 📈 Connect to Central Grafana

If you have a central Grafana instance in another cluster, follow these steps to connect your new Prometheus setup.

### Step 1: Access Grafana

```bash
# Switch to your Grafana cluster context
export KUBECONFIG=/path/to/grafana-cluster.config

# Port-forward to Grafana locally
kubectl port-forward -n monitoring svc/grafana 3000:80

# Get the admin password
kubectl get secret -n monitoring grafana \
  -o jsonpath="{.data.admin-password}" | base64 --decode
echo

```

1. Open a browser and navigate to `http://localhost:3000`.
2. Login using **Username:** `admin` and the **Password** retrieved from the command above.

### Step 2: Add Prometheus Data Source

1. Click **Configuration** (⚙️) → **Data Sources**.
2. Click **Add data source**.
3. Select **Prometheus**.
4. Configure the following fields:
* **Name:** `Prometheus-<your-cluster-name>`
* **URL:** `http://prometheus.your-domain.com` (or internal service URL).
* **Access:** `Server (default)`.
* **HTTP Method:** `POST`.
* **Scrape interval:** `30s`.


5. Click **Save & Test**. It should show: *"Data source is working"*.

### Step 3: Import Popular Dashboards

1. Click **Dashboards** → **Import**.
2. Import these popular dashboards by entering their IDs:
* **`1860`** (Node Exporter Full - Detailed node metrics)
* **`315`** (Kubernetes Cluster Monitoring - Overall cluster health)
* **`893`** (Kubernetes Monitoring (cAdvisor) - Container metrics)
* **`12006`** (Kubernetes Cluster (Prometheus) - Cluster overview)


3. For each dashboard: Enter the ID, click **Load**, select your Prometheus data source, and click **Import**.

```

```
