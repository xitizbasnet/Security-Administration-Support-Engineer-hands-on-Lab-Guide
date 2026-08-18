# LAB 06 — Certificate Deployment in Kubernetes & Docker

> **Manage TLS in containerized environments using Secrets, cert-manager, and Let's Encrypt**

## Objective

Deploy certificates in Kubernetes via Secrets and Ingress, automate with cert-manager, and configure Docker with TLS for secure registry/daemon communication.

---

## Task 1: Create TLS Secret in Kubernetes

### Step 1: Create TLS Secret from Existing Certificate and Key

```bash
kubectl create secret tls mysite-tls \
--cert=server.crt \
--key=server.key \
--namespace=production
```

### Step 2: Verify the Secret

```bash
kubectl get secret mysite-tls -n production
kubectl describe secret mysite-tls -n production
```

### Step 3: Decode and Inspect the Certificate from the Secret

```bash
kubectl get secret mysite-tls -n production \
-o jsonpath='{.data.tls\.crt}' | base64 -d \
| openssl x509 -noout -text
```

> **❗ Best Practice**
>
> TLS Secrets in Kubernetes store base64-encoded data. Never commit Secret YAML with actual cert data to Git. Use Sealed Secrets or External Secrets Operator.

---

## Task 2: NGINX Ingress with TLS

### Step 1: Create Ingress Resource with TLS

Create the following file:

```text
ingress-tls.yaml
```

```yaml
# ingress-tls.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
name: mysite-ingress
namespace: production
annotations:
nginx.ingress.kubernetes.io/ssl-redirect: 'true'
spec:
tls:
- hosts:
- mysite.example.com
secretName: mysite-tls
rules:
- host: mysite.example.com
http:
paths:
- path: /
pathType: Prefix
backend:
service:
name: mysite-svc
port:
number: 80
```

> **📄 Source Reference:** Security Administration & Support Engineer Lab Guide | Page 14 of 27

### Step 2: Apply and Verify

```bash
kubectl apply -f ingress-tls.yaml
kubectl describe ingress mysite-ingress -n production
kubectl get ingress -n production
```

---

## Task 3: Install cert-manager

### Step 1: Install cert-manager via Helm

Add the Jetstack Helm repository:

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
```

Install cert-manager:

```bash
helm install cert-manager jetstack/cert-manager \
--namespace cert-manager --create-namespace \
--set installCRDs=true
```

### Step 2: Verify Pods Are Running

```bash
kubectl get pods -n cert-manager
```

---

## Task 4: Let's Encrypt via cert-manager

### Step 1: Create ClusterIssuer for Let's Encrypt

Create the following configuration:

```yaml
# letsencrypt-issuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
name: letsencrypt-prod
spec:
acme:
server: https://acme-v02.api.letsencrypt.org/directory
email: admin@example.com
privateKeySecretRef:
name: letsencrypt-prod-key
solvers:
- http01:
ingress:
class: nginx
```

### Step 2: Create Certificate Resource

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
name: mysite-cert
namespace: production
spec:
secretName: mysite-tls
issuerRef:
name: letsencrypt-prod
kind: ClusterIssuer
dnsNames:
- mysite.example.com
```

### Step 3: Check Certificate Status

```bash
kubectl describe certificate mysite-cert -n production
kubectl get certificaterequest -n production
```

> **❗ Best Practice**
>
> cert-manager automatically renews Let's Encrypt certificates 30 days before expiry. Monitor with:
>
> ```bash
> kubectl get certificates -A
> ```

> **📄 Source Reference:** Security Administration & Support Engineer Lab Guide | Page 15 of 27

---

## Task 5: Docker TLS for Secure Registry

### Step 1: Generate Certificates for Docker Registry

Create the certificate directory and move into it:

```bash
mkdir -p ~/docker-certs && cd ~/docker-certs
```

Generate the registry certificate:

```bash
openssl req -newkey rsa:2048 -nodes -keyout registry.key \
-x509 -days 365 -out registry.crt -subj "/CN=registry.example.com"
```

### Step 2: Run Docker Registry with TLS

```bash
docker run -d -p 5000:5000 --name registry \
-v ~/docker-certs:/certs \
-e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/registry.crt \
-e REGISTRY_HTTP_TLS_KEY=/certs/registry.key \
registry:2
```

### Step 3: Trust the Registry Certificate on Docker Daemon

Create the Docker certificate directory:

```bash
sudo mkdir -p /etc/docker/certs.d/registry.example.com:5000
```

Copy the registry certificate:

```bash
sudo cp ~/docker-certs/registry.crt \
/etc/docker/certs.d/registry.example.com:5000/ca.crt
```

Restart Docker:

```bash
sudo systemctl restart docker
```
