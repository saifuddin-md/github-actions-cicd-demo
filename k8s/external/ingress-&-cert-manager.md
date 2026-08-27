**Install HELM**
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
helm
```
---
## Step 1: Install NGINX Ingress Controller

**Add Helm repo:**

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

**Install:**

```bash
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace
```

(*This ensures AWS creates an NLB.*).

```xml
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-type"="external" \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-nlb-target-type"="ip"
```

**Check:**
```bash
kubectl get pods -n ingress-nginx
```
**Expected:**

ingress-nginx-controller-xxxxx   Running 

---

## Step 2: Verify Controller Service

```bash
kubectl get svc -n ingress-nginx
```

**Expected:**

| NAME                       | TYPE           | EXTERNAL-IP
|----------------------------|----------------|----------------
| ingress-nginx-controller   | LoadBalancer   | a1b2c3d4...


```bash
kubectl get deploy -n ingress-nginx
```
```bash
kubectl describe pod <pod-name> -n ingress-nginx
```

*AWS automatically creates a Load Balancer.*

*Save the external hostname.*

**Example:**

```bash
a1b2c3d4.ap-south-1.elb.amazonaws.com
```
---

## Step 3: Verify Your Services


```bash
kubectl get svc -n rr-app
```

**You should see something like:**

- ct-frontend-service   ClusterIP   80/TCP
- ct-backend-service    ClusterIP   5000/TCP

*If they exist, continue.*

---

## Step 4: Create Ingress

Create ingress.yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rr-app-ingress
  namespace: rr-app

spec:
  ingressClassName: nginx
  rules:
  - host: saiftech.com
    http:
      paths:

      # Frontend
      - path: /
        pathType: Prefix
        backend:
          service:
            name: ct-frontend-service
            port:
              number: 80

      # Backend API
      - path: /api/v1
        pathType: Prefix
        backend:
          service:
            name: ct-backend-service
            port:
              number: 5001
```

**Apply:**

```bash
kubectl apply -f ingress.yaml
```
---

## Step 5: Verify Ingress

```bash
kubectl get ingress -n rr-app
```

**Expected:*

| NAME             |CLASS   | HOSTS     |
|------------------|--------|-----------|
| rr-app-ingress   \ nginx  | app.example.com |

```bash
kubectl describe ingress rr-app-ingress -n rr-app

kubectl logs -n ingress-nginx <controller-pod>

kubectl get ingressclass
```
---

## Step 6: Configure DNS

*In your DNS provider (for example, Route53), create:*

app.example.com

**Point it to:**

a1b2c3d4.ap-south-1.elb.amazonaws.com

(the hostname from Step 2)

---

## Step 7: Test HTTP

**Frontend:**

http://app.example.com

**Backend:**

http://app.example.com/api

At this stage everything works over HTTP.

---

## Step 8: Add HTTPS with cert-manager

After HTTP is working:

- Install cert-manager.
- Create a ClusterIssuer for Let's Encrypt.
- Add TLS configuration to the Ingress.
- cert-manager automatically obtains and renews certificates.

### Step 8.1: Install cert-manager

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
```

**Install cert-manager:**

```bash
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true
```

**Delete**

```bash
helm uninstall cert-manager -n cert-manager
kubectl delete namespace cert-manager
kubectl get crd | grep cert-manager | awk '{print $1}' | xargs kubectl delete crd
```

**Verify:**

```bash
kubectl get pods -n cert-manager
```

**You should see:**

- cert-manager
- cert-manager-cainjector
- cert-manager-webhook

all in Running state.


### Step 8.2: Create ClusterIssuer

Create clusterissuer.yaml

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: your-email@example.com
    server: https://acme-v02.api.letsencrypt.org/directory

    privateKeySecretRef:
      name: letsencrypt-prod

    solvers:
    - http01:
        ingress:
          class: nginx
```

**Apply:**

```bash
kubectl apply -f clusterissuer.yaml
```

**Verify:**

```bash
kubectl get clusterissuer
```
**Expected:**

| NAME               | READY |
|--------------------|-------|
| letsencrypt-prod   | True  |


### Step 8.3: Update Ingress

**Modify your Ingress:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rr-app-ingress
  namespace: rr-app
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod    #

spec:
  ingressClassName: nginx

  tls:                                                 #
  - hosts:                                             #
    - saiftech.com                                     #
    secretName: rr-app-tls                             #

  rules:
  - host: saiftech.com
    http:
      paths:

      # Frontend
      - path: /
        pathType: Prefix
        backend:
          service:
            name: ct-frontend-service
            port:
              number: 80

      # Backend API
      - path: /api/v1
        pathType: Prefix
        backend:
          service:
            name: ct-backend-service
            port:
              number: 5001
```

**Apply:**

```bash
kubectl apply -f ingress.yaml
```

### Step 8.4: What cert-manager does

**When it sees:**

cert-manager.io/cluster-issuer: letsencrypt-prod

it automatically:

1. Requests a certificate from Let's Encrypt.
2. Creates a temporary challenge route.
3. Proves ownership of app.example.com.
4. Receives the certificate.
5. Stores it in a Secret:  **rr-app-tls**
6. Configures NGINX to use it.
7. Renews it automatically before expiry.

### Step 8.5: Verify

**Check certificate:**

```bash
kubectl get certificate -n rr-app
```

**Expected:**

| NAME        | READY   |
|-------------|---------|
| rr-app-tls    | True |

**Check secret:**

```bash
kubectl get secret -n rr-app
```
**Expected:**

rr-app-tls

**Check**

```bash
kubectl get secret rr-app-tls -n rr-app -o yaml
```
**Inside the Secret are:**

- tls.crt   <-- SSL certificate
- tls.key   <-- Private key

### Step 8.6: Test HTTPS

https://app.example.com
