# Kubernetes Deployment Guide for SkibidiChat

This guide will help you deploy SkibidiChat on a fresh Ubuntu instance using Kubernetes. Follow the steps in order to get your application running.

## 📋 Prerequisites

- A fresh Ubuntu 20.04+ instance (or any Linux distribution)
- SSH access with sudo privileges
- Internet connectivity

---

## 🛠️ Step 1: Install Required Tools (In Order)

### 1.1 Update System Packages

```bash
sudo apt update && sudo apt upgrade -y
```

### 1.2 Install Docker

**Yes, Docker is required!** You need Docker to build the custom images (`skibidichat/jettybackend:latest` and `skibidichat/caddy-jwt-custom:latest`) that are referenced in the Kubernetes manifests.

```bash
# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add your user to docker group (to run docker without sudo)
sudo usermod -aG docker $USER

# Start and enable Docker
sudo systemctl start docker
sudo systemctl enable docker

# Verify installation
docker --version
```

**Important:** Log out and log back in (or run `newgrp docker`) for the group changes to take effect.

### 1.3 Install kubectl

```bash
# Download kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Install kubectl
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Verify installation
kubectl version --client
```

### 1.4 Install a Kubernetes Cluster

Choose one of the following options:

#### Option A: Minikube (Recommended for Local/Testing)

```bash
# Install Minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Start Minikube cluster
minikube start

# Enable Minikube Docker registry (so you can use local images)
eval $(minikube docker-env)
```

#### Option B: k3s (Lightweight Kubernetes)

```bash
# Install k3s
curl -sfL https://get.k3s.io | sh -

# Verify installation
sudo k3s kubectl get nodes
```

#### Option C: Use an Existing Cloud Kubernetes Cluster

If you're using AWS EKS, Google GKE, or Azure AKS, configure kubectl to connect:

```bash
# Example for AWS EKS
aws eks update-kubeconfig --name your-cluster-name --region your-region

# Example for Google GKE
gcloud container clusters get-credentials your-cluster-name --zone your-zone

# Example for Azure AKS
az aks get-credentials --resource-group your-resource-group --name your-cluster-name
```

---

## 🐳 Step 2: Build Docker Images

Since the Kubernetes manifests reference custom images (`skibidichat/jettybackend:latest` and `skibidichat/caddy-jwt-custom:latest`), you need to build them first.

### 2.1 Clone the Repository

```bash
git clone https://github.com/pravindeshmukh8702/SkibidiChat.git
cd SkibidiChat
```

### 2.2 Build the Images

**For Minikube users:**
```bash
# Make sure Minikube Docker environment is active
eval $(minikube docker-env)

# Build Jetty backend image
docker build -t skibidichat/jettybackend:latest -f Dockerfile .

# Build Caddy image
docker build -t skibidichat/caddy-jwt-custom:latest -f caddy.Dockerfile .
```

**For k3s or Cloud Kubernetes users:**
You need to push images to a container registry (Docker Hub, AWS ECR, GCR, etc.):

```bash
# Tag images for your registry (example with Docker Hub)
docker build -t skibidichat/jettybackend:latest -f Dockerfile .
docker build -t skibidichat/caddy-jwt-custom:latest -f caddy.Dockerfile .

# Login to your registry
docker login

# Tag and push
docker tag skibidichat/jettybackend:latest your-registry/skibidichat/jettybackend:latest
docker tag skibidichat/caddy-jwt-custom:latest your-registry/skibidichat/caddy-jwt-custom:latest

docker push your-registry/skibidichat/jettybackend:latest
docker push your-registry/skibidichat/caddy-jwt-custom:latest
```

**Then update the image names in `k8s/backend.yaml` and `k8s/caddy.yaml`** to use your registry path.

**Note:** You do NOT need to run `docker compose pull` or `docker compose up` for Kubernetes deployment. Docker Compose is only needed if you want to run the application using Docker Compose instead of Kubernetes.

---

## ☸️ Step 3: Apply Kubernetes Manifests (In Order)

Apply the manifests in the following order to ensure dependencies are created correctly:

### 3.1 Create Namespace

```bash
kubectl apply -f k8s/namespace.yaml
```

Verify:
```bash
kubectl get namespace skibidichat
```

### 3.2 Create JWT Secret

```bash
kubectl apply -f k8s/secret-jwt.yaml
```

**Important:** Before applying, you should update `k8s/secret-jwt.yaml` with your own base64-encoded JWT secret:

```bash
# Generate a base64-encoded secret
echo -n "your-super-secret-jwt-key" | base64

# Update the value in k8s/secret-jwt.yaml
```

Verify:
```bash
kubectl get secret skibidichat-jwt-secret -n skibidichat
```

### 3.3 Deploy MongoDB

```bash
kubectl apply -f k8s/mongodb.yaml
```

This creates:
- PersistentVolumeClaim for MongoDB data
- Deployment for MongoDB
- Service for MongoDB

Verify:
```bash
kubectl get pods -n skibidichat -l app=mongodb
kubectl get svc -n skibidichat mongodb
```

### 3.4 Deploy Redis

```bash
kubectl apply -f k8s/redis.yaml
```

This creates:
- PersistentVolumeClaim for Redis data
- Deployment for Redis
- Service for Redis

Verify:
```bash
kubectl get pods -n skibidichat -l app=redis
kubectl get svc -n skibidichat redis
```

### 3.5 Deploy Caddy Configuration

```bash
kubectl apply -f k8s/caddy-config.yaml
```

This creates:
- ConfigMap for Caddyfile
- ConfigMap for web static files (placeholder)

Verify:
```bash
kubectl get configmap -n skibidichat
```

### 3.6 Deploy Jetty Backend

```bash
kubectl apply -f k8s/backend.yaml
```

This creates:
- Deployment for Jetty backend
- Service for Jetty backend

Verify:
```bash
kubectl get pods -n skibidichat -l app=jettybackend
kubectl get svc -n skibidichat jettybackend
```

### 3.7 Deploy Caddy (Reverse Proxy)

```bash
kubectl apply -f k8s/caddy.yaml
```

This creates:
- PersistentVolumeClaims for Caddy data and config
- Deployment for Caddy
- Service (LoadBalancer) for Caddy

Verify:
```bash
kubectl get pods -n skibidichat -l app=caddy
kubectl get svc -n skibidichat caddy
```

---

## 📊 Step 4: Check Pod Status and Logs

### 4.1 Check Pod Status

**View all pods in the namespace:**
```bash
kubectl get pods -n skibidichat
```

**Watch pods in real-time:**
```bash
kubectl get pods -n skibidichat -w
```

**Get detailed pod information:**
```bash
kubectl describe pod <pod-name> -n skibidichat
```

**Check if all pods are running:**
```bash
kubectl get pods -n skibidichat
```

You should see all pods in `Running` state. If any pod is in `Pending`, `Error`, or `CrashLoopBackOff`, check the logs (see below).

### 4.2 View Pod Logs

**View logs for a specific pod:**
```bash
# Get pod name first
kubectl get pods -n skibidichat

# View logs
kubectl logs <pod-name> -n skibidichat

# Example:
kubectl logs caddy-xxxxxxxxxx-xxxxx -n skibidichat
kubectl logs jettybackend-xxxxxxxxxx-xxxxx -n skibidichat
kubectl logs mongodb-xxxxxxxxxx-xxxxx -n skibidichat
kubectl logs redis-xxxxxxxxxx-xxxxx -n skibidichat
```

**Follow logs in real-time (like `tail -f`):**
```bash
kubectl logs -f <pod-name> -n skibidichat
```

**View logs for all pods with a specific label:**
```bash
kubectl logs -l app=caddy -n skibidichat
kubectl logs -l app=jettybackend -n skibidichat
```

**View previous container logs (if pod restarted):**
```bash
kubectl logs <pod-name> -n skibidichat --previous
```

### 4.3 Check Service Status

```bash
# View all services
kubectl get svc -n skibidichat

# Get detailed service information
kubectl describe svc caddy -n skibidichat
```

---

## 🌐 Step 5: Access the Application

### 5.1 Get the Public IP and Port

The Caddy service is configured as `LoadBalancer` type, which exposes the application externally.

**For Minikube:**
```bash
# Get the service URL
minikube service caddy -n skibidichat --url

# Or use port forwarding (see below)
```

**For Cloud Kubernetes (AWS, GCP, Azure):**
```bash
# Get the external IP
kubectl get svc caddy -n skibidichat

# Wait for EXTERNAL-IP to be assigned (may take a few minutes)
# Example output:
# NAME    TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)                        AGE
# caddy   LoadBalancer   10.96.xxx.xxx   203.0.113.42     80:3xxxx/TCP,443:3xxxx/TCP,8080:3xxxx/TCP   5m
```

**For k3s (or if LoadBalancer doesn't work):**
Use port forwarding (see section 5.2).

### 5.2 Port Forwarding (Alternative Access Method)

If LoadBalancer doesn't provide an external IP, or for quick local testing:

```bash
# Forward port 8080 from Caddy service to your local machine
kubectl port-forward svc/caddy 8080:8080 -n skibidichat

# Or forward all ports
kubectl port-forward svc/caddy 80:80 443:443 8080:8080 -n skibidichat
```

**Keep this terminal open** while accessing the application. Access via:
- `http://localhost:8080` (if forwarding port 8080)
- `http://localhost:80` (if forwarding port 80)

### 5.3 Access via Public IP

Once you have the external IP (from LoadBalancer or your instance's public IP):

1. **Get your instance's public IP:**
   ```bash
   curl ifconfig.me
   # or
   curl icanhazip.com
   ```

2. **Configure Security Group/Firewall:**
   - **AWS EC2:** Open ports 80, 443, and 8080 in Security Group
   - **GCP:** Create firewall rule allowing ports 80, 443, 8080
   - **Azure:** Configure Network Security Group for ports 80, 443, 8080
   - **Local/On-premise:** Configure firewall:
     ```bash
     sudo ufw allow 80/tcp
     sudo ufw allow 443/tcp
     sudo ufw allow 8080/tcp
     sudo ufw reload
     ```

3. **Access the application:**
   - `http://<PUBLIC_IP>:8080` (recommended, based on Caddyfile configuration)
   - `http://<PUBLIC_IP>:80`
   - `https://<PUBLIC_IP>:443` (if TLS is configured)

**Example:**
If your public IP is `203.0.113.42`, access:
```
http://203.0.113.42:8080
```

---

## 🔧 Troubleshooting

### Pods Not Starting

```bash
# Check pod events
kubectl describe pod <pod-name> -n skibidichat

# Check pod logs
kubectl logs <pod-name> -n skibidichat

# Check if images are available
kubectl describe pod <pod-name> -n skibidichat | grep Image
```

### Image Pull Errors

If you see `ImagePullBackOff` or `ErrImagePull`:

- **Minikube:** Make sure you built images with `eval $(minikube docker-env)` active
- **Cloud/Remote:** Ensure images are pushed to a registry and image pull secrets are configured if needed

### Service Not Accessible

```bash
# Check service endpoints
kubectl get endpoints -n skibidichat

# Check if pods are ready
kubectl get pods -n skibidichat

# Test service connectivity from within cluster
kubectl run -it --rm debug --image=busybox --restart=Never -n skibidichat -- wget -O- http://caddy:8080
```

### Database Connection Issues

```bash
# Check MongoDB logs
kubectl logs -l app=mongodb -n skibidichat

# Check Redis logs
kubectl logs -l app=redis -n skibidichat

# Verify backend can reach databases
kubectl exec -it <jettybackend-pod> -n skibidichat -- env | grep -E "MONGO|REDIS"
```

---

## 📝 Quick Reference Commands

```bash
# View all resources
kubectl get all -n skibidichat

# Delete everything (cleanup)
kubectl delete namespace skibidichat

# Restart a deployment
kubectl rollout restart deployment/<deployment-name> -n skibidichat

# Scale a deployment
kubectl scale deployment/<deployment-name> --replicas=2 -n skibidichat

# View resource usage
kubectl top pods -n skibidichat
kubectl top nodes
```

---

## 🎯 Summary

1. ✅ Install Docker, kubectl, and a Kubernetes cluster (Minikube/k3s/Cloud)
2. ✅ Build Docker images (`jettybackend` and `caddy-jwt-custom`)
3. ✅ Apply manifests in order: namespace → secret → mongodb → redis → caddy-config → backend → caddy
4. ✅ Check pod status: `kubectl get pods -n skibidichat`
5. ✅ View logs: `kubectl logs <pod-name> -n skibidichat`
6. ✅ Access via LoadBalancer IP or port forwarding: `http://<PUBLIC_IP>:8080`

**Docker is required** to build the custom images, but you **do NOT need** to run `docker compose pull` or `docker compose up` for Kubernetes deployment.

---

## 📚 Additional Resources

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [Minikube Documentation](https://minikube.sigs.k8s.io/docs/)
- [k3s Documentation](https://k3s.io/)

