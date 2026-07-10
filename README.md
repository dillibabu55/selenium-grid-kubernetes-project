 # Multi-Node Kubernetes (kubeadm) Setup Guide
**Master Machine:** `zprojects-u18-aio` (Ubuntu 18.04)  
**Worker Machine:** `zp-u16-aio-11`  
**Purpose:** Selenium Grid for ZPTool Web Automation  
**Date:** June 2026  

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Prerequisites](#2-prerequisites)
3. [Phase 1 — Tear Down minikube on Master](#3-phase-1--tear-down-minikube-on-master)
4. [Phase 2 — Prepare Both Machines](#4-phase-2--prepare-both-machines)
5. [Phase 3 — Initialize Master Node](#5-phase-3--initialize-master-node)
6. [Phase 4 — Join Worker Node](#6-phase-4--join-worker-node)
7. [Phase 5 — Deploy Selenium Grid](#7-phase-5--deploy-selenium-grid)
8. [Phase 6 — NodePort + socat as systemd Service](#8-phase-6--nodeport--socat-as-systemd-service)
9. [Phase 7 — Update Autospark Properties](#9-phase-7--update-autospark-properties)
10. [Phase 8 — Verification Checklist](#10-phase-8--verification-checklist)
11. [Live VNC Browser Viewing](#11-live-vnc-browser-viewing)
12. [Scaling Pods at Runtime](#12-scaling-pods-at-runtime)
13. [Useful kubectl Commands](#13-useful-kubectl-commands)
14. [Troubleshooting](#14-troubleshooting)
15. [Comparison: minikube vs kubeadm](#15-comparison-minikube-vs-kubeadm)

---

## 1. Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         zprojects-u18-aio  (MASTER)                       │
│                                                                           │
│   ZPTool / AutoSpark Java App (:9090)                                     │
│   hub_url = http://zprojects-u18-aio:30444  ── socat :4444→:30444 ────┐ │
│                                                                         │ │
│   kube-apiserver  kube-scheduler  kube-controller  etcd                 │ │
│   (Control plane only — no Selenium pods run here)                      │ │
└─────────────────────────────────────────────────────────────────────────┼─┘
                                                                          │
                                              K8s API + port-forward      │
                                                                          │
┌─────────────────────────────────────────────────────────────────────────┼─┐
│         WORKERS: zp-u16-aio-11 (15Gi) + zpbt-autovm33 (30Gi) + zpbt-autovm34 (30Gi) │ │
│                                                                          │ │
│   kubelet  kube-proxy  containerd  (on each worker)                     │ │
│                                                                          │ │
│   ┌──────────────────────────────────────────────────────────────────┐  │ │
│   │  namespace: selenium  (pods distributed across all 3 workers)    │  │ │
│   │                                                                   │  │ │
│   │  selenium-hub Pod (1)                                             │  │ │
│   │       │                                                           │  │ │
│   │  ┌────┴──────────┬──────────────────┐                            │  │ │
│   │  ▼               ▼                  ▼                            │  │ │
│   │  Chrome Pods    Firefox Pods       Edge Pods                     │  │ │
│   │  (25 replicas)  (25 replicas)      (8 replicas)                  │  │ │
│   │  50 slots       50 slots           16 slots = 116 total          │  │ │
│   │  VNC port 7900  VNC port 7900      VNC port 7900                 │  │ │
│   └──────────────────────────────────────────────────────────────────┘  │ │
└─────────────────────────────────────────────────────────────────────────┘ │
                                                                            │
         hub_url=http://zprojects-u18-aio:30444  ◀──────────────────────────┘
```

---

## 2. Prerequisites

### Machine Requirements

| Role | Machine | CPU | RAM | OS |
|---|---|---|---|---|
| Master | `zprojects-u18-aio` | 2 vCPU | 4 GB | Ubuntu 18.04 |
| Worker 1 | `zp-u16-aio-11` | 8 vCPU | 15.4 GB | Ubuntu 18.04.5 LTS |
| Worker 2 | `zpbt-autovm33` | 10 vCPU | 30.5 GB | Ubuntu 22.04 LTS |
| Worker 3 | `zpbt-autovm34` | 10 vCPU | 30.5 GB | Ubuntu 22.04 LTS |

### Network Requirements
- Both machines must be on the same network / able to ping each other
- Master's IP must be reachable from worker on port `6443` (K8s API)
- Find master IP: `hostname -I | awk '{print $1}'`

### What's already on Master (`zprojects-u18-aio`)
| Tool | Version | Status |
|---|---|---|
| containerd | v1.6.12 | ✅ Installed |
| kubectl | v1.36.1 | ✅ Installed |
| Docker | 20.10.21 | ✅ Installed |
| minikube | v1.38.1 | ✅ Will be removed in Phase 1 |

---

## 3. Phase 1 — Tear Down minikube on Master

> Run on **`zprojects-u18-aio`** only.

```bash
# Stop and delete the existing minikube cluster
sudo minikube stop
sudo minikube delete --all --purge

# Confirm K8s is gone
kubectl get nodes
# Expected: "The connection to the server localhost:8080 was refused"
```

---

## 4. Phase 2 — Prepare Both Machines

> Run **all steps in this section on BOTH master and worker** unless noted.

### Step 2.1 — Disable Swap

```bash
sudo swapoff -a

# Make permanent across reboots
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Verify
free -h
# Swap line should show 0
```

### Step 2.2 — Enable Required Kernel Modules

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Verify modules are loaded
lsmod | grep -E "overlay|br_netfilter"
```

### Step 2.3 — Set sysctl Network Params

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system

# Verify
sysctl net.ipv4.ip_forward
# Expected: net.ipv4.ip_forward = 1
```

### Step 2.4 — Install containerd

> **Master:** containerd is already installed. Only run the SystemdCgroup fix below.  
> **Worker:** Run the full block.

```bash
# WORKER ONLY — install containerd
sudo apt-get update
sudo apt-get install -y containerd

# Generate default config
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
```

### Step 2.5 — Enable SystemdCgroup (BOTH machines)

> This is required for Kubernetes — without it, kubelet will fail to start.

```bash
# Check current value
grep "SystemdCgroup" /etc/containerd/config.toml

# If it shows false, fix it
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# Verify the fix
grep "SystemdCgroup" /etc/containerd/config.toml
# Expected: SystemdCgroup = true

# Restart containerd
sudo systemctl restart containerd
sudo systemctl enable containerd

# Verify running
sudo systemctl status containerd | grep Active
# Expected: Active: active (running)
```

### Step 2.6 — Install kubeadm, kubelet, kubectl (BOTH machines)

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl

# Create keyring directory
sudo mkdir -p /etc/apt/keyrings

# Add Kubernetes apt repo (v1.28)
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl

# Pin versions to prevent accidental upgrades
sudo apt-mark hold kubelet kubeadm kubectl

# Verify
kubeadm version
kubectl version --client
kubelet --version
```

---

## 5. Phase 3 — Initialize Master Node

> Run on **`zprojects-u18-aio` (master) only**.

### Step 3.1 — Get Master IP

```bash
hostname -I | awk '{print $1}'
# Note this IP — used in kubeadm init and by the worker
# Example: 10.71.114.X
```

### Step 3.2 — Initialize the Cluster

```bash
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --kubernetes-version=v1.28.0 \
  --cri-socket=unix:///run/containerd/containerd.sock

# Expected output at the end:
# Your Kubernetes control-plane has initialized successfully!
# ...
# kubeadm join 10.71.114.X:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

> **IMPORTANT:** Copy and save the `kubeadm join ...` command from the output. You will need it in Phase 4.

### Step 3.3 — Configure kubectl Access

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Verify
kubectl get nodes
# NAME               STATUS     ROLES           AGE
# zprojects-u18-aio  NotReady   control-plane   1m
# (NotReady is expected — CNI not installed yet)
```

### Step 3.4 — Install Flannel CNI (Network Plugin)

```bash
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

# Wait 30-60 seconds, then check
kubectl get nodes
# NAME               STATUS   ROLES           AGE
# zprojects-u18-aio  Ready    control-plane   2m
```

### Step 3.5 — Get the Join Command (if you didn't save it)

```bash
kubeadm token create --print-join-command
# Output:
# kubeadm join 10.71.114.X:6443 --token abc123... --discovery-token-ca-cert-hash sha256:...
```

---

## 6. Phase 4 — Join Worker Node

> Run on the **worker machine only**.

### Step 4.1 — Join the Cluster

```bash
# Paste the join command from Step 3.5 with sudo
sudo kubeadm join 10.71.114.X:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --cri-socket=unix:///run/containerd/containerd.sock
```

### Step 4.2 — Verify from Master

```bash
# Back on zprojects-u18-aio (master)
kubectl get nodes

# Expected:
# NAME               STATUS   ROLES           AGE
# zprojects-u18-aio  Ready    control-plane   5m
# zp-u16-aio-11      Ready    <none>          1m
```

### Step 4.3 — Label the Worker (optional but useful)

```bash
kubectl label node zp-u16-aio-11 node-role.kubernetes.io/worker=worker

# Verify
kubectl get nodes
# NAME               STATUS   ROLES           
# zprojects-u18-aio  Ready    control-plane   
# zp-u16-aio-11      Ready    worker
```

---

## 7. Phase 5 — Deploy Selenium Grid

> Run on **master** (`zprojects-u18-aio`).

### selenium-k8s.yaml (Full YAML)

Save this file at `~/selenium-k8s.yaml` on the master:

```yaml
# ─── NAMESPACE ───────────────────────────────────────────────────────
apiVersion: v1
kind: Namespace
metadata:
  name: selenium

# ─── SELENIUM HUB ────────────────────────────────────────────────────
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: selenium-hub
  namespace: selenium
  labels:
    app: selenium-hub
spec:
  replicas: 1
  selector:
    matchLabels:
      app: selenium-hub
  template:
    metadata:
      labels:
        app: selenium-hub
    spec:
      containers:
      - name: selenium-hub
        image: selenium/hub:4.33.0
        ports:
        - containerPort: 4444
        - containerPort: 4442
        - containerPort: 4443
        env:
        - name: SE_SESSION_REQUEST_TIMEOUT
          value: "300"
        - name: SE_SESSION_RETRY_INTERVAL
          value: "5"
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: selenium-hub
  namespace: selenium
spec:
  type: NodePort
  selector:
    app: selenium-hub
  ports:
  - name: web
    port: 4444
    targetPort: 4444
    nodePort: 30444
  - name: publish
    port: 4442
    targetPort: 4442
  - name: subscribe
    port: 4443
    targetPort: 4443

# ─── CHROME NODES ────────────────────────────────────────────────────
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: selenium-node-chrome
  namespace: selenium
  labels:
    app: selenium-node-chrome
spec:
  replicas: 5
  selector:
    matchLabels:
      app: selenium-node-chrome
  template:
    metadata:
      labels:
        app: selenium-node-chrome
    spec:
      containers:
      - name: chrome
        image: selenium/node-chrome:4.33.0
        env:
        - name: SE_EVENT_BUS_HOST
          value: "selenium-hub"
        - name: SE_EVENT_BUS_PUBLISH_PORT
          value: "4442"
        - name: SE_EVENT_BUS_SUBSCRIBE_PORT
          value: "4443"
        - name: SE_NODE_MAX_SESSIONS
          value: "2"
        - name: SE_NODE_SESSION_TIMEOUT
          value: "3600"
        - name: SE_NODE_GRID_URL
          value: "http://zprojects-u18-aio:30444"
        - name: SE_VNC_NO_PASSWORD
          value: "1"
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "4Gi"
            cpu: "1"
        volumeMounts:
        - mountPath: /dev/shm
          name: dshm
      volumes:
      - name: dshm
        emptyDir:
          medium: Memory
          sizeLimit: 1Gi

# ─── FIREFOX NODES ───────────────────────────────────────────────────
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: selenium-node-firefox
  namespace: selenium
  labels:
    app: selenium-node-firefox
spec:
  replicas: 5
  selector:
    matchLabels:
      app: selenium-node-firefox
  template:
    metadata:
      labels:
        app: selenium-node-firefox
    spec:
      containers:
      - name: firefox
        image: selenium/node-firefox:4.33.0
        env:
        - name: SE_EVENT_BUS_HOST
          value: "selenium-hub"
        - name: SE_EVENT_BUS_PUBLISH_PORT
          value: "4442"
        - name: SE_EVENT_BUS_SUBSCRIBE_PORT
          value: "4443"
        - name: SE_NODE_MAX_SESSIONS
          value: "2"
        - name: SE_NODE_SESSION_TIMEOUT
          value: "3600"
        - name: SE_NODE_GRID_URL
          value: "http://zprojects-u18-aio:30444"
        - name: SE_VNC_NO_PASSWORD
          value: "1"
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "4Gi"
            cpu: "1"
        volumeMounts:
        - mountPath: /dev/shm
          name: dshm
      volumes:
      - name: dshm
        emptyDir:
          medium: Memory
          sizeLimit: 1Gi

# ─── EDGE NODES ──────────────────────────────────────────────────────
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: selenium-node-edge
  namespace: selenium
  labels:
    app: selenium-node-edge
spec:
  replicas: 2
  selector:
    matchLabels:
      app: selenium-node-edge
  template:
    metadata:
      labels:
        app: selenium-node-edge
    spec:
      containers:
      - name: edge
        image: selenium/node-edge:4.33.0
        env:
        - name: SE_EVENT_BUS_HOST
          value: "selenium-hub"
        - name: SE_EVENT_BUS_PUBLISH_PORT
          value: "4442"
        - name: SE_EVENT_BUS_SUBSCRIBE_PORT
          value: "4443"
        - name: SE_NODE_MAX_SESSIONS
          value: "2"
        - name: SE_NODE_SESSION_TIMEOUT
          value: "3600"
        - name: SE_NODE_GRID_URL
          value: "http://zprojects-u18-aio:30444"
        - name: SE_VNC_NO_PASSWORD
          value: "1"
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1"
        volumeMounts:
        - mountPath: /dev/shm
          name: dshm
      volumes:
      - name: dshm
        emptyDir:
          medium: Memory
          sizeLimit: 1Gi
```

### Deploy

```bash
kubectl apply -f ~/selenium-k8s.yaml

# Watch pods come up — they will run on WORKER node
kubectl get pods -n selenium -o wide -w

# Expected (all pods on worker node IP):
# NAME                                    READY   STATUS    NODE
# selenium-hub-xxxxx                      1/1     Running   zp-u16-aio-11
# selenium-node-chrome-xxxxx-1            1/1     Running   zp-u16-aio-11
# ... (1 hub + 5 chrome + 5 firefox + 2 edge = 13 pods)
```

---

## 8. Phase 6 — NodePort + socat as systemd Service

> Run on **master** (`zprojects-u18-aio`).

The hub is exposed as a **NodePort** service on port `30444` on the worker node. A `socat` systemd service on the master forwards local port `4444` → `zprojects-u18-aio:30444` so that any existing config using port 4444 continues to work.

### Step 6.1 — Verify NodePort is Applied

The YAML already defines the hub service as `NodePort` with `nodePort: 30444`. After applying:

```bash
kubectl get svc selenium-hub -n selenium
# Expected:
# NAME           TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)
# selenium-hub   NodePort   10.96.x.x     <none>        4444:30444/TCP,...
```

### Step 6.2 — Open Firewall on Master

```bash
sudo ufw allow 30444/tcp
sudo ufw allow 4444/tcp
sudo ufw reload

# Verify
sudo ufw status | grep -E "30444|4444"
```

### Step 6.3 — Install socat

```bash
sudo apt-get install -y socat
```

### Step 6.4 — Create socat systemd Service

```bash
sudo tee /etc/systemd/system/selenium-4444.service > /dev/null <<'EOF'
[Unit]
Description=Socat forward port 4444 to Selenium NodePort 30444
After=network.target

[Service]
ExecStart=/usr/bin/socat TCP-LISTEN:4444,fork,reuseaddr TCP:zprojects-u18-aio:30444
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now selenium-4444

# Verify
sudo systemctl status selenium-4444
```

### Step 6.5 — Verify Hub is Reachable

```bash
# Via NodePort (direct — used by autovm32)
curl -s http://zprojects-u18-aio:30444/status | python3 -m json.tool | grep '"ready"'
# Expected: "ready": true

# Via socat (port 4444 — for any config still using port 4444)
curl -s http://zprojects-u18-aio:4444/status | python3 -m json.tool | grep '"ready"'
# Expected: "ready": true
```

---

## 9. Phase 7 — Update Autospark Properties

> Properties are on **autovm32** (`zpbt-autovm32`) at `/home/test/Spark/Zoho/AutoSpark/webapps/spark/Autospark/`.

The app reads two hub URL properties: `hub_url` (local/fallback) and `hub_url_vm32` (used when running from autovm32). **Both must point to the K8s hub.**

### Files that needed updating

| File | Property | Value |
|---|---|---|
| `WEB_DAILY.properties` | `hub_url` | `http://zprojects-u18-aio:30444` |
| `WEB_DAILY.properties` | `hub_url_vm32` | `http://zprojects-u18-aio:30444` |
| `CMPTHREE.properties` | `hub_url` | `http://zprojects-u18-aio:4444` |
| `CMPTHREE.properties` | `hub_url_vm32` | `http://zprojects-u18-aio:4444` |
| `DAILY.properties` | `hub_url` | `http://zprojects-u18-aio:4444` |
| `DAILY.properties` | `hub_url_vm32` | `http://zprojects-u18-aio:4444` |
| `Autospark.properties` | `hub_url` | `http://10.71.114.142:4444` (old grid — fallback only) |
| `Autospark.properties` | `hub_url_vm32` | `http://zprojects-u18-aio:30444` |

> **Key lesson:** If `hub_url_vm32` is missing from a properties file, the app falls back to `hub_url` from `Autospark.properties` (old grid IP `10.71.114.142:4444`), sending all requests to the wrong grid. Always add `hub_url_vm32` to every properties file used for web automation runs.

### Example — CMPTHREE.properties

```properties
hub_url=http://zprojects-u18-aio:4444
hub_url_vm32=http://zprojects-u18-aio:4444
```

> No Java code changes needed. Changes take effect on the next test run (properties are read at runtime).

---

## 10. Phase 8 — Verification Checklist

Run in order on **master**:

```bash
# 1. Both nodes Ready?
kubectl get nodes
# Expected: master=Ready (control-plane), worker=Ready

# 2. All Selenium pods Running on worker?
kubectl get pods -n selenium -o wide
# Expected: 13 pods Running, all on worker node

# 3. Hub registered all nodes?
curl -s http://zprojects-u18-aio:30444/status | python3 -m json.tool | grep -E '"ready"|"nodeCount"'
# Expected: "ready": true

# 4. Count registered browser nodes
curl -s http://zprojects-u18-aio:30444/status | python3 -c "
import json, sys
data = json.load(sys.stdin)
nodes = data['value']['nodes']
print('Total nodes registered:', len(nodes))
for n in nodes:
    caps = n['slots'][0]['stereotype'] if n['slots'] else {}
    print(' -', caps.get('browserName','?'), '|', n['uri'])
"

# 5. Open Grid UI
# http://zprojects-u18-aio:30444
# All 3 browser types visible

# 6. Trigger a web run from ZPTool UI
# Watch pods get active sessions
kubectl get pods -n selenium -w

# 7. After run — check ZPTool report
# Result should match expected pass/fail
```

---

## 11. Live VNC Browser Viewing

Each browser pod has a built-in noVNC viewer on port `7900`.

### Step 1 — Find the pod with active session

```bash
curl -s http://zprojects-u18-aio:30444/status | python3 -c "
import json, sys
data = json.load(sys.stdin)
for node in data['value']['nodes']:
    for slot in node.get('slots', []):
        if slot.get('session'):
            print('Container:', slot['session']['capabilities'].get('se:containerName'))
            print('Browser  :', slot['session']['capabilities'].get('browserName'))
            print('SessionId:', slot['session']['sessionId'])
"
```

### Step 2 — Port-forward the pod's VNC

```bash
kubectl port-forward pod/<pod-name> 7900:7900 -n selenium
# Example:
kubectl port-forward pod/selenium-node-chrome-7b7f6d8ddd-hvglx 7900:7900 -n selenium
```

### Step 3 — Open in browser

**If using the machine directly:**
```
http://127.0.0.1:7900
```

**If SSH'd in from your Mac:**
```bash
# On your Mac — open new terminal
ssh -L 7900:127.0.0.1:7900 projects@zprojects-u18-aio -N
```
Then open `http://127.0.0.1:7900` in Mac browser.

> No password needed — `SE_VNC_NO_PASSWORD=1` is set in the YAML.

### Watch multiple sessions simultaneously

```bash
# Terminal 1
kubectl port-forward pod/selenium-node-chrome-xxxx-1 7900:7900 -n selenium
# Terminal 2
kubectl port-forward pod/selenium-node-firefox-xxxx-1 7901:7900 -n selenium
# Terminal 3
kubectl port-forward pod/selenium-node-chrome-xxxx-2 7902:7900 -n selenium
```

Open `http://127.0.0.1:7900`, `7901`, `7902` in browser tabs.

---

## 12. Scaling Pods at Runtime

### Resource Reference

| Worker | RAM | Allocatable | Max safe pods (1Gi request) |
|---|---|---|---|
| `zp-u16-aio-11` | 15.4Gi | ~15Gi | 12 |
| `zpbt-autovm33` | 30.5Gi | ~30Gi | 25 |
| `zpbt-autovm34` | 30.5Gi | ~30Gi | 25 |
| **Total** | **76Gi** | **~75Gi** | **60 pods → 120 slots** |

> At 9 concurrent threads × ~1.5Gi actual browser usage = ~13.5Gi active memory — well within safe limits even at 120 slots.

```bash
# Current scale (116 slots across 3 workers)
kubectl scale deployment selenium-node-chrome -n selenium --replicas=25
kubectl scale deployment selenium-node-firefox -n selenium --replicas=25
kubectl scale deployment selenium-node-edge -n selenium --replicas=8

# Maximum safe scale (120 slots — needs CPU headroom)
kubectl scale deployment selenium-node-chrome -n selenium --replicas=25
kubectl scale deployment selenium-node-firefox -n selenium --replicas=25
kubectl scale deployment selenium-node-edge -n selenium --replicas=10

# Scale down after run
kubectl scale deployment selenium-node-chrome -n selenium --replicas=5
kubectl scale deployment selenium-node-firefox -n selenium --replicas=5
kubectl scale deployment selenium-node-edge -n selenium --replicas=2

# Check current counts
kubectl get pods -n selenium | grep -v hub | wc -l
```

> **Note:** Scaling UP is safe while tests are running — K8s only creates new pods and never touches existing running ones. Scaling DOWN while tests run risks killing pods with active sessions.

---

## 13. Useful kubectl Commands

```bash
# All pods with node assignment
kubectl get pods -n selenium -o wide

# Watch pods in real time
kubectl get pods -n selenium -w

# Hub logs
kubectl logs deployment/selenium-hub -n selenium

# Chrome node logs
kubectl logs deployment/selenium-node-chrome -n selenium

# Pod resource usage (requires metrics-server)
kubectl top pods -n selenium

# Describe a pod (for pending/crash issues)
kubectl describe pod <pod-name> -n selenium

# Rolling restart all browser pods
kubectl rollout restart deployment/selenium-node-chrome -n selenium
kubectl rollout restart deployment/selenium-node-firefox -n selenium
kubectl rollout restart deployment/selenium-node-edge -n selenium

# Check cluster nodes status
kubectl get nodes -o wide

# Delete and re-apply everything
kubectl delete -f ~/selenium-k8s.yaml
kubectl apply -f ~/selenium-k8s.yaml

# Check port-forward service
sudo systemctl status selenium-portforward
sudo systemctl restart selenium-portforward

# Get worker node details
kubectl describe node zp-u16-aio-11
```

---

## 14. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `kubectl get nodes` — worker NotReady | CNI not installed or kubelet not running on worker | `sudo systemctl status kubelet` on worker, check logs |
| Pods stuck in `Pending` | Worker has insufficient CPU/RAM | `kubectl describe pod <name> -n selenium` → check Events |
| Pods stuck in `Pending` — no node available | Master taint blocking scheduling | `kubectl taint nodes zprojects-u18-aio node-role.kubernetes.io/control-plane:NoSchedule-` |
| `Connection refused` on port 4444 | Port-forward service not running | `sudo systemctl restart selenium-portforward` |
| Hub shows 0 nodes | Nodes can't reach hub DNS `selenium-hub` | Check CoreDNS pods: `kubectl get pods -n kube-system` |
| `kubeadm join` fails with cert error | Token expired (24h TTL) | On master: `kubeadm token create --print-join-command` |
| Worker can't reach master:6443 | Firewall blocking | `sudo ufw allow 6443/tcp` on master |
| CrashLoopBackOff on browser pods | `/dev/shm` mount issue | Verify `emptyDir` volume in YAML |
| `kubelet` not starting on master | Swap still enabled | `sudo swapoff -a` and verify `/etc/fstab` |
| `SystemdCgroup` error in kubelet logs | containerd not using SystemdCgroup | Redo Step 2.5 and restart containerd |
| Selenium tests failing after migration | `hub_url` still pointing to old VM | Update `Autospark.properties` and redeploy WAR |
| Hub receiving ZERO session requests | `hub_url_vm32` missing from properties file; app falls back to old grid IP `10.71.114.142:4444` | Add `hub_url_vm32=http://zprojects-u18-aio:30444` to every `.properties` file on autovm32 |
| `JdkWebSocketClient` / BiDi connection error after session created | Nodes advertise pod IP in `se:cdp`; autovm32 cannot reach pod IPs directly | Set `SE_NODE_GRID_URL=http://zprojects-u18-aio:30444` on all node deployments: `kubectl set env deployment/selenium-node-chrome deployment/selenium-node-firefox deployment/selenium-node-edge -n selenium SE_NODE_GRID_URL=http://zprojects-u18-aio:30444` |
| `NullPointerException` in hub `LocalDistributor.newSession` | Version mismatch between Selenium Java client and grid (e.g. client 4.33.0 vs grid 4.20.0) | Upgrade hub + all node images to match the Java client: `kubectl -n selenium set image deployment/selenium-hub selenium-hub=selenium/hub:4.33.0` and same for node deployments |
| Firefox/Chrome pods OOMKilled (Exit Code 137) during long runs | Pod memory limit too low for concurrent sessions (default 2Gi) | Increase limit: `kubectl set resources deployment/selenium-node-firefox deployment/selenium-node-chrome -n selenium --requests=memory=1Gi,cpu=500m --limits=memory=4Gi,cpu=1` |
| New worker pods stuck in `ContainerCreating` with Calico CNI error (`x509: certificate signed by unknown authority`) | New worker has leftover Calico CNI config from a previous cluster; current cluster uses Flannel | On new worker: `sudo rm -f /etc/cni/net.d/10-calico.conflist /etc/cni/net.d/calico-kubeconfig && sudo systemctl restart kubelet` then delete stuck pods from master |
| `kubeadm join` fails: `unable to fetch kubeadm-config: this version only supports >= 1.32.0` | New worker has kubeadm v1.33 installed but cluster runs v1.28 | Downgrade: `sudo apt-mark unhold kubelet kubeadm kubectl && sudo apt-get install -y --allow-downgrades kubelet=1.28.15-1.1 kubeadm=1.28.15-1.1 kubectl=1.28.15-1.1 && sudo apt-mark hold kubelet kubeadm kubectl` |
| kubelet fails with `No such file or directory` after minikube purge | kubelet systemd drop-in still points to `/var/lib/minikube/binaries/v1.28.0/kubelet` | `sudo sed -i 's\|/var/lib/minikube/binaries/v1.28.0/kubelet\|/usr/bin/kubelet\|g' /etc/systemd/system/kubelet.service.d/10-kubeadm.conf && sudo systemctl daemon-reload` |
| Browser nodes crash-loop with `UnknownHostException: selenium-hub` | Docker's leftover `DOCKER-ISOLATION-STAGE-2` chain has blanket DROP rule blocking cross-pod traffic | See fix below |
| Browser nodes fail ZeroMQ connect after DNS fix | ZeroMQ uses pod IP not ClusterIP — DNS resolves but ZMQ still fails via service | Set `SE_EVENT_BUS_HOST` to hub pod IP (see below) |

### Worker node kubelet not starting — check logs

```bash
# On worker machine
sudo systemctl status kubelet
sudo journalctl -u kubelet -n 50 --no-pager
```

### Fix: kubelet pointing to old minikube binary (master only)

```bash
# Check drop-in
cat /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

# Fix binary path
sudo sed -i 's|/var/lib/minikube/binaries/v1.28.0/kubelet|/usr/bin/kubelet|g' \
  /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

### Fix: Docker isolation chains blocking cross-pod traffic (master only)

Symptom: Browser nodes crash-loop with `UnknownHostException: selenium-hub` or ZeroMQ connect failures.

Cause: Old Docker iptables rules (`DOCKER-ISOLATION-STAGE-2`) have a blanket `DROP ALL` rule that blocks pod-to-pod traffic between worker and master.

```bash
# Check for the DROP rule
sudo iptables -L DOCKER-ISOLATION-STAGE-2 -n
# If you see: DROP  all  --  0.0.0.0/0  0.0.0.0/0 → fix it:

sudo iptables -F DOCKER-ISOLATION-STAGE-1
sudo iptables -F DOCKER-ISOLATION-STAGE-2
sudo iptables -A DOCKER-ISOLATION-STAGE-1 -j RETURN
sudo iptables -A DOCKER-ISOLATION-STAGE-2 -j RETURN

# Also ensure FORWARD policy is ACCEPT
sudo iptables -P FORWARD ACCEPT
sudo iptables -I FORWARD 1 -s 10.244.0.0/16 -j ACCEPT
sudo iptables -I FORWARD 2 -d 10.244.0.0/16 -j ACCEPT

# Save permanently
sudo apt-get install -y iptables-persistent
sudo netfilter-persistent save
```

### Fix: ZeroMQ event bus failing even after DNS works

Symptom: `Connecting to tcp://selenium-hub:4442 and tcp://selenium-hub:4443 failed` despite DNS resolving.

Cause: ZeroMQ uses the hub's advertised pod IP directly (`tcp://10.244.1.3:4442`), not the ClusterIP service. The selenium node must connect directly to that pod IP.

```bash
# Get hub pod IP
kubectl get pod -n selenium -l app=selenium-hub -o jsonpath='{.items[0].status.podIP}'

# Set all browser nodes to use hub pod IP directly
kubectl set env deployment/selenium-node-chrome \
  deployment/selenium-node-firefox \
  deployment/selenium-node-edge \
  -n selenium SE_EVENT_BUS_HOST=<HUB_POD_IP>

# Verify registration
curl -s http://zprojects-u18-aio:30444/status | python3 -m json.tool | grep -E '"ready"|"uri"'
```

> **Note:** If the hub pod restarts, its IP changes. Re-run `kubectl set env` with the new IP. For a permanent fix, deploy the hub with a headless service or use `hostNetwork: true`.

### Master node ports that must be open

| Port | Used by |
|---|---|
| `6443` | Kubernetes API (worker joins here) |
| `2379-2380` | etcd |
| `10250` | kubelet API |
| `10251` | kube-scheduler |
| `10252` | kube-controller-manager |

```bash
# Open required ports on master
sudo ufw allow 6443/tcp
sudo ufw allow 10250/tcp
sudo ufw allow 2379:2380/tcp
```

### Worker node ports that must be open

```bash
# Open required ports on worker
sudo ufw allow 10250/tcp
sudo ufw allow 30000:32767/tcp   # NodePort range (optional)
```

---

## 15. Comparison: minikube vs kubeadm

| | minikube (`--driver=none`) | kubeadm multi-node |
|---|---|---|
| Machines needed | 1 | 2 |
| Master + Worker | Same machine | Separate machines |
| Resource sharing | ZPTool app + pods compete | Pods on dedicated worker |
| After reboot | Must run `minikube start` manually | kubelet auto-starts (systemd) |
| kubectl config | `/home/projects/.kube/config` | Same path — no change |
| Namespace | `selenium` | Same |
| YAML file | `selenium-k8s.yaml` | Same file — no changes |
| hub_url in properties | `http://127.0.0.1:4444` | `http://zprojects-u18-aio:30444` (NodePort) |
| Port-forward command | Same | socat NodePort (see Phase 6) |
| Java code changes | None | None |
| Add more nodes | Not possible | `kubeadm join` on new machine |
| Fault tolerance | Single point of failure | Worker crash doesn't stop master |
| Production readiness | Dev/testing | Production-grade |

---

## Key File Locations

| File | Machine | Path |
|---|---|---|
| Selenium K8s manifest | Master | `~/selenium-k8s.yaml` |
| kubeconfig | Master | `/home/projects/.kube/config` |
| K8s admin config | Master | `/etc/kubernetes/admin.conf` |
| systemd socat forward | Master | `/etc/systemd/system/selenium-4444.service` |
| Autospark properties | autovm32 | `/home/test/Spark/Zoho/AutoSpark/webapps/spark/Autospark/` |
| containerd config | All nodes | `/etc/containerd/config.toml` |
| kubelet drop-in | Master | `/etc/systemd/system/kubelet.service.d/10-kubeadm.conf` |
| kubelet service | All nodes | `systemctl status kubelet` |
| CNI plugins | All nodes | `/opt/cni/bin/` |
| Flannel CNI config | All nodes | `/etc/cni/net.d/10-flannel.conflist` |
| iptables saved rules | Master | `/etc/iptables/rules.v4` |

---

## 16. Adding More Worker Nodes

> Use this when joining new VMs (e.g. `zpbt-autovm33`, `zpbt-autovm34`) to the existing cluster.

### Step 1 — Generate join command (on master)

```bash
kubeadm token create --print-join-command
```

### Step 2 — Prepare the new VM

```bash
# Disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Kernel modules + sysctl
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay && sudo modprobe br_netfilter
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system

# Install/fix containerd
sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd && sudo systemctl enable containerd
```

### Step 3 — Install kubeadm/kubelet/kubectl matching cluster version (v1.28)

> **Important:** If the VM has a newer version (e.g. v1.33), downgrade with `--allow-downgrades`.

```bash
sudo apt-get install -y apt-transport-https ca-certificates curl
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update

# If already has newer version:
sudo apt-mark unhold kubelet kubeadm kubectl
sudo apt-get install -y --allow-downgrades kubelet=1.28.15-1.1 kubeadm=1.28.15-1.1 kubectl=1.28.15-1.1
sudo apt-mark hold kubelet kubeadm kubectl

# Verify
kubeadm version
```

### Step 4 — Remove leftover CNI config (if VM had previous K8s)

```bash
# Only needed if VM was previously part of another cluster
sudo kubeadm reset -f
sudo rm -rf /etc/kubernetes /var/lib/kubelet ~/.kube

# Remove old Calico CNI if present (cluster uses Flannel)
sudo rm -f /etc/cni/net.d/10-calico.conflist /etc/cni/net.d/calico-kubeconfig
sudo systemctl restart containerd
```

### Step 5 — Join the cluster

```bash
sudo kubeadm join <MASTER_IP>:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --cri-socket=unix:///run/containerd/containerd.sock
```

### Step 6 — Label and verify (on master)

```bash
kubectl get nodes
kubectl label node <new-node-name> node-role.kubernetes.io/worker=worker
```

### Step 7 — Scale up Selenium pods

```bash
kubectl scale deployment selenium-node-chrome -n selenium --replicas=15
kubectl scale deployment selenium-node-firefox -n selenium --replicas=15
kubectl scale deployment selenium-node-edge -n selenium --replicas=6

# Watch new pods distribute across workers
kubectl get pods -n selenium -o wide -w
```

> If new pods get stuck in `ContainerCreating` on the new worker, check: `kubectl describe pod <pod-name> -n selenium | tail -15`. If Calico CNI error → see troubleshooting table.

---

*Generated for zprojects-u18-aio (master) + zp-u16-aio-11, zpbt-autovm33, zpbt-autovm34 (workers) — ZPTool Web Automation kubeadm Multi-Node Cluster*
