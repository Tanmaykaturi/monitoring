Prometheus Monitoring Setup
Quick installation guide for deploying Prometheus with SSL certificate management.

Prerequisites
kubectl and helm installed
Kubeconfig access to target cluster
Nginx Ingress Controller installed
cert-manager installed
Domain name for Prometheus
Installation Steps
Step 1: Configure Values
Update the following files with your cluster-specific values:

prometheus-values.yaml

Line 12: cluster: "YOUR-CLUSTER-NAME"
Line 18: storageClassName: "YOUR-STORAGE-CLASS"
To find your storage class:


kubectl get storageclass
cert-issuer.yaml

Line 8: email: your-email@example.com
prometheus-ingress.yaml

Line 13: - prometheus.YOUR-DOMAIN.com
Line 18: - host: prometheus.YOUR-DOMAIN.com
Step 2: Set Kubeconfig

export KUBECONFIG=/path/to/your/cluster.config
kubectl get nodes  # Verify connection
Step 3: Create Namespace

kubectl create namespace monitoring
Step 4: Add Helm Repository

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
Step 5: Install Prometheus

helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values prometheus-values.yaml
Step 6: Wait for Pods

# Check status
kubectl get pods -n monitoring

# Wait for ready state
kubectl wait --for=condition=ready pod \
  -l app.kubernetes.io/name=prometheus \
  -n monitoring \
  --timeout=300s
Step 7: Create SSL Certificate Issuer

kubectl apply -f cert-issuer.yaml

# Verify issuer
kubectl get issuer -n monitoring
Step 8: Apply Ingress

kubectl apply -f prometheus-ingress.yaml

# Check certificate status
kubectl get certificate -n monitoring
Step 9: Configure DNS
Get the ingress IP/hostname:


kubectl get ingress -n monitoring prometheus-ingress
Create DNS record:

Type: A or CNAME
Host: prometheus.your-domain.com
Value: [IP/hostname from above]
Step 10: Verify Installation
Wait 2-5 minutes for DNS propagation and SSL certificate issuance.


# Check certificate is ready
kubectl get certificate -n monitoring

# Access Prometheus
# https://prometheus.your-domain.com
Or test locally:


kubectl port-forward -n monitoring svc/prometheus-operated 9090:9090
# Access: http://localhost:9090
Add to Grafana
Step 1: Access Grafana

# Switch to monitoring cluster
export KUBECONFIG=/path/to/monitoring-cluster.config

# Port-forward to Grafana
kubectl port-forward -n monitoring svc/grafana 3000:80

# Get admin password
kubectl get secret -n monitoring grafana \
  -o jsonpath="{.data.admin-password}" | base64 --decode
Step 2: Add Data Source
Login to Grafana at http://localhost:3000
Go to Configuration → Data Sources
Click Add data source → Select Prometheus
Configure:
Name: Prometheus-<cluster-name>
URL: https://prometheus.your-domain.com
HTTP Method: POST
Click Save & Test
Verify Metrics Collection
In Prometheus UI:

Go to Status → Targets
All targets should show as "UP"
In Grafana:

Go to Explore
Select your Prometheus datasource
Query: up
You should see metrics
Useful Commands

# Check all resources
kubectl get pods,svc,ingress,certificate -n monitoring

# View Prometheus logs
kubectl logs -n monitoring prometheus-prometheus-kube-prometheus-prometheus-0

# Check certificate details
kubectl describe certificate -n monitoring prometheus-tls-cert

# Restart ingress if needed
kubectl rollout restart deployment -n ingress-nginx ingress-nginx-controller
Troubleshooting
Certificate not issuing

kubectl describe certificate -n monitoring prometheus-tls-cert
kubectl get certificaterequest -n monitoring
kubectl logs -n cert-manager -l app=cert-manager
Ingress not working

kubectl describe ingress -n monitoring prometheus-ingress
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx
Pods not starting

kubectl describe pod -n monitoring prometheus-prometheus-kube-prometheus-prometheus-0
kubectl get events -n monitoring --sort-by='.lastTimestamp'
Upgrade Configuration
After changing values:


helm upgrade prometheus prometheus-community/kube-prometheus-stack \
  -n monitoring \
  --values prometheus-values.yaml
Uninstall

helm uninstall prometheus -n monitoring
kubectl delete issuer letsencrypt-monitoring -n monitoring
kubectl delete namespace monitoring
