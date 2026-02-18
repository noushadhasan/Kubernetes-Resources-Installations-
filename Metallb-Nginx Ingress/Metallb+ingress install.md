# Kubernetes Load Balancing with MetalLB + NGINX Ingress on k3s

This guide explains how to install **MetalLB** (for bare-metal LoadBalancer support) and **NGINX Ingress Controller** (for routing HTTP/HTTPS traffic) in a k3s cluster.

---

## Prerequisites
A running [k3s](https://k3s.io/) cluster installed **without Traefik**:
```bash
  curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable traefik" sh -
```

# Install MetalLB
```bash
kubectl create namespace metallb-system
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.5/config/manifests/metallb-native.yaml
```
# Wait for pods to be ready:
```bash
kubectl get pods -n metallb-system
```
* kubectl configured and pointing to your cluster.
* A range of unused IPs in your LAN/VM network (e.g., 192.168.1.100-192.168.1.120)
# Configure IP Address Pool
Create a config file metallb-config.yaml:
```bash
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-address-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.100-192.168.1.120   # <-- change to your network range
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default-l2
  namespace: metallb-system
```
Apply it:
```bash
kubectl apply -f metallb-config.yaml
```
# Install Helm
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```
# Install NGINX Ingress Controller
Add the Helm repo and install ingress-nginx:
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

kubectl create namespace ingress-nginx

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --set controller.publishService.enabled=true
```
chech the service:
```bash
kubectl get svc -n ingress-nginx
```
You should see an EXTERNAL-IP assigned by MetalLB.
