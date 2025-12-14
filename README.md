# Challenge

## 1. Environment Setup

**Objective:** Prepare the single Ubuntu VM to host the cluster.

### 1.0 Initial Server Setup (Run as Root)

```bash
# 1. Create a new user (e.g., 'mmdii')

adduser mmdii
usermod -aG sudo mmdii
su - mmdii

sudo apt-get update && sudo apt-get upgrade -y

```

From now on, run all commands as the mmdii user.

### 1.1 Install Prerequisites

Run these commands to install Docker, Kubectl, Helm, and k3d.

```bash
# Update and install Docker
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg lsb-release
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Add user to docker group (avoid sudo for docker)
sudo usermod -aG docker $USER
newgrp docker

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Install k3d
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
```

### 1.2 Prepare Ports for SSL

We need port 80 and 443 free for the cluster to handle real domain traffic.

```bash
# Stop conflicting services
sudo systemctl stop nginx || true
sudo systemctl stop apache2 || true
sudo systemctl disable nginx || true
sudo systemctl disable apache2 || true
```

### 1.3 System Tuning (Fixes "Too Many Open Files" Error)

**Critical:** Loki and Promtail require higher file monitoring limits than the Linux default.

```bash
sudo sysctl -w fs.inotify.max_user_watches=524288
sudo sysctl -w fs.inotify.max_user_instances=512

# Make it persistent across reboots
echo "fs.inotify.max_user_watches = 524288" | sudo tee -a /etc/sysctl.conf
echo "fs.inotify.max_user_instances = 512" | sudo tee -a /etc/sysctl.conf
```

## 2. Cluster Creation (3 Master, 3 Worker)

**Objective:** Create the multi-node topology using containers.

### 2.1 Create the Cluster

**Update:** We now map ports 80 and 443 directly to the host. This allows Let's Encrypt to validate your domain and issue SSL certificates.

```bash
k3d cluster create interview-cluster \
    --servers 3 \
    --agents 3 \
    --port "80:80@loadbalancer" \
    --port "443:443@loadbalancer" \
    --api-port 6443 \
    --k3s-arg "--disable=traefik@server:0"
```

### 2.2 Verify Nodes

```bash
kubectl get nodes
```

**Expected Output:** 3 master nodes, 3 agent (worker) nodes.

## 3. Observability (Monitoring & Logging)

**Objective:** Install Prometheus (Monitoring) and Loki (Logging).

### 3.1 Install Prometheus Stack (Monitoring)

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
kubectl create ns monitoring

# Install with default settings (alertmanager, grafana, node-exporter included)
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack -n monitoring
```

### 3.2 Install Loki Stack (Logging) - Debug Mode

This installs Loki (log store) and Promtail (log shipper).

**Step 1: Install with Safe Configuration**

```bash
# Clean up if needed
helm uninstall loki -n logging 2>/dev/null
kubectl create ns logging 2>/dev/null

# Create configuration file
cat <<EOF > loki-values.yaml
promtail:
  enabled: true
  config:
    clients:
      - url: http://loki.logging.svc.cluster.local:3100/loki/api/v1/push
grafana:
  enabled: false
loki:
  isDefault: false
EOF

helm install loki grafana/loki-stack --namespace logging -f loki-values.yaml
```

### 3.3 Connect Logging to Monitoring (The GitOps Way)

Instead of manually adding the Datasource in the UI (which can fail with health check errors), we create a Secret. Grafana automatically detects this secret and adds Loki correctly.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: loki-datasource
  namespace: monitoring
  labels:
    grafana_datasource: "1"
type: Opaque
stringData:
  loki-datasource.yaml: |-
    apiVersion: 1
    datasources:
    - name: Loki
      type: loki
      url: http://loki.logging.svc.cluster.local:3100
      access: proxy
      isDefault: false
EOF

# Restart Grafana to pick up the new secret immediately
kubectl rollout restart deployment kube-prometheus-stack-grafana -n monitoring
```

## 4. Databases (High Availability)

**Objective:** Deploy HA Postgres and HA Redis with PodAntiAffinity.

### 4.1 PostgreSQL Cluster (using CloudNativePG)

**Install the Operator:**

```bash
kubectl apply -f https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.22/releases/cnpg-1.22.1.yaml
```

**Deploy the HA Cluster:**

Create dummy credentials for backup validation:

```bash
kubectl create secret generic backup-creds \
  --from-literal=ACCESS_KEY_ID=dummy \
  --from-literal=SECRET_ACCESS_KEY=dummy
```

Since we specify the secret name in the manifest below, we MUST create it first.

```bash
kubectl create secret generic voting-db-app-user \
  --from-literal=username=appuser \
  --from-literal=password=postgres
```

**Create postgres-ha.yaml** :

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: voting-db
spec:
  instances: 3
  affinity:
    enablePodAntiAffinity: true 
    topologyKey: kubernetes.io/hostname
  storage:
    size: 1Gi
  # FIX: Allow legacy apps (Result/Worker) to connect without SCRAM-SHA-256
  postgresql:
    pg_hba:
      - host all all all trust
  backup:
    retentionPolicy: "30d"
    barmanObjectStore:
      destinationPath: s3://backups/
      endpointURL: http://minio:9000
      s3Credentials:
        accessKeyId:
          name: backup-creds
          key: ACCESS_KEY_ID
        secretAccessKey:
          name: backup-creds
          key: SECRET_ACCESS_KEY
      wal:
        compression: gzip
  bootstrap:
    initdb:
      database: app
      owner: appuser
      secret:
        name: voting-db-app-user
```

**Apply it:**

```bash
kubectl apply -f postgres-ha.yaml
```

### 4.2 Redis Cluster (using Bitnami)

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install voting-redis bitnami/redis \
  --set architecture=replication \
  --set sentinel.enabled=true \
  --set replica.replicaCount=3 \
  --set replica.podAntiAffinityPreset=hard \
  --set auth.enabled=false
```

## 5. Application (Voting App via ArgoCD)

**Objective:** Deploy the app using Helm and ArgoCD.

### 5.1 Create the Helm Chart Locally

**Important:** Your files should be in a folder named `templates` and `Chart.yaml` at the root of the repo if using the config below.

**Create Directory Structure:**

```bash
mkdir -p templates
# (Assuming you are at the root of your git repo)
```

**Create Chart.yaml:**

```yaml
apiVersion: v2
name: example-voting-app
description: A Helm chart for the Docker Voting App using HA DBs
version: 0.1.0
appVersion: "1.0"
```

**Create templates/bridge-services.yaml:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis
spec:
  type: ExternalName
  externalName: voting-redis-master.default.svc.cluster.local
---
apiVersion: v1
kind: Service
metadata:
  name: db
spec:
  type: ExternalName
  externalName: voting-db-rw.default.svc.cluster.local
```

**Create templates/vote-deploy.yaml:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vote
spec:
  replicas: 1
  selector:
    matchLabels:
      app: vote
  template:
    metadata:
      labels:
        app: vote
    spec:
      containers:
      - name: vote
        image: dockersamples/examplevotingapp_vote:before
        ports:
        - containerPort: 80
        env:
        - name: REDIS_HOST
          value: "redis"
---
apiVersion: v1
kind: Service
metadata:
  name: vote
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: vote
```

**Create templates/result-deploy.yaml:**

Update: Now pointing to voting-db-rw and app database.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: result
spec:
  replicas: 1
  selector:
    matchLabels:
      app: result
  template:
    metadata:
      labels:
        app: result
    spec:
      containers:
      - name: result
        image: dockersamples/examplevotingapp_result:before
        ports:
        - containerPort: 80
        env:
        - name: POSTGRES_HOST
          value: "voting-db-rw" # Direct Service Name
        - name: POSTGRES_PORT
          value: "5432"
        - name: POSTGRES_USER
          value: "appuser"
        - name: POSTGRES_DB
          value: "app"
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: voting-db-app-user
              key: password
---
apiVersion: v1
kind: Service
metadata:
  name: result
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: result
```

**Create templates/worker-deploy.yaml:**

Update: Now pointing to voting-db-rw, voting-redis-master and app database.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: worker
spec:
  replicas: 1
  selector:
    matchLabels:
      app: worker
  template:
    metadata:
      labels:
        app: worker
    spec:
      containers:
      - name: worker
        image: dockersamples/examplevotingapp_worker
        env:
        - name: REDIS_HOST
          value: "voting-redis-master"
        - name: POSTGRES_HOST
          value: "voting-db-rw"
        - name: POSTGRES_USER
          value: "appuser"
        - name: POSTGRES_DB
          value: "app"
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: voting-db-app-user
              key: password
```

### 5.3 Deploy via ArgoCD (Option A: The GitOps Way)

Use this configuration to point ArgoCD to the ROOT of your GitHub repository.

**Install ArgoCD:**

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

**Create application.yaml:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: voting-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/mmdii/technolife-challenge.git'
    path: .  # Points to the root where Chart.yaml is
    targetRevision: HEAD
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

**Apply it:**

```bash
kubectl apply -f application.yaml
```

### 5.3 Manual Deployment (Fast Track)

**IMPORTANT:** Use this if you haven't set up the Git repository yet.

Run this command inside the directory where you created `templates/`:

```bash
# This mimics what ArgoCD would do
kubectl apply -R -f templates/
```

**Verify Deployment:**

Now check your pods again. You should see vote, result, and worker running.

```bash
kubectl get pods
```

## 6. Autoscaling (KEDA)

```bash
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
helm install keda kedacore/keda --namespace keda --create-namespace
```

**Create keda-scale.yaml:**

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: voting-worker-scaler
  namespace: default
spec:
  scaleTargetRef:
    name: worker
  triggers:
  - type: redis
    metadata:
      address: voting-redis-master.default.svc.cluster.local:6379
      listName: vote
      listLength: "5"
```

**Apply it:**

```bash
kubectl apply -f keda-scale.yaml
```

## 7. Ingress with SSL (Cert-Manager)

**Objective:** Configure technolife.mmdii.net with auto-renewing SSL certificates.

### 7.1 Install Cert-Manager

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.13.1 \
  --set installCRDs=true
```

### 7.2 Create ClusterIssuer (Let's Encrypt)

**Create cluster-issuer.yaml.** Replace with your real email.

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@mmdii.net
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: traefik
```

**Apply:**

```bash
kubectl apply -f cluster-issuer.yaml
```

### 7.3 Create App Ingress (Default Namespace)

**Create ingress-app.yaml.** This handles the voting app and results page.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: default
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: traefik
  tls:
  - hosts:
    - vote.technolife.mmdii.net
    - result.technolife.mmdii.net
    secretName: app-tls-secret
  rules:
  - host: vote.technolife.mmdii.net
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: vote
            port:
              number: 80
  - host: result.technolife.mmdii.net
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: result
            port:
              number: 80
```

**Apply:**

```bash
kubectl apply -f ingress-app.yaml
```

### 7.4 Create Monitoring Ingress (Monitoring Namespace)

**Critical Fix for Grafana:** We create this ingress inside the monitoring namespace so it can access the Grafana service directly.

**Create ingress-monitoring.yaml:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana-ingress
  namespace: monitoring
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: traefik
  tls:
  - hosts:
    - grafana.technolife.mmdii.net
    secretName: grafana-tls-secret
  rules:
  - host: grafana.technolife.mmdii.net
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: kube-prometheus-stack-grafana
            port:
              number: 80
```

**Apply:**

```bash
kubectl apply -f ingress-monitoring.yaml
```

### 7.5 Create ArgoCD Ingress & Access

**Objective:** Access the ArgoCD Dashboard securely.

**Enable Insecure Mode (Patch ArgoCD):**

We need ArgoCD to serve HTTP so Traefik can handle the SSL termination.

**Fixed Command:** This uses a generic patch that works regardless of whether the container uses command or args array structures.

```bash
kubectl patch deployment argocd-server -n argocd --patch '{"spec": {"template": {"spec": {"containers": [{"name": "argocd-server", "args": ["--insecure"]}]}}}}'

# Wait for restart
kubectl rollout status deployment argocd-server -n argocd
```

**Create Ingress for ArgoCD:**

**Create ingress-argocd.yaml:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-ingress
  namespace: argocd
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: traefik
  tls:
  - hosts:
    - argocd.technolife.mmdii.net
    secretName: argocd-tls-secret
  rules:
  - host: argocd.technolife.mmdii.net
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              number: 80
```

**Apply it:**

```bash
kubectl apply -f ingress-argocd.yaml
```

**Get Login Password:**

- **User:** admin
- **Password:** Run this command:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

**Objective:** Control the cluster from your personal laptop.

**On Server:** Get the kubeconfig with the public IP.

```bash
# Ensure port 6443 is open on server firewall if needed
k3d kubeconfig get interview-cluster > config.yaml
sed -i 's/0.0.0.0/185.231.59.38/g' config.yaml
cat config.yaml
```

**On Local Machine:**

1. Copy the output of `cat config.yaml`
2. Create a file `~/.kube/config-mmdii` and paste the content

**Important:** Since the SSL cert generated by k3d is for "localhost", your computer will reject the certificate for the IP "185.231.59.38".

**Quick Fix:** Edit your local config file. Under `clusters:` -> `cluster:`, add:

```yaml
insecure-skip-tls-verify: true
# Remove "certificate-authority-data" if you use this
```

**Run:**

```bash
export KUBECONFIG=~/.kube/config-mmdii
```

**Test:**

```bash
kubectl get nodes
```

## 8. Dashboard Import IDs

Once you log into Grafana (<https://grafana.technolife.mmdii.net>), use these IDs to import dashboards (Dashboards → New → Import):

- **Kubernetes Cluster Overview:** 3119
- **Loki Logs (Pod Search):** 15141
