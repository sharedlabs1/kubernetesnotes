# Deploying Prometheus and Grafana in Kubernetes

This guide will walk you through deploying Prometheus for monitoring and Grafana for visualization in a Kubernetes cluster.

## Prerequisites

- A running Kubernetes cluster
- `kubectl` configured to communicate with your cluster
- Helm 3 installed (optional, but recommended)

## Step 1: Create a Monitoring Namespace

First, create a dedicated namespace for your monitoring stack:

```bash
kubectl create namespace monitoring
```

## Step 2: Deploy Prometheus

We'll use Helm to deploy Prometheus, as it simplifies the process significantly.

1. Add the Prometheus community Helm repository:

   ```bash
   helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
   helm repo update
   ```

2. Install Prometheus:

   ```bash
   helm install prometheus prometheus-community/prometheus \
     --namespace monitoring \
     --set alertmanager.persistentVolume.storageClass="gp2" \
     --set server.persistentVolume.storageClass="gp2"
   ```

   Note: Replace "gp2" with your preferred storage class.

3. Verify the deployment:

   ```bash
   kubectl get pods -n monitoring
   ```

## Step 3: Deploy Grafana

Now, let's deploy Grafana using Helm:

1. Add the Grafana Helm repository:

   ```bash
   helm repo add grafana https://grafana.github.io/helm-charts
   helm repo update
   ```

2. Install Grafana:

   ```bash
   helm install grafana grafana/grafana \
     --namespace monitoring \
     --set persistence.storageClassName="gp2" \
     --set persistence.enabled=true \
     --set adminPassword='YOUR_ADMIN_PASSWORD' \
     --values - <<EOF
   datasources:
     datasources.yaml:
       apiVersion: 1
       datasources:
       - name: Prometheus
         type: prometheus
         url: http://prometheus-server
         access: proxy
         isDefault: true
   EOF
   ```

   Replace 'YOUR_ADMIN_PASSWORD' with a secure password.

3. Verify the deployment:

   ```bash
   kubectl get pods -n monitoring
   ```

## Step 4: Access Grafana

1. Get the Grafana admin password:

   ```bash
   kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
   ```

2. Port-forward the Grafana service:

   ```bash
   kubectl port-forward -n monitoring service/grafana 3000:80
   ```

3. Access Grafana by opening a web browser and navigating to `http://localhost:3000`.

## Step 5: Configure Grafana

1. Log in to Grafana using the username "admin" and the password you retrieved earlier.

2. Go to "Configuration" > "Data Sources".

3. You should see Prometheus already configured. If not, add a new data source:
   - Select Prometheus
   - Set the URL to `http://prometheus-server`
   - Click "Save & Test"

4. Import dashboards:
   - Click the "+" icon in the left sidebar and select "Import"
   - Enter dashboard ID 3119 for a Kubernetes cluster monitoring dashboard
   - Select the Prometheus data source and click "Import"

## Step 6: (Optional) Expose Grafana

To expose Grafana outside the cluster, you can use a LoadBalancer or an Ingress controller. Here's an example using a LoadBalancer:

```bash
kubectl patch svc grafana -n monitoring -p '{"spec": {"type": "LoadBalancer"}}'
```

Get the external IP:

```bash
kubectl get svc -n monitoring grafana
```

## Verification and Testing

1. Check Prometheus targets:
   - Port-forward the Prometheus server: `kubectl port-forward -n monitoring service/prometheus-server 9090:80`
   - Open `http://localhost:9090/targets` in your browser
   - Verify that targets are up and scraping is successful

2. Check Grafana dashboards:
   - Navigate to the Kubernetes cluster dashboard you imported
   - Verify that metrics are being displayed correctly

## Cleanup

To remove Prometheus and Grafana:

```bash
helm uninstall prometheus -n monitoring
helm uninstall grafana -n monitoring
kubectl delete namespace monitoring
```

Remember to adjust storage classes, passwords, and other parameters according to your specific Kubernetes environment and security requirements.
