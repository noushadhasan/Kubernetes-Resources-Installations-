# Apply the Metrics Server Manifest on your Master Node
```
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```
* This command downloads and applies the latest Metrics Server manifest, which includes the necessary components (Deployment, Service, etc.) in the kube-system namespace.

# Verify the Installation
```
kubectl get pods -n kube-system | grep metrics-server
```
#Insecure TLS: Some clusters require additional flags to handle self-signed certificates or other TLS issues. Edit the Metrics Server deployment to add the --kubelet-insecure-tls flag:
```
kubectl edit deployment metrics-server -n kube-system
```
Add the following under spec.template.spec.containers[0].args:
```
- --kubelet-insecure-tls
```
Save and exit, then wait for the pod to restart.
