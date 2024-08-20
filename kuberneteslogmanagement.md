# Kubernetes Logging Lab: Exploring Logs in a Cluster

This lab will guide you through the process of deploying a simple application, generating logs, and exploring various ways to view and analyze logs in Kubernetes.

## Prerequisites

- A running Kubernetes cluster (minikube, kind, or a cloud-based cluster)
- `kubectl` configured to communicate with your cluster

## Lab Steps

### 1. Create a Namespace

First, let's create a namespace for our logging experiments:

```bash
kubectl create namespace logging-lab
```

### 2. Deploy a Sample Application

We'll deploy a simple web application that generates logs:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: logging-lab
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web-app
        image: nginx:latest
        ports:
        - containerPort: 80
        command: ["/bin/sh"]
        args: ["-c", "while true; do echo $(date) - App is running; sleep 5; done"]
```

Save this as `web-app.yaml` and apply it:

```bash
kubectl apply -f web-app.yaml
```

### 3. View Pod Logs

Now, let's explore different ways to view logs:

a. View logs for a specific pod:
   ```bash
   POD_NAME=$(kubectl get pods -n logging-lab -l app=web-app -o jsonpath="{.items[0].metadata.name}")
   kubectl logs -n logging-lab $POD_NAME
   ```

b. View logs for all pods with a specific label:
   ```bash
   kubectl logs -n logging-lab -l app=web-app --all-containers=true
   ```

c. View logs with timestamps:
   ```bash
   kubectl logs -n logging-lab $POD_NAME --timestamps=true
   ```

d. Follow logs in real-time:
   ```bash
   kubectl logs -n logging-lab $POD_NAME -f
   ```

### 4. Explore Multi-Container Pod Logs

Let's add a sidecar container to our deployment to explore multi-container logging:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app-sidecar
  namespace: logging-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-app-sidecar
  template:
    metadata:
      labels:
        app: web-app-sidecar
    spec:
      containers:
      - name: web-app
        image: nginx:latest
        ports:
        - containerPort: 80
        command: ["/bin/sh"]
        args: ["-c", "while true; do echo $(date) - Main app is running; sleep 5; done"]
      - name: sidecar
        image: busybox
        command: ["/bin/sh"]
        args: ["-c", "while true; do echo $(date) - Sidecar is running; sleep 7; done"]
```

Save this as `web-app-sidecar.yaml` and apply it:

```bash
kubectl apply -f web-app-sidecar.yaml
```

Now, let's view logs from this multi-container pod:

a. View logs from a specific container:
   ```bash
   POD_NAME=$(kubectl get pods -n logging-lab -l app=web-app-sidecar -o jsonpath="{.items[0].metadata.name}")
   kubectl logs -n logging-lab $POD_NAME -c web-app
   ```

b. View logs from all containers in the pod:
   ```bash
   kubectl logs -n logging-lab $POD_NAME --all-containers=true
   ```

### 5. Use `kubectl describe` for Log-Related Information

`kubectl describe` can provide valuable information about the pod and its log sources:

```bash
kubectl describe pod -n logging-lab $POD_NAME
```

### 6. Explore Log Rotation

Kubernetes automatically rotates logs for you. To see this in action, we can generate a lot of logs quickly:

```bash
kubectl exec -n logging-lab $POD_NAME -c web-app -- /bin/sh -c 'for i in {1..100000}; do echo "Log line $i"; done'
```

Then view the logs again to see how Kubernetes has handled the large volume of logs.

### 7. Use a Log Aggregator (Optional)

For a more advanced logging setup, you might want to use a log aggregator like Fluentd, Logstash, or ELK stack. Here's a simple example using Fluentd:

a. Deploy Fluentd:
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/fluent/fluentd-kubernetes-daemonset/master/fluentd-daemonset-elasticsearch.yaml
   ```

b. Deploy Elasticsearch:
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/fluentd-elasticsearch/es-service.yaml
   kubectl apply -f https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/fluentd-elasticsearch/es-statefulset.yaml
   ```

c. Deploy Kibana:
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/fluentd-elasticsearch/kibana-deployment.yaml
   kubectl apply -f https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/fluentd-elasticsearch/kibana-service.yaml
   ```

d. Access Kibana:
   ```bash
   kubectl port-forward svc/kibana-logging 5601:5601
   ```

   Open a web browser and navigate to `http://localhost:5601` to explore your logs in Kibana.

### 8. Clean Up

When you're done experimenting, clean up the resources:

```bash
kubectl delete namespace logging-lab
kubectl delete -f https://raw.githubusercontent.com/fluent/fluentd-kubernetes-daemonset/master/fluentd-daemonset-elasticsearch.yaml
kubectl delete -f https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/fluentd-elasticsearch/es-service.yaml
kubectl delete -f https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/fluentd-elasticsearch/es-statefulset.yaml
kubectl delete -f https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/fluentd-elasticsearch/kibana-deployment.yaml
kubectl delete -f https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/fluentd-elasticsearch/kibana-service.yaml
```

This lab provides a comprehensive exploration of logging in Kubernetes, from basic log viewing to advanced log aggregation. It covers various scenarios and tools you might encounter in real-world Kubernetes environments.
