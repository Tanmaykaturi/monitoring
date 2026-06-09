# 📊 Prometheus Monitoring with cAdvisor & Blackbox Exporter - Multi-Cluster Setup

## 🚀 Installation Steps

### Step 1: Prepare Environment

Set your kubeconfig and locate your available storage class.
```bash
# Set your kubeconfig
export KUBECONFIG=/path/to/your-cluster.config

# Find your storage class
kubectl get storageclass
# Note the NAME column - you will need this for the values file.
Step 2: Create Monitoring Namespace

kubectl create namespace monitoring
Step 3: Add Prometheus Helm Repository

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
Step 4: Configure prometheus-values.yaml
Update prometheus-values.yaml before installing. Find and replace these values:

Line 13 (cluster): Change to your unique cluster name (e.g., "production-us-east").
Line 19 (storageClassName): Change to the storage class you found in Step 1.
Step 5: Install Prometheus

helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values prometheus-values.yaml
Wait for all pods to reach a Ready state:


kubectl wait --for=condition=ready pod \
  -l app.kubernetes.io/name=prometheus \
  -n monitoring \
  --timeout=300s
Step 6: Deploy cAdvisor

# Deploy cAdvisor DaemonSet
kubectl apply -f cadvisor-daemonset.yaml

# Configure Prometheus to scrape cAdvisor
kubectl apply -f cadvisor-servicemonitor.yaml
Step 7: Configure prometheus-ingress.yaml and Apply
Update prometheus-ingress.yaml before applying.

Line 11 (host): Change to your domain (e.g., prometheus.your-domain.com).

kubectl apply -f prometheus-ingress.yaml
Step 8: Configure DNS (If using a custom domain)
Get your ingress external IP:


kubectl get ingress -n monitoring prometheus-ingress -o wide
Create a DNS record (Type: A or CNAME) pointing your configured domain (prometheus.your-domain.com) to the External IP retrieved above.

🔍 Blackbox Exporter - Endpoint Monitoring
Blackbox Exporter monitors external endpoints (HTTP, HTTPS, ICMP, TCP) to ensure your applications are reachable.

Step 1: Prepare Blackbox Configuration
Copy the template file and customize it for your cluster:


# Copy the template
cp blackbox-exporter-values-template.yaml blackbox-exporter-values-<cluster-name>.yaml

# Example: For production cluster
cp blackbox-exporter-values-template.yaml blackbox-exporter-values-production.yaml
Step 2: Update Placeholders
Edit the new file and replace these placeholders:

Placeholder	Description	Example
<GRAFANA_URL>	Your Grafana URL	https://grafana.monitoring.example.com
<PROMETHEUS_URL>	Your Prometheus URL	http://prometheus.example.com
<CLUSTER_NAME>	Your cluster identifier	production, staging, dev
Quick replace using sed:


sed -i 's|<CLUSTER_NAME>|production|g' blackbox-exporter-values-production.yaml
sed -i 's|<GRAFANA_URL>|https://grafana.example.com|g' blackbox-exporter-values-production.yaml
sed -i 's|<PROMETHEUS_URL>|http://prometheus.example.com|g' blackbox-exporter-values-production.yaml
Step 3: Generate Ingress Targets (Optional)
Auto-generate all ingress endpoints to monitor:


# Make script executable
chmod +x generate-blackbox-ingress.sh

# Generate targets for your cluster
./generate-blackbox-ingress.sh /path/to/kubeconfig.yaml cluster-name

# Save output to file
./generate-blackbox-ingress.sh /path/to/kubeconfig.yaml cluster-name > ingress-targets.yaml
Copy the output and paste it into your values file under the targets: section (after the infrastructure endpoints).

Step 4: Install Blackbox Exporter

helm upgrade --install blackbox-exporter prometheus-community/prometheus-blackbox-exporter \
  --namespace monitoring \
  --values blackbox-exporter-values-<cluster-name>.yaml
Step 5: Verify Deployment
Check if Blackbox Exporter is running:


# Check pod status
kubectl get pods -n monitoring | grep blackbox

# Should show: 1/1 Running

# Check ServiceMonitors
kubectl get servicemonitor -n monitoring | grep blackbox
Step 6: Test Probes
Verify metrics are being collected:


# Port forward to Prometheus
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090 &

# Check if probes are working (in another terminal or browser at http://localhost:9090)
# Run these queries in Prometheus UI:

# 1. See all probes
probe_success{job=~".*blackbox.*"}

# 2. Count total probes
count(probe_success{job=~".*blackbox.*"})

# 3. Check for failed probes
probe_success{job=~".*blackbox.*"} == 0

# 4. Check SSL certificate expiry (days remaining)
(probe_ssl_earliest_cert_expiry{job=~".*blackbox.*"} - time()) / 86400
Step 7: Add Blackbox Alerts to Grafana (Optional)
Import pre-built Blackbox Exporter dashboard:

Open Grafana → Dashboards → Import
Enter Dashboard ID: 7587 (Prometheus Blackbox Exporter)
Select your Prometheus data source
Click Import
Recommended Alerts:

Alert	Query	Threshold
Endpoint Down	probe_success{job=~".*blackbox.*"} == 0	= 0
SSL Cert Expiry Warning	(probe_ssl_earliest_cert_expiry - time()) / 86400 < 30	< 30 days
SSL Cert Expiry Critical	(probe_ssl_earliest_cert_expiry - time()) / 86400 < 7	< 7 days
High Response Time	probe_http_duration_seconds > 5	> 5 seconds
