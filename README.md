# Local Kubernetes Fundamentals (Minikube)

## 📖 Key Concepts (Theory)

Before the practical steps, here is a breakdown of the core technologies used in this project.

---

## 1. Docker: Building Images

**Concept:**  
Docker packages an application and all its dependencies (libraries, code, runtime) into a single **Image**. This ensures the app runs exactly the same on your laptop as it does on a server.

**In this project:**  
We built a custom Nginx image. Crucially, we pointed our local terminal to Minikube's internal Docker daemon so we didn't have to push the image to a public registry (like Docker Hub).

---

## 2. Deployments: Running Stateless Apps

**Concept:**  
A Deployment is a supervisor for your Pods. You describe the desired state (e.g., *"I want 2 copies of Nginx"*), and the Deployment ensures that state is always true. If a Pod crashes, the Deployment replaces it.

**Stateless:**  
The pods don't remember data. If they die, their memory is wiped. This is perfect for web servers.

---

## 3. Services: Networking

**Concept:**  
Pods are *"mortal"*—they die and get new IP addresses constantly. A Service is a stable, permanent address (like a receptionist) that sits in front of the Pods and forwards traffic to whichever ones are currently alive.

**NodePort:**  
Exposes the service on a specific port on the Node (VM) itself, allowing external access.

**ClusterIP:**  
Exposes the service only inside the cluster (internal only).

---

## 4. PVCs (Persistent Volume Claims): Managing Storage

**Concept:**  
To save data permanently (like for a database), we cannot use the Pod's internal filesystem (which is ephemeral). We request a *"slice"* of external storage called a Persistent Volume.

**PVC:**  
This is the *"ticket"* or request we submit to Kubernetes asking for storage (e.g., *"I need 1GB"*). Kubernetes finds a matching Volume and attaches it to our Pod.

---

## 5. ConfigMaps & Secrets: Managing Configuration

**Concept:**  
You should never hardcode passwords or configuration files inside your Docker image.

**ConfigMap:**  
Stores non-sensitive data (config files, HTML, environment variables). We used this to inject `index.html`.

**Secret:**  
Stores sensitive data (passwords, keys) encoded in Base64. We used this for the Redis password.

---

## 6. HPA (Horizontal Pod Autoscaler): Auto-scaling

**Concept:**  
HPA watches the resource usage (like CPU) of your Pods. If the average usage exceeds a target (e.g., 50%), HPA instructs the Deployment to add more Pods (scale out). When the load drops, it removes them (scale in).

---

# 🛠️ Implementation Guide

## Prerequisites

- **Minikube:** The local Kubernetes cluster  
- **Kubectl:** The command-line tool to talk to the cluster  
- **Docker:** The container runtime  

---

## Step 1: Environment Setup & Docker Build

We need to build our custom image directly inside Minikube's environment so Kubernetes can find it.

### Point Docker to Minikube

```bash
eval $(minikube -p minikube docker-env)
````

### Build the Image

```bash
docker build -t k8s-web-app:v1 .
```

> **Note:** We used `imagePullPolicy: Never` in the YAML to force Kubernetes to use this local image.

---

## Step 2: Namespace Isolation

We created a separate namespace to keep our work organized and isolated from the default system services.

### Create the Namespace

```bash
kubectl apply -f namespace.yaml
```

### Switch Context (The "Pro" move)

This ensures all future `kubectl` commands run inside our namespace by default.

```bash
kubectl config set-context --current --namespace=k8s-learn-ns
```

---

## Step 3: Deploying the Stateless Frontend

We deployed the web server and exposed it to our local browser.

### Apply Configuration

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

### View the App

```bash
minikube service web-app-service -n k8s-learn-ns
```

---

## Step 4: Adding Persistence (Redis Database)

We added a Redis database that retains data even if the Pod is deleted.

### Apply Storage & Database

```bash
kubectl apply -f redis-pvc.yaml
kubectl apply -f redis-deployment.yaml
kubectl apply -f redis-service.yaml
```

### Verify Persistence

1. Exec into the pod → Write data (`set key value`) → Save
2. Delete the pod

   ```bash
   kubectl delete pod <name>
   ```
3. Exec into the new pod → Read data

The data should still be there.

---

## Step 5: Decoupling Configuration (ConfigMaps & Secrets)

We moved the HTML content and Database Password out of the code and into Kubernetes objects.

### Apply Configs

```bash
kubectl apply -f redis-secret.yaml
kubectl apply -f webapp-config.yaml
```

### Update Deployments

* Redis was updated to read the password from the Secret
* Web App was updated to mount the `index.html` from the ConfigMap

```bash
kubectl apply -f redis-deployment.yaml
kubectl apply -f deployment.yaml
```

---

## Step 6: Auto-Scaling (HPA)

We configured the app to scale automatically under stress.

### Enable Metrics Server (Required for HPA)

```bash
minikube addons enable metrics-server
```

### Apply HPA Rule (CPU > 50%)

```bash
kubectl apply -f hpa.yaml
```

### Stress Test

We deployed a **Load Generator (BusyBox)** that constantly sent requests to the service.

```bash
kubectl apply -f load-generator.yaml
```

### Monitor Scaling

```bash
kubectl get hpa -w
```

**Result:**
Replica count increased from **1 → 10** under load.

---

## Step 7: Cleanup

Delete the entire project using the namespace.

```bash
kubectl delete namespace k8s-learn-ns
```

### Stop Minikube

```bash
minikube stop
```


