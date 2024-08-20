# Deploying Prometheus and Grafana in Kubernetes with Local Storage

This guide will walk you through deploying Prometheus and Grafana in a Kubernetes cluster using local storage.

## Prerequisites

- A running Kubernetes cluster
- `kubectl` configured to communicate with your cluster
- Helm 3 installed

## Step 1: Create a Monitoring Namespace

```bash
kubectl create namespace monitoring
```

## Step 2: Create StorageClass for Local Storage

First, we need to create a StorageClass for local storage:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

Save this as `local-storage-class.yaml` and apply:

```bash
kubectl apply -f local-storage-class.yaml
```

## Step 3: Create PersistentVolumes for Prometheus and Grafana

We'll create PersistentVolumes (PVs) for both Prometheus and Grafana. You need to run these on the nodes where you want the storage to be allocated.

For Prometheus:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: prometheus-local-storage
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /mnt/data/prometheus
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - your-node-name  # Replace with your actual node name
```

For Grafana:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: grafana-local-storage
spec:
  capacity:
    storage: 5Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /mnt/data/grafana
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - your-node-name  # Replace with your actual node name
```

Save these as `prometheus-pv.yaml` and `grafana-pv.yaml` respectively, and apply:

```bash
kubectl apply -f prometheus-pv.yaml
kubectl apply -f grafana-pv.yaml
```

Make sure the directories `/mnt/data/prometheus` and `/mnt/data/grafana` exist on the specified node.

## Step 4: Deploy Prometheus

Add the Prometheus community Helm repository:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

Install Prometheus:

```bash
helm install prometheus prometheus-community/prometheus \
  --namespace monitoring \
  --set alertmanager.persistentVolume.storageClass="local-storage" \
  --set server.persistentVolume.storageClass="local-storage" \
  --set server.persistentVolume.size=10Gi
```

## Step 5: Deploy Grafana

Add the Grafana Helm repository:

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

Install Grafana:

```bash
helm install grafana grafana/grafana \
  --namespace monitoring \
  --set persistence.storageClassName="local-storage" \
  --set persistence.size=5Gi \
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

## Step 6: Access and Configure Grafana

Follow the same steps as in the previous guide to access and configure Grafana:

1. Get the Grafana admin password:

   ```bash
   kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
   ```

2. Port-forward the Grafana service:

   ```bash
   kubectl port-forward -n monitoring service/grafana 3000:80
   ```

3. Access Grafana at `http://localhost:3000` and log in.

4. Configure the Prometheus data source and import dashboards as described in the previous guide.

## Verification and Testing

Verify that the PersistentVolumeClaims (PVCs) are bound:

```bash
kubectl get pvc -n monitoring
```

Check that Prometheus and Grafana pods are running:

```bash
kubectl get pods -n monitoring
```

## Cleanup

To remove Prometheus, Grafana, and the local storage setup:

```bash
helm uninstall prometheus -n monitoring
helm uninstall grafana -n monitoring
kubectl delete pv prometheus-local-storage grafana-local-storage
kubectl delete storageclass local-storage
kubectl delete namespace monitoring
```

Remember to also clean up the local directories on your nodes.

Note: Using local storage means that if a pod is rescheduled to a different node, it will lose access to its previous data. This setup is more suitable for testing or development environments. For production, consider using a distributed storage solution.
