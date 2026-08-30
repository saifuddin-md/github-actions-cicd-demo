
**Install HELM**
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
helm
```
---
## Install Metrics Server

**Add Helm repo:**

```bash
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm repo update
```

**Install:**

```bash
helm upgrade --install metrics-server metrics-server/metrics-server
```

(*OR*).

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

**Check:**
```bash
kubectl top pods
kubectl top nodes
```
**Expected:**

ingress-nginx-controller-xxxxx   Running 
